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
| **kernel / target** (`mediatek/filogic`, 6.18) | 22 MTK patches (`999-*`) added under `target/linux/mediatek/patches-6.18/`, **27** fork-added files in that directory in total (`999-*` plus `197`, `402`, `947`, `948`, `954`), with **none** removed. Verify the delta — not the total — with `comm -23 <(ls target/linux/mediatek/patches-6.18/ \| sort) <(git ls-tree --name-only vanilla/main target/linux/mediatek/patches-6.18/ \| sed 's\|.*/\|\|' \| sort) \| wc -l`. The **absolute** count (163 as of 2026-09-04, against 136 in vanilla) is not a useful figure and should not be quoted here: it moves whenever upstream adds or drops a mediatek patch of its own, independently of this fork. It has already gone stale twice this way — an old "140 = 117 vanilla + …" breakdown was wrong before this branch even started, and the "156" recorded on 2026-08-30 drifted to 163 purely because upstream landed seven mediatek patches (Adtran SmartRG support and the `leds-smartrg` series) in the 2026-09-04 sync — `mtk_eth_soc` RSS/LRO, jumbo frames, 2.5G rate limit, NAPI tuning; `mtk_wed` (Wireless Ethernet Dispatch) hwrro/SER/cleanup/reset fixes; plus mt7981/mt7986 DTS (RSS irqs, PMU) and USB power control. `999-9911`/`999-9912` dropped 2026-07-24 as byte-identical subsets of the ported `954` (see PPE/WED port below). `999-2710`/`999-2711` (RSS) re-based 2026-07-25 onto the current feed (`25.12` `999-eth-08`/`999-eth-09`; pesa's copies were the 24.10-era version): `2710` payload-identical to `999-eth-08` (`lro_alt_timer` into the pdma reg map, fixed `MTK_PDMA_LRO_ALT_REFRESH_TIMER` define dropped; its sole `mtk_hwlro_rx_init` user converted in `2711`, mirroring feed `999-eth-10` which we don't carry — HW LRO is off on filogic and would contend with RSS for rx rings 1-3 on netsys_v2); `2711` carries the `999-eth-09` header and form (`desc_shift == const_ilog2()` type checks, feed caps layout) adapted for the 6.18 IRQ rework, keeping the fork hardening (rss_num/caps guards in ethtool ops, indirection-table bounds check + extack, `dev_err_probe` on missing PDMA IRQs) and pesa extras (multi-queue netdev registration + `skb_record_rx_queue`, mt7986 ring sizing tx 4K / rx 1K×4); the ethtool hfunc reporting bug fixed inline (feed's `if (rxfh->hfunc)` is a stale NULL-check from the old `u8 *hfunc` API that never fires on GET) — `999-9903` dropped as absorbed. Four more MTK patches ported 2026-08-30 from pesa1234 `next-r4.9.2.rss.mtk` (152 → **156**): `197-…correct-timer-frequency` (the MT7986 architected timer actually runs at 12,986,200 Hz, not the 13 MHz firmware reports via `CNTFRQ_EL0`; describing the real rate fixes system clock drift and the resulting NTP saturation — MediaTek feed original `6a4c41c414`, pesa's copy `b2fea8cd4b`; **since 2026-09-04 this file is the verbatim copy from upstream PR 24943**, pesa1234's own submission of the same fix, so it auto-drops on merge — see the cherry-pick list below) and `999-9919`/`999-9920`/`999-9921` (`mtk_eth_soc`: optimise SRAM RX descriptor reads, switch to a per-MAC SGMII syscfg0 mask instead of a shared one, fix a spurious MDIO timeout — pesa `de30e8730a`/`f0f707ee4e`/`406ea292e3`) | pesa1234 `next-r4.8.3.rss.mtk` (synced; `2710`/`2711` since re-aligned to the MTK feed form); 4 new 2026-08-30 patches from `next-r4.9.2.rss.mtk` | MediaTek `mtk-openwrt-feeds` `mtk_eth_soc` / WED downstream |
| **mac80211** | 25 MTK patches into `package/kernel/mac80211/patches/mtk/`, plus 2 lines in `package/kernel/mac80211/Makefile` registering the `mtk/` patch subdir (applied right after `subsys/`) | pesa1234 branch `next-r4.8.3.rss.mtk` (backports **6.18.26**), copied verbatim | MediaTek `mtk-openwrt-feeds` WiFi-6 set (`autobuild/autobuild_5.4_mac80211_release/.../mac80211/patches/subsys/`), which pesa1234 relocates to a `mtk/` subdir |
| **hostapd** | 25 MTK patches into `package/network/services/hostapd/patches/mtk/`, plus a `Build/Patch` block in `package/network/services/hostapd/Makefile` applying the `mtk/` subdir after the base patches. Same hostapd source as vanilla, currently **`2.12`** (`2026-08-07`, merged upstream 2026-08-12 as PR 24599); `mtk/0013` (EDCCA), `mtk/0014` (MU), `mtk/0016` (iBF) and `mtk/0041` (STA TX queue params) re-anchored for that base (2026-08-06: upstream's new `CONFIG_AFC` block shifted the `hostapd_config` tail and the ctrl_iface dispatch chain; `wpa_supplicant_event_assoc()` now returns a value) (config parser + SET/GET_EDCCA to the parser/ctrl-iface fallthroughs, `get_ibf` to the end of the hostapd_cli table); vanilla base patches untouched apart from one fork-added FT patch (`022-hostapd-ft-readd-unassociated-sta-before-ptk`, 802.11r — pairs with the mac80211 `400` FT pick). BW160 EDCCA read-back was **actually broken end-to-end before this branch**, not merely incomplete: `mtk/0013`'s `edcca_info_handler()` requires `MTK_VENDOR_ATTR_EDCCA_DUMP_SEC160_VAL` in the vendor dump reply and `return NL_SKIP`s (leaving the fourth `edcca_threshold` value unpopulated) when it's absent, but the mt76 driver's dump handler never sent that attribute — only `PRI20`/`SEC40`/`SEC80`. Fixed since 2026-08-30: the mt76 vendor dump reply now also returns `SEC160_VAL` (`9999-04`, `nla_put_u8`), so read-back tooling sees the BW160 threshold that was set via the ucode pipeline's `edcca_threshold` list (`thres_0..thres_3`, `hostapd.uc:579-583`). pesa1234's r4.9.2 hostapd delta (`599c78e55d`, touching `mtk/0013`/`0015`/`0016`/`0020`/`0041`/`0116`) was **not** taken here — this fork re-anchored those same patches for the hostapd 2.12 base independently, in `dc0fb275b0` | pesa1234 `next-r4.8.3.rss.mtk`, copied verbatim | MediaTek `mtk-openwrt-feeds` WiFi-6 hostapd set (`autobuild/autobuild_5.4_mac80211_release/.../hostapd/patches`). vs feed: 5 identical, 12 modified (mostly trailer/typo; `0013` EDCCA & `0116` txpower are real rewrites), 8 pesa-original |
| **mt76** | **101** mt7915 patches in `package/kernel/mt76/patches/` (flat), plus Makefile changes (always-on `CONFIG_NL80211_TESTMODE`, `MAC80211_DEBUGFS`, `Build/Compile`). Source pin on **upstream `openwrt/mt76` `be5ce791`** (2026-09-01), which ships the newer mt7986 firmware **and native HW ATF/VoW** — so no custom fork is needed. Local deltas vs pesa's stack: `2004-wed-HW-ATF` **dropped** (it duplicated upstream's now-native VoW), its developer tuning knobs re-added on top of upstream as `9999-07-…-add-vow-debugfs-tuning-knobs`; `1032-…-Establish-BA-in-VO-queue` **dropped** (upstream now allows TX aggregation on the VO queue natively); `0000_001-…-load-efuse-data` re-anchored over upstream's new `get_eeprom` bounds check; `1012-…-testmode` re-anchored over upstream's `add_interface` failure-unwind (unwind logic inlined into `init_vif`); `2016-…-refine-twt` shrunk (its mac.c unlink-on-reject hunk is now upstream); `9999-08-…-fix-vow-band-after-init-vif-split` moves upstream's `mt7915_mcu_set_vow_band()` into `init_vif()` after pesa's `1012` splits it out of `add_interface()` (also covers testmode's iBF-cal second vif; adopted from pesa's r4.9.1 807e449d22). **2026-08-30 r4.9.2 import** (92 → **102**): 9 of pesa's 10 new fork-local airtime/air-monitor patches taken verbatim from `next-r4.9.2.rss.mtk` — `9999-09` (harden-air-monitor-vendor-abi), `9999-10` (serialize-air-monitor-state), `9999-11` (survey-spike-filter-time-aware), `9999-12` (keep-airtime-counters-driver-private), `9999-13` (scale-airtime-filter-with-poll-delay), `9999-14` (poll-airtime-once-per-device), `9999-15` (defer-airtime-counter-reset), `9999-16` (airtime-feature-param-read-only) and `9999-18` (fix-kernel-panic-in-tx-scheduling-on-dual-band-Wi-Fi-6). `9999-17` (fix-stranded-frames-in-tx-scheduler, pesa `bec97cd705`) is **not** imported: it's Felix Fietkau's upstream mt76 fix dated 2026-07-22, already carried by our pin `2c84469c` (2026-08-18) — applying it triggers "Reversed (or previously applied) patch detected". So the imported run is `9999-09` through `9999-16` plus `9999-18`, **not** a contiguous `09..18`. Four of the nine (`9999-10`, `9999-11`, `9999-15`, `9999-16`) needed re-anchoring, across five failing hunks with **two distinct causes** — neither is drift between the two forks' mt76 pins. (1) **Truncated trailing context**: `9999-11` hunk #3 and `9999-15`'s `mac.c` hunk carry trailing context 1-2 lines shorter than a standard unified diff, which GNU patch's `-F0` matcher refuses even when every line present, including the one it truncated after, matches the tree byte for byte. (2) **Stale quoted context text**, unrelated to length — full 3-line context, but the context itself no longer matches either tree: `9999-16` quotes the pre-expansion register-table form `[INT_SOURCE_CSR] = MT_INT_SOURCE_CSR,` where both trees have long since carried the literal `[INT_SOURCE_CSR] = 0xd7010,`; `9999-15`'s `mt7915.h` hunk quotes the old `mt7915_mac_enable_nf(struct mt7915_dev *dev, u8 band);` prototype where both trees have the renamed `bool ext_phy` form; `9999-10` quotes a three-tab continuation-line indent for `mt7915_amnt_dump()` where both trees use two tabs. In every one of these, pesa1234's own tree already contradicts his own `9999-*` hunk elsewhere in his stack (e.g. his `1054-…` already quotes `bool ext_phy` as context earlier). Proven by running his full series against **his own pin** `59676919` at `-F0`: it fails on these same four patches for the same reasons (his quilt build never notices, since OpenWrt's default patch invocation allows fuzz). Only context lines and `@@` hunk headers were corrected; every `+`/`-` payload line stays byte-identical to pesa's originals. Expect the same on the next sync. `9999-03` (harden-all-sta-airtime-event) now **diverges** from the import: the `skb->len` bounds check is reordered to run before the `res->sta_num` read it was meant to guard, fixing a two-byte out-of-bounds read the imported hunk left in place — a deliberate break from this row's "imported patches stay verbatim" rule, since the defect is in the patch pesa shipped, and his own `9999-13` expects the check-then-read ordering as context. New fork-local **`9999-20`** (harden-local-patch-runtime-guards) fixes three defects in patches this fork already carries: an under-counting loop in `9510`'s `mt7915_mcu_rx_bss_acq_pkt_cnt()` (a bitmap test was used as the loop *terminator*, so counting stopped at the first unset bit instead of scanning the whole bitmap), a NULL `sta` dereference in `0012`'s `mt76_tx()` warning path, and an unguarded list-head dereference in `1008`'s testmode iteration — adapted from pesa's `9999-07-…-harden-local-patch-runtime-guards` (`9a330637f4`), which has five hunks; **two** were deliberately dropped rather than imported as part of `9999-20`, not one: the `mt7915_init_vif` `-ENOSPC` unwind hunk, since upstream mt76 added the same unwind between his pin `59676919` and ours `2c84469c`, already inherited here via the `1012` re-anchor (`96c673c41e`); and his `@@ -535,9 +542,9 @@` hunk (the `sta_num`/`skb->len` reorder), folded into `9999-03` instead (see that patch's divergence, above) since `9999-03` is where the bug it fixes actually lives — the net final source for both hunks is identical to pesa's intended final state, only relocated. `9999-20` itself carries the remaining three hunks (bitmap-terminator under-count, NULL `sta` deref, unguarded list-head deref). `9999-04` (add-edcca-bw160-support) extended to report BW160 EDCCA in the vendor dump reply (`nla_put_u8` for `MTK_VENDOR_ATTR_EDCCA_DUMP_SEC160_VAL`, which the control path already accepted but the dump handler omitted) and to initialise `edcca_compensation = 0` on paths `1015` leaves uninitialised — ported from pesa's r4.9.2 `599c78e55d`; that commit's hostapd `mtk/0013`/`0015`/`0016`/`0020`/`0041`/`0116` churn is **not** taken, being offset noise from re-anchoring his stack onto hostapd 2.12, which this fork did independently in `dc0fb275b0`. Like the rest of this row, these are single-author driver changes with no upstream review, kept as a contiguous block (`9999-09`..`9999-16`, `9999-18`) so they can be dropped cheaply if the device misbehaves. **mac80211 ceiling (historical, now lifted):** while this fork was on backports `6.18.39`, mt76 could not go past `2c84469c` — the next three upstream commits (`d49721c205` action code moved out of per-type frame structs, `67c5f7ae25` unsolicited probe response template by link ID, `254a3442b3` FILS discovery template by link ID) need `IEEE80211_MIN_ACTION_SIZE(type)` as a macro, an un-nested `u.action.addba_req`, and the 3-argument `ieee80211_get_fils_discovery_tmpl()`. openwrt/main took backports `7.2` on 2026-08-31 (`bf44cb5fb0`), so the ceiling is gone and the pin now tracks upstream HEAD. **2026-08-31 pin bump** (`c5a3bd91` -> `6d1c6a75`, +46 upstream commits, mostly mt7925 NAN work plus several mt7915 fixes): `0020-…-add-additional-chain-signal-info` **dropped** — upstream's `c0af238844` makes the same change via the new `mt7915_band_chainmask(phy)` helper (`chainmask >> mt7915_band_chainshift(phy)`), which is our patch's computation in cleaner form. `0004-…-fix-txpower-issues` **reduced to its two `debugfs.c` hunks**: only its `mt7915/main.c` hunk was upstreamed (as `8baa3813f4`, SKU power limits after antenna change) — note `patch` reports "Reversed (or previously applied)" if ANY hunk is reversed, so check per-hunk before dropping a whole patch; dropping all of `0004` breaks `1033`, which builds on its `_len` macro. `1008-…-testmode` hunks 19/20 re-anchored onto upstream's `mt7915_band_chainmask()` refactor, **keeping** our `mt7915_tm_check_antenna()` helper: it tests `tx_antenna_mask & ~chainmask` (correct bitmask semantics) where upstream compares numerically with `>`, and logs both values on rejection. `2005-…-conditionally-hide-airtime-fairness` re-anchored for upstream's new `dev->init_wiphy = NULL;` line in `mt76_alloc_device()`. **2026-09-04 pin bump** (`6d1c6a75` -> `be5ce791`, +11 upstream commits): openwrt/main took the same pin in `a46721f04d`, so the `PKG_SOURCE_*` block is once again byte-identical to vanilla and the fork's mt76 Makefile delta reduces to the always-on `NL80211_TESTMODE` / `MAC80211_DEBUGFS` flags and the `Build/Compile` override. All **101** patches apply at `-F0` with zero fuzz; none needed re-anchoring and none became redundant. Three of the eleven matter here: `a57185c` adds the missing `napi_disable()` loop to `mt7915_stop_hardware()` before `mt7915_mcu_exit()`; `0898393` gates the bufferable-MMPDU check in `mt76_txq_schedule_pending_wcid()` on the new `MT_WCID_TX_INFO_SET` bit so frames to a departed station take the PSD path — this does **not** duplicate the fork's `1050-…-assign-DEAUTH-to-ALTX-queue`, which lives in `mt76_connac_mac.c`, fires only under `ieee80211_is_cert_mode()` and rewrites `MT_TXD0_Q_IDX` for deauth specifically, so `1050` is kept; and `d73f612` is the mt7996 twin of the fork's mt7915 `6013-…-avoid-list-corruption-on-failure-to-add-TWT`. Upstream also deleted its last mt76 patch (`100-mac80211-support-kernel-version-7.1`) in `6c315233aa`, which this fork had already removed with the import — so that deletion is no longer a fork delta. See the `vow_atf` runtime note below | pesa1234 `next-r4.8.3.rss.mtk` (patches only; source pin on upstream `openwrt/mt76`, **not** a fork); `9999-09..16`/`18`/`20` and the `9999-03`/`9999-04` deltas are from `next-r4.9.2.rss.mtk` (2026-08-30) | MediaTek `mtk-openwrt-feeds` mt7915/WiFi-6 set (`autobuild/autobuild_5.4_mac80211_release/.../mt76/patches`) |
| **iwinfo** | 1 patch `patches/0001-nl80211-add-support-for-QAM-256-in-2.4GHz-802.11n` | pesa1234 `next-r4.8.3.rss.mtk` | **pesa-original** (the MTK feed has no iwinfo patches) |
| **wifi-scripts** | EDCCA + `vendor_vht` + iBF/eBF (`itxbfen`/`etxbfen`) config glue **ported into the ucode pipeline** (`files-ucode/usr/share/ucode/wifi/hostapd.uc` + new options in `files-ucode/usr/share/schema/wireless.wifi-device.json`). A new `vow_atf` per-radio boolean (2026-08-30) is wired into `files-ucode/lib/netifd/wireless/mac80211.sh`'s `setup_phy()`, writing `/sys/kernel/debug/ieee80211/<phy>/mt76/vow_atf` at radio bring-up when set (schema addition in the same `wireless.wifi-device.json`) — see the `vow_atf` runtime note below | pesa1234 had the EDCCA/`vendor_vht`/iBF-eBF glue only in the **shell** variant (`files/`), inert under `CONFIG_WIFI_SCRIPTS_UCODE=y`; re-implemented here for ucode. `vow_atf` has no pesa ucode/shell equivalent — it replaces the runtime ATF handling in his `advanced_setup` init.d script, which this fork does not carry (see below) | pesa-original (not in the MTK feed, which carries these only as hostapd C patches) |
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
  (PR 23181, in sync with the PR head `75311aa2c8`): defers AP-side FT key
  upload until station association. Pairs with the hostapd `022` FT patch
  above. The PR was rewritten and re-anchored for backports 7.2 on
  2026-09-04 and this fork re-synced to it, dropping its own 7.2 re-anchor:
  the new version builds the EPP conjunction upstream-side (so the local
  edit that preserved upstream's Enhanced Privacy Protection exception is
  no longer needed), marks deferred keys with an explicit
  `KEY_FLAG_DEFERRED_HW_UPLOAD` and uploads them from a single hook at the
  AUTH→ASSOC transition walking `sta->ptk[]`, and — unlike the fork's
  version — refuses to return `ret = 1` from the unsupported path,
  excluding `SW_CRYPTO_CONTROL` drivers from deferral outright instead of
  faking driver permission for software crypto. Inert for mt76, which does
  not set that flag, but it removes a latent trap. Applies at zero fuzz and
  zero offset.
- **kernel 6.18.45 → .49** (PR 24800, all five bump commits picked
  verbatim, patch-id identical to the PR's, so they auto-drop when it
  merges) plus one fork-local commit, `generic: restore fork patch anchors
  over the 6.18.48/.49 bumps`, holding the fork side that the verbatim
  picks cannot carry: hunk-header re-anchoring of `291`, `650` and
  `736-12`, all three of which sit on `net/netfilter/nf_flow_table_core.c`
  that `.48` grew by a line. Payload is untouched. `.49` additionally
  drops two patches upstreamed into that release
  (`backport-6.18/501-v7.1-ksmbd-harden-file-lifetime…`,
  `894-v7.3-usb-xhci-handle-port-events…`). All fork patches apply to .49
  with zero fuzz and a following `make target/linux/refresh` is a no-op.
  The .46 pick conflicts on `hack-6.18/650-…xt_FLOWOFFLOAD` — keep OURS (it
  carries `DEV_PATH_BR_VLAN_KEEP_HW`, which upstream lacks), then
  `make target/linux/refresh` normalises the hunk headers.
- **WED 2.0 WDMA TX hang fix** (PR 24784) — raises the WED v2 WDMA `RESV_BUFF`
  from 0x40 to 0x80 to avoid a CDM TX FIFO overflow that hangs WDMA TX on
  **mt7986 and mt7981** (both fork targets); pulled by upstream from
  `mtk-openwrt-feeds`. **Re-synced 2026-08-28:** upstream reworked the PR from a
  MediaTek target patch into a generic backport, so the fork now carries
  `generic/backport-{6.12,6.18}/734-v7.3-…RESV_BUFF-…patch` (a verbatim pick of
  the PR head, so it will auto-drop on merge) and the old
  `mediatek/patches-6.18/756-…` has been dropped. Keeping the stale 756 would
  have collided with upstream's 734 at merge time (same change, two paths).
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
  on top of the verbatim PR 24419 kernel-bump pick. PR 24038's v3 (head `c33c99717d`,
  2026-08-03) renumbers the series to `675-01..14` and rebases it onto the same
  6.18.40 path-discovery split; re-diffed 2026-08-02, the union payload differs
  by only ~22 lines. The fork deliberately keeps danpawlik's hardening
  (`683ef23b4d`: 802.1AD-aware encap proto recording where the PR hardcodes
  `ETH_P_8021Q`, plus the `hw_outdev` NULL guard) and the MTK-feed `736-11`
  thoff fix, which computes `thoff` from the IP header where the PR simply
  bails. Expect those files to conflict when the PR merges; keep the fork
  versions.
