---
title: WiFi Roaming — Overview & Fast-Roaming Methods
tags: [roaming, bss-transition, fast-roaming, latency-gap, key-caching, 80211r, 80211k, 80211v, cwsp]
sources: [https://mrncciew.com/2014/09/02/cwsp-802-11-roaming-basics/, https://mrncciew.com/2020/12/10/fast-roaming-lessons/, references/wifi-roaming-analysis.jpg]
updated: 2026-06-17
---

# WiFi Roaming — Overview & Fast-Roaming Methods

*A roam is a client-driven BSS transition between APs in the same ESS. A plain RSN roam
repeats full 802.1X/EAP + 4-way handshake on the new AP (~100+ ms of outage); every
fast-roaming method exists to shrink or remove that gap.*

> Sources: mrn-cciew CWSP **802.11 Roaming Basics** (2014-09-02) and **Fast Roaming
> Lessons** course index (2020-12-10). Module map from `references/wifi-roaming-analysis.jpg`
> ([M4] WiFi Roaming Analysis). Protocol detail anchored to the spec-based pages linked below.

This page is the roaming hub. It frames the problem and the menu of solutions; the
mechanisms each have their own page.

## 1. What roaming is

Roaming (a **BSS transition**) is a client moving its association from one AP to another
within the same ESS. The **decision is purely the client's** — the AP/WLC has no direct
control over *when* a client roams (it can only influence it, e.g. via 802.11k/v hints or
by adjusting RF). The client watches link quality (**RSSI**, **SNR**) and its own
*roaming aggressiveness* setting, and when the serving AP degrades past a threshold it
looks for a better BSSID on the same SSID.

**Association vs. reassociation:** an **Association Request** is used for initial entry to
a network; a **Reassociation Request** carries the transition to a new AP and includes the
prior AP context (and, for key caching, a **PMKID Count / PMKID List**).

## 2. The roaming event, step by step

1. **Scan** for candidate APs — *passive* (listen for Beacons) or *active* (Probe
   Request → Probe Response).
2. **Open System Authentication** with the target (Auth Request / Response).
3. **Reassociation** Request / Response with the target.
4. In an RSN, **re-key**: 802.1X/EAP (if not cached) **then** the **4-way handshake** to
   derive the PTK — *after* the client has already left the old AP.

Step 4 is the expensive part. The EAP exchange alone can dominate the roam.

## 3. The latency gap (why fast roaming exists)

Frame-capture timings from the source lab (Cisco WLC 4402, two 5 GHz LAPs) illustrate the
gap:

| Scenario | Observed roam time | Note |
|---|---|---|
| Initial association | ~116 ms | full join, baseline |
| Plain RSN roam (802.1X) | ~119 ms | of which **~90 ms is EAP** |
| CCKM (Cisco) | ~7 ms | client passes traffic on Reassoc Response |
| 802.11r FT over-the-air | ~14 ms | see [FT packet walkthrough](2026-06-17-80211r-ft-packet-walkthrough.md) |
| 802.11r FT over-the-DS | ~88 ms | observed; via DS relay, see same page |

> These are single-lab numbers (one vendor, fixed channels), not universal constants — use
> them for *relative* intuition. For voice/VoWiFi the practical target is a sub-50 ms gap,
> which is why the EAP step must be eliminated, not merely tuned.

## 4. The menu of fast-roaming methods

| Method | Standardized | What it removes | Page |
|---|---|---|---|
| **Preauthentication** | 802.11-2007 | Pre-runs 802.1X with future APs over the DS | — |
| **PMK Caching (SKC / "sticky")** | 802.11-2007 | Skips 802.1X on *return* to a known AP | [Key caching](2026-06-17-wifi-roaming-key-caching.md) |
| **OKC** (Opportunistic Key Caching) | Vendor (de-facto) | Skips 802.1X across all APs sharing a cached PMK | [Key caching](2026-06-17-wifi-roaming-key-caching.md) |
| **CCKM** (Cisco) | Proprietary (needs CCX) | Skips 802.1X *and* 4-way handshake | [Key caching](2026-06-17-wifi-roaming-key-caching.md) |
| **802.11r FT** | 802.11r-2008 | Pre-computes the PTK before the roam; one round trip | [FT (spec)](2026-06-08-80211r-fast-bss-transition.md) · [FT in practice](2026-06-17-80211r-ft-packet-walkthrough.md) |
| **802.11k** | assist | Neighbor Report → smarter, faster target selection | [802.11k](2026-06-08-80211k-radio-resource-measurement.md) |
| **802.11v** | assist | BTM → network can *steer* the client to a better AP | [802.11v](2026-06-08-80211v-wireless-network-management.md) |

Key caching and CCKM cut the **authentication** cost; 802.11r additionally pre-stages the
**keying**; 802.11k/v don't change the handshake at all — they improve *which* AP the
client picks and *when*.

## 5. Course module map ([M4] WiFi Roaming Analysis)

The source course frames the same material as a module (`wifi-roaming-analysis.jpg`):

- **L4.0** Roaming Overview — *this page*
- **L4.1** Key Caching — [Key caching](2026-06-17-wifi-roaming-key-caching.md)
- **L4.2** Fast BSS Transition — Intro — [802.11r (spec)](2026-06-08-80211r-fast-bss-transition.md)
- **L4.3** FT over-the-Air — [FT packet walkthrough §3](2026-06-17-80211r-ft-packet-walkthrough.md)
- **L4.4** FT over-the-DS — [FT packet walkthrough §4](2026-06-17-80211r-ft-packet-walkthrough.md)
- **L4.5** Roaming with 802.11k — [802.11k](2026-06-08-80211k-radio-resource-measurement.md)
- **L4.6** Roaming with 802.11v — [802.11v](2026-06-08-80211v-wireless-network-management.md)

The blog's own breakdown adds *FT Information Elements*, *FT Compatibility Issues*, *Cisco
Adaptive 11r*, and *WiFi Agile Multiband* as deployment topics — relevant pointers for
future pages but not yet drawn into this wiki.
