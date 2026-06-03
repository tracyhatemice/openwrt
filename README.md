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

Notes:
- Imported patches apply cleanly on backports 6.18.26 and build `kmod-mac80211` /
  `kmod-cfg80211` for `mediatek/filogic`.
- Everything else under `package/kernel/mac80211/patches/` is unchanged from vanilla;
  the `mtk/` directory is the only addition.
- The feed's default `unified/.../25.12/` profile targets mt7996 / WiFi-7 and is **not**
  used here; the GL-MT6000 uses the mt7915 / WiFi-6 patch set.

_Planned: mt76 driver patches (pesa1234 `next-r4.8.3.rss.mtk`) — not yet imported._

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