- **firewall4 bridge-offload hardening** (danpawlik devel branch, commit
  `683ef23b4d`, tag `danpawlik-firewall4-bridge-hardening`, follow-up to
  PR 24038) — pulls `kmod-nft-bridge`/`kmod-nft-netdev`/
  `kmod-nf-conntrack-bridge` into firewall4 deps, replaces the per-target hotplug
  scripts with a name-agnostic one shipped by firewall4, guards `hw_ifidx` when
  `hw_outdev` is NULL and records 802.1AD encaps in `675-10`/`675-12`.
- **PPE/WED fixes port** (selective, from danpawlik devel commit `16dd825f5e`,
  tag `danpawlik-ppe-wed-port`, originally for BPI-R4/MT7988) — taken: `736-06…11` (PPE MIB-cache enable fix,
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
  `676-03/05` + `736-12` + mediatek `948`. `676-05`/`652` are the only
  live behavior change (forward-path resolution now also runs for XFRM flows);
  the rest is inline-IPsec groundwork that stays **dormant** — nothing
  registers `mtk_flow_offload_get_cdrt` (needs the feed's EIP-197 inline
  driver, MT7988-class), and `948`'s teardown call compiles out with
  `nf_flow_table=m`. Dan's `676-01/02` (feed `999-crypto-04/05`, xfrm
  packet-mode core changes) were taken 2026-07-25 and **dropped again
  2026-07-26** before ever shipping: they are only exercised by an EIP-197
  inline driver (`NETIF_F_HW_ESP` packet offload) that cannot exist on our
  EIP-97 SoCs, and crypto-04 unconditionally relaxes FWD-direction xfrm
  policy template checks — undesirable on a software-IPsec server (melee/
  nebula run strongswan as IKEv2 client+server). Companion: mediatek
  `402-crypto-inside-secure-avoid-rcu-stall`
  (feed `999-crypto-03`, unmodified) — generic safexcel result-path fix
  (per-request `local_bh` churn + unbounded overflow re-loop) relevant once
  `kmod-crypto-hw-safexcel` accelerates ESP on the EIP-97; the feed's
  `999-crypto-01` (EIP-197 minifw/clock-force, MT7988-class — the fix for the
  AES-GCM drops in upstream issue 21310 on BPi-R4) is **not taken**: its paths
  are inert on EIP-97 and it renames the EIP197 firmware dir for other boards.
