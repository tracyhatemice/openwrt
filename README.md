![OpenWrt logo](include/logo.png)

OpenWrt Project is a Linux operating system targeting embedded devices. Instead
of trying to create a single, static firmware, OpenWrt provides a fully
writable filesystem with package management. This frees you from the
application selection and configuration provided by the vendor and allows you
to customize the device through the use of packages to suit any application.
For developers, OpenWrt is the framework to build an application without having
to build a complete firmware around it; for users this means the ability for
full customization, to use the device in ways never envisioned.

Sunshine!

## Fork notes: patches imported over vanilla

This fork tracks OpenWrt `main` plus a small set of MediaTek WiFi and SoC/Ethernet
patches for the **GL.iNet GL-MT6000** (MT7986 + MT7976, target `mediatek/filogic`,
WiFi-6 / mt7915 driver family). Patches are imported on top of vanilla and recorded
here for provenance and traceability.

| Package | What was imported | From | Upstream origin |
|---|---|---|---|
| **kernel / target** (`mediatek/filogic`, 6.18) | 19 MTK patches (`999-*`) added under `target/linux/mediatek/patches-6.18/` (140 total = 117 vanilla + 4 ported `402`/`947`/`948`/`954` + 19 pesa) — `mtk_eth_soc` RSS/LRO, jumbo frames, 2.5G rate limit, NAPI tuning; `mtk_wed` (Wireless Ethernet Dispatch) hwrro/SER/cleanup/reset fixes; plus mt7981/mt7986 DTS (RSS irqs, PMU) and USB power control. `999-9911`/`999-9912` dropped 2026-07-24 as byte-identical subsets of the ported `954` (see PPE/WED port below). `999-2710`/`999-2711` (RSS) re-based 2026-07-25 onto the current feed (`25.12` `999-eth-08`/`999-eth-09`; pesa's copies were the 24.10-era version): `2710` payload-identical to `999-eth-08` (`lro_alt_timer` into the pdma reg map, fixed `MTK_PDMA_LRO_ALT_REFRESH_TIMER` define dropped; its sole `mtk_hwlro_rx_init` user converted in `2711`, mirroring feed `999-eth-10` which we don't carry — HW LRO is off on filogic and would contend with RSS for rx rings 1-3 on netsys_v2); `2711` carries the `999-eth-09` header and form (`desc_shift == const_ilog2()` type checks, feed caps layout) adapted for the 6.18 IRQ rework, keeping the fork hardening (rss_num/caps guards in ethtool ops, indirection-table bounds check + extack, `dev_err_probe` on missing PDMA IRQs) and pesa extras (multi-queue netdev registration + `skb_record_rx_queue`, mt7986 ring sizing tx 4K / rx 1K×4); the ethtool hfunc reporting bug fixed inline (feed's `if (rxfh->hfunc)` is a stale NULL-check from the old `u8 *hfunc` API that never fires on GET) — `999-9903` dropped as absorbed | pesa1234 `next-r4.8.3.rss.mtk` (synced; `2710`/`2711` since re-aligned to the MTK feed form) | MediaTek `mtk-openwrt-feeds` `mtk_eth_soc` / WED downstream |
| **mac80211** | 25 MTK patches into `package/kernel/mac80211/patches/mtk/`, plus 2 lines in `package/kernel/mac80211/Makefile` registering the `mtk/` patch subdir (applied right after `subsys/`) | pesa1234 branch `next-r4.8.3.rss.mtk` (backports **6.18.26**), copied verbatim | MediaTek `mtk-openwrt-feeds` WiFi-6 set (`autobuild/autobuild_5.4_mac80211_release/.../mac80211/patches/subsys/`), which pesa1234 relocates to a `mtk/` subdir |
| **hostapd** | 25 MTK patches into `package/network/services/hostapd/patches/mtk/`, plus a `Build/Patch` block in `package/network/services/hostapd/Makefile` applying the `mtk/` subdir after the base patches. Same hostapd source as vanilla (**`2026-07-09`**, the once cherry-picked bump merged upstream 2026-07-25); `mtk/0013` (EDCCA) + `mtk/0016` (iBF) re-anchored for that base (config parser + SET/GET_EDCCA to the parser/ctrl-iface fallthroughs, `get_ibf` to the end of the hostapd_cli table); vanilla base patches untouched apart from one fork-added FT patch (`022-hostapd-ft-readd-unassociated-sta-before-ptk`, 802.11r — pairs with the mac80211 `400` FT pick) | pesa1234 `next-r4.8.3.rss.mtk`, copied verbatim | MediaTek `mtk-openwrt-feeds` WiFi-6 hostapd set (`autobuild/autobuild_5.4_mac80211_release/.../hostapd/patches`). vs feed: 5 identical, 12 modified (mostly trailer/typo; `0013` EDCCA & `0116` txpower are real rewrites), 8 pesa-original |
| **mt76** | **92** mt7915 patches in `package/kernel/mt76/patches/` (flat), plus Makefile changes (always-on `CONFIG_NL80211_TESTMODE`, `MAC80211_DEBUGFS`, `Build/Compile`). Source pin on **upstream `openwrt/mt76` `50480826`** (2026-07-24), which ships the newer mt7986 firmware **and native HW ATF/VoW** — so no custom fork is needed. Local deltas vs pesa's stack: `2004-wed-HW-ATF` **dropped** (it duplicated upstream's now-native VoW), its developer tuning knobs re-added on top of upstream as `9999-07-…-add-vow-debugfs-tuning-knobs`; `1032-…-Establish-BA-in-VO-queue` **dropped** (upstream now allows TX aggregation on the VO queue natively); `0000_001-…-load-efuse-data` re-anchored over upstream's new `get_eeprom` bounds check; `1012-…-testmode` re-anchored over upstream's `add_interface` failure-unwind (unwind logic inlined into `init_vif`); `2016-…-refine-twt` shrunk (its mac.c unlink-on-reject hunk is now upstream); `9999-08-…-fix-vow-band-after-init-vif-split` moves upstream's `mt7915_mcu_set_vow_band()` into `init_vif()` after pesa's `1012` splits it out of `add_interface()` (also covers testmode's iBF-cal second vif; adopted from pesa's r4.9.1 807e449d22). See the `vow_atf` runtime note below | pesa1234 `next-r4.8.3.rss.mtk` (patches only; source pin on upstream `openwrt/mt76`, **not** a fork) | MediaTek `mtk-openwrt-feeds` mt7915/WiFi-6 set (`autobuild/autobuild_5.4_mac80211_release/.../mt76/patches`) |
| **iwinfo** | 1 patch `patches/0001-nl80211-add-support-for-QAM-256-in-2.4GHz-802.11n` | pesa1234 `next-r4.8.3.rss.mtk` | **pesa-original** (the MTK feed has no iwinfo patches) |
| **wifi-scripts** | EDCCA + `vendor_vht` + iBF/eBF (`itxbfen`/`etxbfen`) config glue **ported into the ucode pipeline** (`files-ucode/usr/share/ucode/wifi/hostapd.uc` + new options in `files-ucode/usr/share/schema/wireless.wifi-device.json`) | pesa1234 had this only in the **shell** variant (`files/`), inert under `CONFIG_WIFI_SCRIPTS_UCODE=y`; re-implemented here for ucode | pesa-original (not in the MTK feed, which carries these only as hostapd C patches) |
| **bpf-headers** | 1-line bump in `package/kernel/bpf-headers/Makefile`: `PKG_PATCHVER` `6.12` → `6.18` to match the `mediatek/filogic` target kernel. bpf-headers reuses the already-downloaded target tarball instead of fetching a separate `linux-6.12.x` kernel at compile time. The 2 fork-carried bpf patches (`100-support_hz_300`, `102-net-flow_offload`) apply cleanly to 6.18 | fork-local build fix | N/A (vanilla pins bpf-headers to the OpenWrt default 6.12 kernel; not a pesa/MTK patch) |

