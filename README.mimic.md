# Mimic on GL-MT6000

Notes on integrating [hack3ric/mimic](https://github.com/hack3ric/mimic) (eBPF
TCP-in-UDP obfuscator) into this OpenWrt fork for the GL-MT6000 (MT7986A,
filogic).

## Package source

`package/network/services/mimic/Makefile` is a near-copy of upstream's
`net/mimic/Makefile` from the [`openwrt` branch](https://github.com/hack3ric/mimic/tree/openwrt),
with three deltas:

- `commit := 493faf5dfd440bc44bc0d2a88baaca4d7ef0b709` (HEAD of `master` as of
  2025-11-20, post-0.7.0). Upstream pinned `ad7a4b95...` from August 2025.
- `PKG_HASH := 0966e7da8191b6234d2d6f8120f0f34974a9bfb1dda45db96747b13aea8ab954`
  matching the new tarball.
- `DEPENDS` adds `$(BPF_DEPENDS)` so enabling the package auto-selects
  `NEED_BPF_TOOLCHAIN` (mirrors what `tcp-in-udp`, `bridger`, `unetd` do).

`CHECKSUM_HACK=kprobe` is kept (per upstream's
[`openwrt.md`](https://github.com/hack3ric/mimic/blob/master/docs/openwrt.md))
because OpenWrt kernels typically lack BTF for the `kfunc` hack.

## Build configuration

These are not committed (they live in your `.config` / `diffconfig.mt6000`,
which are user-managed and gitignored). Add them to `diffconfig.mt6000` to
persist across `cp diffconfig.mt6000 .config; make defconfig`.

### Required for the package

```
CONFIG_PACKAGE_mimic=y
CONFIG_PACKAGE_kmod-mimic=y
```

### Required for kprobe checksum hack to actually work at runtime

```
CONFIG_KERNEL_KPROBES=y
```

This auto-selects `KERNEL_FTRACE`, `KERNEL_PERF_EVENTS`, `KERNEL_KPROBE_EVENTS`
and translates to `CONFIG_KPROBES=y` + `CONFIG_KRETPROBES=y` in the kernel
build. Cost: ~+2 MB on the kernel image, ~+8 KB on `mimic.ko`.

### Required for the BPF toolchain (build-host side)

```
CONFIG_DEVEL=y
CONFIG_BPF_TOOLCHAIN_HOST=y
CONFIG_USE_LLVM_HOST=y
CONFIG_BPF_TOOLCHAIN_HOST_PATH="/usr/lib/llvm-18"
```

`CONFIG_DEVEL=y` only unhides advanced settings in `menuconfig`; it doesn't
change the firmware itself. Without it, `defconfig` silently picks
`BPF_TOOLCHAIN_BUILD_LLVM`, which builds LLVM 22.x from source (~30 min).

`BPF_TOOLCHAIN_HOST_PATH` must point at a directory whose `bin/` has
**unversioned** `clang`, `llvm-strip`, `llc`, `opt`, etc. On Ubuntu/Debian
the unversioned binaries live under `/usr/lib/llvm-<N>/bin/`, not directly
in `/usr/bin/` — `/usr/bin` only has the suffixed `clang-18` /
`llvm-strip-18` form, which `bpf.mk`'s `find-llvm-tool` does not find when
the unsuffixed `clang` symlink resolves first and sets `LLVM_VER=""`.

Mimic requires clang >= 15 (see
[`docs/building.md`](https://github.com/hack3ric/mimic/blob/master/docs/building.md)).
Tested with the host's `/usr/lib/llvm-18`.

## When do you need `kmod-mimic` loaded at runtime?

Per [`docs/checksum-hacks.md`](https://github.com/hack3ric/mimic/blob/master/docs/checksum-hacks.md#when-is-checksum-hack-not-necessary)
both conditions must match for the kernel-module checksum hack to be
unnecessary:

1. NIC driver does not use `csum_offset`, OR checksum offload is manually
   disabled. **MT6000's `mtk_eth_soc` driver does not use `csum_offset`** —
   condition 1 is automatic on this hardware.
2. Tunnel software uses a userspace UDP socket.

| Tunnel | Userspace UDP? | Need `kmod-mimic` + `KPROBES`? |
|---|---|---|
| `kmod-wireguard` (kernel WireGuard) | no — uses `udptunnel4/6` with `CSUM_PARTIAL` | yes |
| `wireguard-go` | yes | no |
| OpenVPN, Hysteria, etc. | yes | no |
| FOU / GENEVE / other in-kernel UDP tunnels | no | yes |

Practical note: kernel WireGuard is the typical choice on a router because
it's much faster, so most users will need the kernel module.

## Runtime usage on the device

### Load the kernel module

```sh
modprobe mimic
echo mimic > /etc/modules.d/mimic        # persist across reboots
dmesg | tail                             # confirm kprobe registration
```

### Reading kmod state: `lsmod` shows refcount 0, that's normal here

On this OpenWrt build mimic appears as `mimic 12288 0` — the third column
("Used by") is **0** even while `mimic.ko` is actively rewriting packets.
On a desktop/server distro you'll see `1` instead. Same module, different
hook mechanism:

| | Distro / Ubuntu | This OpenWrt build |
|---|---|---|
| Kernel BTF (`CONFIG_DEBUG_INFO_BTF`) | y | not set |
| `CHECKSUM_HACK` mode | `kfunc` (mimic default) | **`kprobe`** (per upstream `openwrt.md`) |
| `ip -d link show` on the mimic'd iface | shows `btf_id NNN` | no `btf_id` (we build with `STRIP_BTF_EXT=1`) |
| BPF → kmod link | BPF program references mimic's exported kfunc; kernel does `try_module_get(mimic)` on program load | BPF program calls vanilla `bpf_skb_change_type` / `bpf_skb_change_proto`; mimic kprobes those vmlinux symbols |
| `lsmod` "Used by" | 1 (BPF program holds ref) | 0 (`try_module_get` in `register_kretprobe` pins the *probed* module — vmlinux/NULL — not the handler-owning module) |
| `rmmod mimic` while userspace mimic runs | fails (refcount > 0) | succeeds; mimic's `module_exit` calls `unregister_kretprobe` cleanly |

To confirm the kprobes are actually live on OpenWrt:

```sh
mount -t debugfs none /sys/kernel/debug 2>/dev/null
grep -E 'bpf_skb_change' /sys/kernel/debug/kprobes/list
# expect lines like:
#   ffffffc0xxxxxxxx  r  bpf_skb_change_type+0x0   [FTRACE]  ...
#   ffffffc0xxxxxxxx  r  bpf_skb_change_proto+0x0  [FTRACE]  ...
```

(`r` = kretprobe.) Or watch for the runtime-fired log lines:

```
mimic: bpf_skb_change_type with magic flag called, skb->ip_summed = 3
mimic: bpf_skb_change_proto with magic flag called, skb->csum_offset changed from 0 to 0
```

These are the kretprobe handler's printk's; if you see them in `dmesg`,
the hook is doing its job. The "Used by 0" is purely a refcount-accounting
quirk of the kprobe path, not a sign of inactivity.

### Firewall: allow both TCP and UDP to the peer

Mimic's userspace daemon injects raw TCP packets (SYN, keepalive) via a
`SOCK_RAW + IPPROTO_TCP` socket. These hit the netfilter OUTPUT hook **as
TCP**, so OpenWrt's `fw4` will mark them `ct state invalid` and reject with
EPERM (visible as `failed to send: Operation not permitted` in mimic's
log). Per [`docs/getting-started.md#firewall`](https://github.com/hack3ric/mimic/blob/master/docs/getting-started.md#firewall),
mimic traffic must be allowed as **both TCP and UDP** for the same
`<peer_ip>:<port>`.

```
config rule
	option name 'Allow-mimic-tcp-out'
	option src '*'
	option direction 'out'
	option dest '*'
	option proto 'tcp'
	option dest_ip '<peer_ip>'
	option dest_port '<peer_port>'
	option target 'ACCEPT'

config rule
	option name 'Allow-mimic-udp-out'
	option src '*'
	option direction 'out'
	option dest '*'
	option proto 'udp'
	option dest_ip '<peer_ip>'
	option dest_port '<peer_port>'
	option target 'ACCEPT'
```

If `ct state invalid drop` still trips, considering modifying the egress zone to allow invalid packets from the peer (config below assume `wan` zone for egress interface pppeo-wan):

```sh
config zone
        option name 'wan'
		...
        option masq_allow_invalid '1'
```

### Pick the right interface

For a PPPoE WAN, attach mimic to **`pppoe-wan`**, not the underlying `eth+`.
Mimic's filter (`remote=<ip>:<port>`) reads at L3/L4; on `pppoe-wan` the
packet is at the IP layer with no L2, so the BPF filter matches directly.
On `eth1` every packet is wrapped in Ethernet + 8-byte PPPoE session header
+ 2-byte PPP, which mimic's BPF program does not parse.

```sh
mimic run -v -f remote=<peer_ip>:<peer_port> pppoe-wan -L none
```

`-L none` is the no-L2 link type for PPP. Mimic 0.7.1+ also auto-detects
this for PPP interfaces, so passing `-L` may not be required on the version
this branch ships, but it's harmless.

XDP on PPP is `skb` mode (the driver doesn't support native XDP); that's
expected and shows up in the deploy log.

### Performance / offload tuning

NIC offloads on the underlying physical interface (`eth1` on MT6000) can
silently break a mimic + kernel-WireGuard flow. The reason is *where in the
RX pipeline* each offload runs:

```
wire → eth1 NIC + driver → napi_gro_receive ← GRO merges here
              → PPPoE demux
              → pppoe-wan
              → mimic XDP (skb mode) ← mimic sees post-GRO
              → kernel UDP / WireGuard
```

GRO runs on `eth1` *before* mimic's XDP hook on `pppoe-wan`. Linux has GRO
callbacks for PPPoE-encapsulated flows, so it merges them by the inner-L4
5-tuple. Mimic-disguised traffic looks like a normal TCP flow at `eth1`
level (same 5-tuple, monotonic seq, no PSH on intermediate segments) —
exactly the conditions GRO uses to merge. GRO will happily concatenate
several "fake-WireGuard-as-TCP" packets into one super-segment.

When mimic then transforms that merged super-segment TCP → UDP on
`pppoe-wan`, the resulting UDP payload contains the **concatenated bodies
of several original WireGuard datagrams**. WireGuard expects each UDP
datagram to be one independent encrypted packet; it can't parse a blob
with multiple datagrams glued together, so it drops them. Visible
symptoms: handshake works, throughput collapses, dmesg shows WG decryption
failures.

TSO/GSO are the symmetric problem on egress: kernel WG sends a large UDP
datagram, mimic's TC eBPF rewrites it to TCP, then the NIC's segmentation
offload splits it on TCP framing assumptions that don't reflect WireGuard
boundaries.

| ethtool feature | Default on MT6000 | For mimic | Why |
|---|---|---|---|
| `gro` | on | **off** | Merges pre-XDP, breaks per-datagram framing |
| `lro` (HW LRO) | already off | leave off | We disabled MTK HW LRO when enabling RSS; not in `MT7986_CAPS` |
| `tso` | on | **off** | Egress segmentation reshapes packets after TC eBPF rewrite |
| `gso` | on | **off** | Same reasoning as `tso` |
| `rx-checksum` / `tx-checksum-*` | on | leave on | The `csum_offset` issue is what `kmod-mimic` already fixes; don't disable offload |

Apply at runtime:

```sh
ethtool -K eth1 gro off lro off tso off gso off
```

Persist via a hotplug hook so it re-applies on `eth1` ifup:

```sh
cat >/etc/hotplug.d/iface/99-mimic-offload <<'EOF'
[ "$INTERFACE" = "wan" ] && [ "$ACTION" = "ifup" ] && \
        ethtool -K eth1 gro off lro off tso off gso off 2>/dev/null
EOF
chmod +x /etc/hotplug.d/iface/99-mimic-offload
```

Cost: software segmentation on `eth1` instead of NIC-offloaded. On
MT7986A this is well under the SoC's capability for typical PPPoE WAN
speeds (≤2.5 Gbit), so the throughput impact on non-mimic traffic is
negligible.

## Build verification done in this tree

End-to-end clean world build with mimic + KPROBES enabled produced:

- `mimic` userspace: 151 KB aarch64-musl ELF
- `mimic.ko`: 69 KB (with kprobe hooks; was 61 KB without `KERNEL_KPROBES`)
- `bin/packages/aarch64_cortex-a53/base/mimic-0.7.1.20251120-r1.apk`
- `bin/targets/mediatek/filogic/packages/kmod-mimic-6.18.25.0.7.1.20251120-r1.apk`

Total firmware delta vs pre-mimic: ~+700 KB on `squashfs-sysupgrade.bin`,
mostly from the kprobe/ftrace/perf-events kernel additions.