- **MT7986 system timer rate** (PR 24943, picked verbatim) —
  `mediatek/patches-6.18/197-…-correct-timer-frequency`, which this fork had
  been carrying as its own copy of the MediaTek feed original since
  2026-08-30. pesa1234 submitted the same fix upstream on 2026-08-29 and it
  is also on its way to the kernel (`Cc: stable`, lore link in the header),
  so on 2026-09-04 the fork's copy was replaced by the PR's. The applied
  hunk is byte-identical — only the patch header changed — so the emitted
  DTB is unchanged; the point is that the fork's commit is now patch-id
  identical to the PR's and auto-drops when it merges.
- **wireless-regdb 2026.09.03** (PR 25008, picked verbatim) — regulatory
  database bump from `2026.05.30`. Upstream regdb changes are 320 MHz for
  Hong Kong, revised 2.4/5 GHz rules for South Africa, and a 60 GHz DFS-flag
  removal for Togo; none of them change behaviour on these two devices, this
  just keeps the database current. `PKG_HASH` re-verified locally against a
  fresh download.
- **Not taken from pesa1234 `next-r4.9.2.rss.mtk`** — 32 of his fork-local
  commits are skipped. The range holds 945 non-merge commits total; the
  other 913 are verbatim vanilla OpenWrt. Reproduce the split with
  `git rev-list --no-merges --count
  pesa1234-r4.9.0-import-base..pesa1234-r4.9.2-import-base` → 945 (tags
  pinned at `next-r4.9.0.rss.mtk` tip `72ab39f45c` and
  `next-r4.9.2.rss.mtk` tip `611fea61ac`; vanilla merge-base
  `78f136f03d`). The tags exist because pesa1234's branch names move:
  `next-r4.9.0.rss.mtk` was force-updated *backwards* mid-port
  (`72ab39f45c` → `d23120ca5e`), and `next-r4.9.2.rss.mtk` is an
  ordinary remote-tracking ref that a routine `git fetch --prune`
  already deleted once during this fork's own review — so running the
  same command against the live branch names, or without `--no-merges`
  (16 merge commits, from pesa1234 merging `openwrt:main` rather than
  rebasing), will not reproduce 32/913. The backwards force-update
  would have added the commit that created his `9999-07`, already
  accounted for here via the `9999-03`/`9999-20` split above.

  The two danpawlik commits cited below are pinned by tag for the same
  reason, and it is no longer hypothetical: `683ef23b4d`
  (`danpawlik-firewall4-bridge-hardening`) and `16dd825f5e`
  (`danpawlik-ppe-wed-port`) were rebased out of his repository and are
  absent from **all 17** of his branches. When this was noticed on
  2026-09-04 they survived only as dangling objects in this clone, one
  `git gc` from being unrecoverable; both are now annotated tags pushed
  to origin. Adding a `danpawlik` remote does **not** substitute for
  this — the commits cannot be re-fetched from anywhere.

  These 32 are deliberately skipped:
  - `advanced_setup` (`96e05f752f`, `43a7ed27e2`, `8431be6fb6`, `fb585731e1`):
    a ~1065-line filogic init.d script that is mostly SMP-affinity/RPS/USB
    power control. Only its ATF portion is wanted; that is re-implemented as
    the ucode `vow_atf` option instead (see the runtime note below).
  - **ksmbd TCP keepalive** (`833a3e55ff`, `8c89b550ce`): skipped here because
    ksmbd is not used on these devices — but moot since 2026-09-01: pesa1234
    upstreamed them himself and vanilla OpenWrt now ships exactly these as
    `backport-6.18/503-01-v7.3-…` / `503-02-v7.3-…` (`065a9b9abc`), so the
    fork carries them as ordinary vanilla files.
  - **His `999-9922` WED WDMA TX hang patch** (`ec602512ab`): we carry
    upstream PR 24784's generic form
    (`generic/backport-6.18/734-...RESV_BUFF...`), which auto-drops on merge.
    Keeping both would collide.
  - **His `9999-07` `init_vif` unwind hunk**: upstream mt76 added the same
    unwind between his pin `59676919` and ours `2c84469c`; we inherit it via
    the `1012` re-anchor in `96c673c41e`. (The rest of his `9999-07` — three
    runtime-guard fixes — is taken separately as our fork-local `9999-20`.)
  - **`9999-17` fix-stranded-frames-in-tx-scheduler** (`bec97cd705`): already
    upstream in mt76 as a Felix Fietkau fix dated 2026-07-22, which our pin
    `2c84469c` (2026-08-18) already carries; applying it is a no-op
    ("Reversed (or previously applied) patch detected").
  - **His 6.12 backport twins**: both targets are `KERNEL_PATCHVER:=6.18`.
  - **ksmbd signing / session teardown** (`ba77feec2a`, `4b3d91053a`,
    `cccfb4cf28`, `331fb5fc7c`, `d05c645d9c`, `4b9eeef65b`, `aa9cb338e9`):
    these landed in vanilla OpenWrt as `backport-6.18/501-v7.1-...` and
    `502-v7.3-...`. `502-v7.3-...` is still carried as a vanilla file;
    `501-v7.1-...` was dropped by the 6.18.49 bump, having reached the stable
    kernel itself.

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
`ls -d /sys/kernel/debug/ieee80211/phy*/mt76/vow_atf` — and a raw debugfs write like
the above is **not** persistent across reboot or Wi-Fi restart. The fork also adds a
sibling `…/mt76/vow` file (patch `9999-07`) for finer VoW tuning (WATF per-level
quantum, per-station DRR quantum, deficit bound, dumps); `vow_atf` is the plain
on/off switch.