Notes:
- Imported patches apply cleanly on backports 6.18.26 and build `kmod-mac80211` /
  `kmod-cfg80211` for `mediatek/filogic`.
- Everything else under `package/kernel/mac80211/patches/` is unchanged from vanilla;
  the `mtk/` directory (plus the cherry-picked `subsys/400-…-defer-ap-side-ft-key-upload`
  FT patch, see below) is the only addition.
- The feed's default `unified/.../25.12/` profile targets mt7996 / WiFi-7 and is **not**
  used here; the GL-MT6000 uses the mt7915 / WiFi-6 patch set.
- Vanilla `hack-6.18/660-fq_codel_defaults` is **dropped** (2026-07-24): it caps
  fq_codel `memory_limit` at 4 MB on non-x86_64 for small-RAM devices; both targets
  have 1 GB RAM, so the kernel's 32 MB default is kept. On future rebases an
  upstream refresh of this patch shows up as a modify/delete conflict — resolve by
  keeping the deletion.

### Cherry-picked upstream fixes (on top of the pesa import)

A few upstream OpenWrt changes are cherry-picked on top of the MTK import above:

- **mac80211** — `subsys/400-mac80211-defer-ap-side-ft-key-upload`
  (PR 23181, in sync with the PR): defers AP-side FT key upload until station
  association. Pairs with the hostapd `022` FT patch above.
