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
| **kernel / target** (`mediatek/filogic`, 6.18) | 22 MTK patches (`999-*`) added under `target/linux/mediatek/patches-6.18/` (139 total = 117 vanilla + 22 custom) — `mtk_eth_soc` RSS/LRO, jumbo frames, 2.5G rate limit, NAPI tuning; `mtk_wed` (Wireless Ethernet Dispatch) hwrro/SER/cleanup/reset fixes; plus mt7981/mt7986 DTS (RSS irqs, PMU) and USB power control. Pure additions; no vanilla patch modified | pesa1234 `next-r4.8.3.rss.mtk` (synced verbatim) | MediaTek `mtk-openwrt-feeds` `mtk_eth_soc` / WED downstream |
| **mac80211** | 25 MTK patches into `package/kernel/mac80211/patches/mtk/`, plus 2 lines in `package/kernel/mac80211/Makefile` registering the `mtk/` patch subdir (applied right after `subsys/`) | pesa1234 branch `next-r4.8.3.rss.mtk` (backports **6.18.26**), copied verbatim | MediaTek `mtk-openwrt-feeds` WiFi-6 set (`autobuild/autobuild_5.4_mac80211_release/.../mac80211/patches/subsys/`), which pesa1234 relocates to a `mtk/` subdir |
| **hostapd** | 25 MTK patches into `package/network/services/hostapd/patches/mtk/`, plus a `Build/Patch` block in `package/network/services/hostapd/Makefile` applying the `mtk/` subdir after the base patches. Same hostapd source as vanilla (`2026-04-02`); the 59 base OpenWrt patches are untouched | pesa1234 `next-r4.8.3.rss.mtk`, copied verbatim | MediaTek `mtk-openwrt-feeds` WiFi-6 hostapd set (`autobuild/autobuild_5.4_mac80211_release/.../hostapd/patches`). vs feed: 5 identical, 12 modified (mostly trailer/typo; `0013` EDCCA & `0116` txpower are real rewrites), 8 pesa-original |
| **mt76** | 92 mt7915 patches in `package/kernel/mt76/patches/` (flat), plus Makefile changes (always-on `CONFIG_NL80211_TESTMODE`, `MAC80211_DEBUGFS`, `Build/Compile`). Source pin kept on **upstream `openwrt/mt76`** and bumped to the latest commit `2ab64980` (2026-06-02); 2 of pesa's 94 patches (`002-hrtimer_setup`, `003-ccflags`) dropped as already upstreamed | pesa1234 `next-r4.8.3.rss.mtk` (patches only; source pin **not** switched to pesa's fork) | MediaTek `mtk-openwrt-feeds` mt7915/WiFi-6 set (`autobuild/autobuild_5.4_mac80211_release/.../mt76/patches`) |
| **iwinfo** | 1 patch `patches/0001-nl80211-add-support-for-QAM-256-in-2.4GHz-802.11n` | pesa1234 `next-r4.8.3.rss.mtk` | **pesa-original** (the MTK feed has no iwinfo patches) |
| **wifi-scripts** | EDCCA + `vendor_vht` + iBF/eBF (`itxbfen`/`etxbfen`) config glue **ported into the ucode pipeline** (`files-ucode/usr/share/ucode/wifi/hostapd.uc` + new options in `files-ucode/usr/share/schema/wireless.wifi-device.json`) | pesa1234 had this only in the **shell** variant (`files/`), inert under `CONFIG_WIFI_SCRIPTS_UCODE=y`; re-implemented here for ucode | pesa-original (not in the MTK feed, which carries these only as hostapd C patches) |

Notes:
- Imported patches apply cleanly on backports 6.18.26 and build `kmod-mac80211` /
  `kmod-cfg80211` for `mediatek/filogic`.
- Everything else under `package/kernel/mac80211/patches/` is unchanged from vanilla;
  the `mtk/` directory is the only addition.
- The feed's default `unified/.../25.12/` profile targets mt7996 / WiFi-7 and is **not**
  used here; the GL-MT6000 uses the mt7915 / WiFi-6 patch set.

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
