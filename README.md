# MT7921 TX Power Reporting Fix

Kernel patch fixing incorrect txpower reporting in the Linux mt76 driver for MT7921-based WiFi adapters (MT7921U, MT7922, and related chipsets).

## The Problem

The mt7921 driver never updates `phy->txpower_cur` from the rate power configuration sent to firmware. `mt76_get_txpower()` reports bogus values via nl80211 -- typically 3 dBm regardless of actual regulatory or SAR limits:

```
$ iw dev wlan0 info | grep txpower
        txpower 3.00 dBm    # Wrong
```

This affects all MT7921/MT7922 devices on all bands. hcxdumptool and other tools that read txpower may crash or behave incorrectly.

## Root Cause

Two issues compound:

1. The rate power loop in `mt76_connac_mcu_rate_txpower_band()` computes the correct bounded TX power per channel but **discards the return value**. The equivalent mt7915 code stores it to `phy->txpower_cur` -- the mt7921 connac path does not.

2. mt7921 uses the chanctx model but its `add_chanctx` callback does not update `phy->chandef`, leaving it stale after association. The rate power loop's channel comparison then fails silently.

## The Fix (v4)

Two changes in one patch:

- **Capture the rate power loop result** in `mt76_connac_mcu_rate_txpower_band()` and store to `txpower_cur` when processing the current channel. Subtract the multi-chain path delta before storing, since `mt76_get_txpower()` adds it back when reporting -- matching how mt7915 handles this via `mt76_get_power_bound()`.

- **Sync `phy->chandef` from chanctx** in `add_chanctx` and `change_chanctx`, with a lightweight `mt7921_update_txpower_cur()` helper that recomputes the bounded power for the current channel without reprogramming firmware rate tables.

After applying:
```
$ iw dev wlan0 info | grep txpower
        txpower 36.00 dBm   # Correct (matches regulatory limit)
```

## Test Results

Tested on Alfa AWUS036AXML (MT7921AU), kernel 6.19.8, Canada (ISED):

| Band | Regulatory Limit | v4 Result | Stock | Status |
|------|-----------------|-----------|-------|--------|
| 2.4 GHz | 36 dBm | 36 dBm | 3 dBm | PASS |
| 5 GHz | 23 dBm | 23 dBm | 3 dBm | PASS |
| 6 GHz | 12 dBm | 12 dBm | 3 dBm | PASS |

Independent testing by sam8641 (MT7921U USB + MT7922 PCIe, US regulatory, Debian 6.12.74):

| Device | Band | Result | Status |
|--------|------|--------|--------|
| MT7921U USB | 2.4/5 GHz | 30 dBm | PASS |
| MT7922 PCIe | 2.4/5/6 GHz | 30/30/18 dBm | PASS |

## Version History

| Version | Date | Approach | Outcome |
|---------|------|----------|---------|
| v1 | 2026-01-25 | Store `hw->conf.power_level` in `set_rate_txpower` | Rejected: wrong source (Felix Fietkau) |
| v2 | 2026-01-30 | Read `bss_conf.txpower` in `bss_info_changed` | Rejected: should come from rate power loop (Sean Wang) |
| v3 | 2026-03-17 | Rate loop store + chanctx calls + BSS_CHANGED_TXPOWER | Rejected: chanctx calls too heavy, BSS_CHANGED breaks multi-vif (Sean Wang) |
| v4 | 2026-03-19 | Rate loop store + lightweight helper + path delta fix | **Submitted, awaiting review** |

## Upstream Status

- **Mailing list:** Submitted to linux-wireless 2026-03-19
- **Patchwork:** [Link](https://patchwork.kernel.org/project/linux-wireless/patch/20260130215458.52886-1-lucid_duck@justthetip.ca/)
- **Lore:** [Search](https://lore.kernel.org/linux-wireless/?q=lucid_duck%40justthetip.ca)
- **Community thread:** [morrownr/USB-WiFi#700](https://github.com/morrownr/USB-WiFi/issues/700)

## Files

- `0001-wifi-mt76-mt7921-fix-txpower-reporting-from-rate-pow.patch` -- The v4 kernel patch (2 files changed, 41 insertions, 3 deletions)

## Applying the Patch

```bash
cd /path/to/linux
git apply 0001-wifi-mt76-mt7921-fix-txpower-reporting-from-rate-pow.patch
```

Or to build just the mt76 modules:
```bash
make -C /lib/modules/$(uname -r)/build M=drivers/net/wireless/mediatek/mt76 modules
```

## Stable Backport Note

v4 depends on commit `56e3867` which renamed `mt76_tx_power_nss_delta()` to `mt76_tx_power_path_delta()`. For kernels before ~6.18, replace `mt76_tx_power_path_delta` with `mt76_tx_power_nss_delta` -- they return identical values for 1-4 chain devices.

## Known Limitations (follow-up work)

- `iw set txpower fixed` not reflected in managed/station mode (BSS_CHANGED_TXPOWER dropped per reviewer feedback -- needs a multi-vif-safe approach). Works in AP mode via `IEEE80211_CONF_CHANGE_POWER`.
- MT7925 needs analogous changes in its own `main.c` (the connac common code fix benefits it, but without the chandef sync the channel comparison never matches).

## License

GPL-2.0 (same as the Linux kernel).