- **mbedtls → `3.6.7`** (PR 24131) — security release (RSA PKCS#1 v1.5 side
  channel, TLS 1.2/1.3 handshake fixes, ECC side channel, many CVEs); drops the
  two backported RSA-PSS patches now included upstream.
- **bridge flow offload** (PR 24038, 12-commit series) — `nft_flow_offload`
  bridge fastpath: generic `pending-6.18/675-*` patches, `kmod-nf-conntrack-bridge`
  (added to filogic default packages), firewall4 bridge-flowtable support, and a
  filogic hotplug script reloading the firewall on bridge-port add. Local note:
  the series' refresh of `hack-6.18/650-…xt_FLOWOFFLOAD…` supersedes this fork's
  offset-only tweaks to the same file (PR side taken). Rebased 2026-07-25 over
  the 6.18.40 flowtable path-discovery split (path code moved to
  `net/netfilter/nf_flow_table_path.c`): `675-03/04/12` + `hack-6.18/290`
  retargeted to the new file, `675-10` split (bridge path builder +
  `if_vlan.h` include in `nf_flow_table_path.c`, eval hunks stay in
  `nft_flow_offload.c`) — carried in the "restore fork flowtable stack" commit
  on top of the verbatim PR 24419 kernel-bump pick; re-diff against PR 24038
  when its author rebases the series onto 6.18.40+.
- **firewall4 bridge-offload hardening** (danpawlik devel branch, commit
  `683ef23b4d`, follow-up to PR 24038) — pulls `kmod-nft-bridge`/`kmod-nft-netdev`/
  `kmod-nf-conntrack-bridge` into firewall4 deps, replaces the per-target hotplug
  scripts with a name-agnostic one shipped by firewall4, guards `hw_ifidx` when
  `hw_outdev` is NULL and records 802.1AD encaps in `675-10`/`675-12`.
- **PPE/WED fixes port** (selective, from danpawlik devel commit `16dd825f5e`,
  originally for BPI-R4/MT7988) — taken: `736-06…11` (PPE MIB-cache enable fix,
  aging-time alignment, cache preserved-line lock, rhashtable leak fix,
  nft_flow_offload thoff fix; `736-06` is netsys_v3-gated no-op), mediatek `947`
  (WED v2 token-FIFO depth fix in txbm reset) and `954` (WED SER fixes: inverted
  `wdma_rx_reset` ring-skip, hwrro double-free). **Not taken:** RSS `950/951`
  (duplicate of pesa `999-2710/2711`), PPPQ QoS `952/953` (netsys_v3-only),
  `731` forced-reset (defaults auto-SER off), mt76 mt7996 MLD fix (already in
  the mt76 pin).
- **xfrm/IPsec offload chain** (adopted 2026-07-25 from danpawlik's branch, MTK
  feed 25.12 originals `999-crypto-04/05` + `999-ppe-07/29/30/31/32`; dan's
  copies verified against the feed — his only deltas are an `IS_BUILTIN` guard,
  a switch `default`, the self-contained `TPORT_EIP197_QDMA` define, the proper
  `dst_xfrm()` accessor and typo fixes): hack `652` + pending
  `676-01/02/03/05` + `736-12` + mediatek `948`. `676-05`/`652` are the only
  live behavior change (forward-path resolution now also runs for XFRM flows);
  the rest is inline-IPsec groundwork that stays **dormant** — nothing
  registers `mtk_flow_offload_get_cdrt` (needs the feed's EIP-197 inline
  driver, MT7988-class), `CONFIG_XFRM` is off in both device configs, and
  `948`'s teardown call compiles out with `nf_flow_table=m`. Caveat if IPsec
  is ever enabled: `676-01` relaxes FWD-direction policy template checks
  unconditionally (feed design for inline-decrypted packets that carry no
  sec_path). Companion: mediatek `402-crypto-inside-secure-avoid-rcu-stall`
  (feed `999-crypto-03`, unmodified) — generic safexcel result-path fix
  (per-request `local_bh` churn + unbounded overflow re-loop) relevant once
  `kmod-crypto-hw-safexcel` accelerates ESP on the EIP-97; the feed's
  `999-crypto-01` (EIP-197 minifw/clock-force, MT7988-class — the fix for the
  AES-GCM drops in upstream issue 21310 on BPi-R4) is **not taken**: its paths
  are inert on EIP-97 and it renames the EIP197 firmware dir for other boards.

### Configuration flags (MTK extensions)

