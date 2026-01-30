# MT7921 TX Power Reporting Fix

This patch fixes incorrect txpower reporting in the Linux kernel mt76 driver for MT7921-based WiFi adapters (including MT7921AU USB adapters like Alfa AWUS036AXML).

## The Problem

The mt7921 driver never updates `phy->txpower_cur`, causing `mt76_get_txpower()` to report incorrect values via nl80211. Users see bogus txpower readings (typically 3 dBm or 67 dBm) when querying with `iw`:

```
$ iw dev wlan0 info | grep txpower
        txpower 3.00 dBm    # Wrong!
```

## The Fix

The patch updates `txpower_cur` in `mt7921_bss_info_changed()` when `BSS_CHANGED_TXPOWER` is set, using the channel's regulatory power limit.

After applying:
```
$ iw dev wlan0 info | grep txpower
        txpower 33.00 dBm   # Correct (matches regulatory limit)
```

## Test Results

Tested on Alfa AWUS036AXML (MT7921AU), kernel 6.18.6:

| Band | Expected | Actual | Status |
|------|----------|--------|--------|
| 2.4GHz ch1 | ~33 dBm | 33.00 dBm | PASS |
| 5GHz ch100 | ~27 dBm | 27.00 dBm | PASS |
| 6GHz ch5 | ~15 dBm | 15.00 dBm | PASS |

## Status

Submitted to linux-wireless mailing list as [PATCH v2] on 2026-01-30.

## Files

- `0001-wifi-mt76-mt7921-fix-txpower-reporting.patch` - The kernel patch

## Applying the Patch

```bash
cd /path/to/linux/drivers/net/wireless/mediatek/mt76
patch -p1 < 0001-wifi-mt76-mt7921-fix-txpower-reporting.patch
```

## License

This patch is submitted under the same license as the Linux kernel (GPL-2.0).