Since 2026-08-30 the switch is also settable declaratively, per radio, in
`/etc/config/wireless`:

```
config wifi-device 'radio0'
	option vow_atf '1'   # 1 = on, 0 = off; unset leaves the driver default alone
```

`files-ucode/lib/netifd/wireless/mac80211.sh`'s `setup_phy()` writes this option to
the debugfs file above on every radio bring-up (boot or `wifi reload`), so — unlike
a raw debugfs poke — it survives a restart. Leaving the option unset writes nothing,
matching how every other optional flag in this fork's ucode pipeline behaves.
**Because the underlying knob is device-wide** (it sets `dev->vow_atf_en`, a
`mt7915_dev` field, not a per-phy one) **despite being exposed per-radio in both
the schema and this config syntax**, setting `radio0` and `radio1` to conflicting
values does not average or error — it silently resolves to whichever radio's
`setup_phy()` ran last. Use the same value on every radio of a multi-radio device.
This replaces the ATF handling in pesa1234's `advanced_setup` init.d script
(`advanced.@defaults[0].atf_enable`, commits `96e05f752f`/`43a7ed27e2`/`8431be6fb6`/
`fb585731e1`), which this fork does not carry — that script is ~1065 lines, mostly
SMP-affinity/RPS/USB power control unrelated to WiFi; only its ATF portion was
wanted, and it is now this ucode option instead.

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