All of these have safe defaults — **none require configuration to boot**. With an
unmodified config, the generator behaves as follows:

| Flag | Default | Default behaviour (no user config) |
|---|---|---|
| EDCCA | `edcca_enable=1` | Auto-enabled with `compensation=-6`, thresholds `-60/-62/-59/-54` — emitted even when `/etc/config/advanced` is absent |
| `vendor_vht` | `0` (off) | No VHT/256-QAM on 2.4 GHz; standard 2.4 GHz operation (no `vendor_vht`/`ieee80211ac`/VHT caps emitted) |
| `itxbfen` (iBF) | unset | `ibf_enable` **not emitted** → driver/hostapd default implicit-BF behaviour |
| `etxbfen` (eBF) | `1` (on) | SU/MU beamformer + beamformee caps kept (explicit BF enabled); `etxbfen=0` drops them |
| `lpi_enable` | `0` | `lpi_enable=0` emitted → Low Power Indoor off |
| `sku_idx` | `0` | `sku_idx=0` emitted → default regulatory SKU / power table |
| `beacon_dup` | `1` | `beacon_dup=1` emitted → beacon duplication on |
| `obss_interval` | n/a (fixed) | `obss_interval=300` emitted only when `ht_coex` is enabled; otherwise not emitted |

Per-flag detail and config syntax follow.

**EDCCA** (Energy Detection CCA) — read by the ucode hostapd config generator from
`/etc/config/advanced`, applied per radio. Create an `edcca` section:

```
config edcca
	option edcca_enable    '1'    # 1=auto (default), 0=force-disable
	option compensation    '-6'   # dBm, range -126..126 (default -6)
	option thres_0         '-60'  # BW20  threshold (dBm)
	option thres_1         '-62'  # BW40  threshold
	option thres_2         '-59'  # BW80  threshold
	option thres_3         '-54'  # BW160 threshold
```

If the section/options are absent, the defaults above are used. Emits
`edcca_enable` / `edcca_compensation` / `edcca_threshold` to the hostapd config
(keys provided by the `mtk/` hostapd EDCCA patch).

**vendor_vht** — per-device option in `/etc/config/wireless` (`config wifi-device`),
boolean, to enable VHT (256-QAM) on 2.4 GHz:

```
config wifi-device 'radio0'
	option vendor_vht '1'
```

When set on a 2.4 GHz device, the ucode generator forces `ieee80211ac=1` and emits
the VHT capabilities (`vht_capab`, `vht_oper_chwidth`, `vht_oper_centr_freq_seg0_idx`)
for the 2.4 GHz radio in addition to `vendor_vht=1` — mirroring pesa's shell
`enable_ac || vendor_vht` logic. Normal 2.4 GHz operation (without `vendor_vht`) is
unaffected.

**iBF / eBF** — per-device options in `/etc/config/wireless` (`config wifi-device`):

```
config wifi-device 'radio0'
	option itxbfen '1'    # implicit TX beamforming -> hostapd ibf_enable=1 (omit option to leave unset)
	option etxbfen '1'    # explicit TX beamforming (default 1); set 0 to drop SU/MU beamformer+beamformee caps
```

`itxbfen` emits `ibf_enable=<0|1>` only when the option is present. `etxbfen=0`
clears `su_beamformer`/`su_beamformee`/`mu_beamformer` (VHT) and
`he_su_beamformer`/`he_mu_beamformer` (HE) before the capability strings are built.
The `ibf_enable` hostapd key is provided by the imported `mtk/` iBF hostapd patch.

**lpi_enable / sku_idx / beacon_dup** — per-device options in `/etc/config/wireless`
(`config wifi-device`), emitted on every radio with defaults (`lpi_enable=0`,
`sku_idx=0`, `beacon_dup=1`) and user-overridable:

```
config wifi-device 'radio0'
	option lpi_enable '0'   # Low Power Indoor mode
	option sku_idx    '0'   # regulatory SKU / power-table index
	option beacon_dup '1'   # beacon duplication
```

These hostapd keys are provided by the imported `mtk/` hostapd + mt76 patches.

`obss_interval` is not a user option: like pesa, the generator emits a fixed
`obss_interval=300` whenever `ht_coex` is enabled on the radio.

### Runtime: HW airtime fairness (`vow_atf`)

Upstream `openwrt/mt76` gained native **hardware airtime fairness (ATF)** — the
MAC's weighted-DWRR (VoW) scheduler balancing TX airtime between stations on the
offloaded/WED path where the host has no airtime accounting — in
[openwrt/mt76 `6b7b986`](https://github.com/openwrt/mt76/commit/6b7b98627bcc0b31423c52b730a8e3c8ed9a3349).
This fork tracks upstream mt76, so it is present on the **GL-MT6000** (mt7986) and
**OpenWrt One** (mt7981); mt7915 is excluded.

It is **enabled by default** and toggled at runtime via debugfs (device-wide; ATF
and WATF together):

```
cat  /sys/kernel/debug/ieee80211/phy0/mt76/vow_atf     # 1 = on, 0 = off
echo 0 > /sys/kernel/debug/ieee80211/phy0/mt76/vow_atf # disable
echo 1 > /sys/kernel/debug/ieee80211/phy0/mt76/vow_atf # enable
```

The phy number can vary — find it with
`ls -d /sys/kernel/debug/ieee80211/phy*/mt76/vow_atf` — and the setting is **not**
persistent across reboot or Wi-Fi restart (re-apply from a hotplug/init hook if
needed). The fork also adds a sibling `…/mt76/vow` file (patch `9999-07`) for finer
VoW tuning (WATF per-level quantum, per-station DRR quantum, deficit bound, dumps);
`vow_atf` is the plain on/off switch.

## Download

Built firmware images are available for many architectures and come with a
package selection to be used as WiFi home router. To quickly find a factory
image usable to migrate from a vendor stock firmware to OpenWrt, try the
*Firmware Selector*.

* [OpenWrt Firmware Selector](https://firmware-selector.openwrt.org/)

If your device is supported, please follow the **Info** link to see install
instructions or consult the support resources listed below.

##

An advanced user may require additional or specific package. (Toolchain, SDK, ...) For everything else than simple firmware download, try the wiki download page:

* [OpenWrt Wiki Download](https://openwrt.org/downloads)

## Development

To build your own firmware you need a GNU/Linux, BSD or macOS system (case
sensitive filesystem required). Cygwin is unsupported because of the lack of a
case sensitive file system.

### Requirements

You need the following tools to compile OpenWrt, the package names vary between
distributions. A complete list with distribution specific packages is found in
the [Build System Setup](https://openwrt.org/docs/guide-developer/build-system/install-buildsystem)
documentation.

```
binutils bzip2 diff find flex gawk gcc-6+ getopt grep install libc-dev libz-dev
make4.1+ perl python3.8+ rsync subversion unzip which
```

### Quickstart

1. Run `./scripts/feeds update -a` to obtain all the latest package definitions
   defined in feeds.conf / feeds.conf.default

2. Run `./scripts/feeds install -a` to install symlinks for all obtained
   packages into package/feeds/

3. Run `make menuconfig` to select your preferred configuration for the
   toolchain, target system & firmware packages.

4. Run `make` to build your firmware. This will download all sources, build the
   cross-compile toolchain and then cross-compile the GNU/Linux kernel & all chosen
   applications for your target system.

### Related Repositories

The main repository uses multiple sub-repositories to manage packages of
different categories. All packages are installed via the OpenWrt package
manager called `opkg`. If you're looking to develop the web interface or port
packages to OpenWrt, please find the fitting repository below.

* [LuCI Web Interface](https://github.com/openwrt/luci): Modern and modular
  interface to control the device via a web browser.

* [OpenWrt Packages](https://github.com/openwrt/packages): Community repository
  of ported packages.

* [OpenWrt Routing](https://github.com/openwrt/routing): Packages specifically
  focused on (mesh) routing.

* [OpenWrt Video](https://github.com/openwrt/video): Packages specifically
  focused on display servers and clients (Xorg and Wayland).

## Support Information

For a list of supported devices see the [OpenWrt Hardware Database](https://openwrt.org/supported_devices)

### Documentation

* [Quick Start Guide](https://openwrt.org/docs/guide-quick-start/start)
* [User Guide](https://openwrt.org/docs/guide-user/start)
* [Developer Documentation](https://openwrt.org/docs/guide-developer/start)
* [Technical Reference](https://openwrt.org/docs/techref/start)

### Support Community

* [Forum](https://forum.openwrt.org): For usage, projects, discussions and hardware advise.
* [Support Chat](https://webchat.oftc.net/#openwrt): Channel `#openwrt` on **oftc.net**.

### Developer Community

* [Bug Reports](https://bugs.openwrt.org): Report bugs in OpenWrt
* [Dev Mailing List](https://lists.openwrt.org/mailman/listinfo/openwrt-devel): Send patches
* [Dev Chat](https://webchat.oftc.net/#openwrt-devel): Channel `#openwrt-devel` on **oftc.net**.

## License

OpenWrt is licensed under GPL-2.0
