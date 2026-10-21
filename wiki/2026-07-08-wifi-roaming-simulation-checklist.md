---
title: WiFi Roaming Simulation — Minimal Setup Checklist
tags: [checklist, setup, testbed, roaming, 80211k, 80211v, 80211r, radius]
sources: [references/80211-2024.pdf, references/nika-benchmarking-paper.pdf, https://openwrt.org/docs/guide-user/network/wifi/basic, https://w1.fi/hostapd/]
updated: 2026-07-08
---

# WiFi Roaming Simulation — Minimal Setup Checklist

*Checklist for a minimal 2-AP roaming testbed.*

> Sources: distilled from this wiki's scenario and protocol pages; terms link to
> [Glossary](2026-07-08-glossary.md).

## Environment

- 2 APs, same ESS
- Wired backhaul between APs (this testbed's way of connecting the two APs so roaming keys can be shared between them — 802.11r doesn't require this specific method)
- 1 client STA with 802.11k/v/r support
- (Optional) monitor-mode radio for packet capture
- (Optional) [RADIUS](2026-07-08-glossary.md#radius) server — only if using WPA2/3-Enterprise

## Setup & Connection

- Matching [SSID](2026-07-08-glossary.md#ssid) on both APs
- Same subnet / L2 domain (a simplification for this simple testbed; real networks can roam across different subnets, but that's more complex to set up)
- Different channels, overlapping coverage (lab convenience for a clean cell boundary — not an 802.11k/v/r requirement; same-channel roaming works too)
- Same country/regulatory domain and band
- Tx power tuned so a roam is actually triggered (lowering power forces a roam in a small test space — WiFi roaming itself doesn't require this)
- [BSS Max Idle Period](2026-07-08-glossary.md#bss-max-idle-period) set sanely

## Security & Encryption

- Same [WPA2/WPA3](2026-07-08-glossary.md#wpa2--wpa3) version on both APs
- Same [AKM](2026-07-08-glossary.md#akm) (PSK, SAE, or Enterprise)
- If Enterprise: matching [RADIUS](2026-07-08-glossary.md#radius) config + EAP method on client

## 802.11r (FT)

- [FT](2026-07-08-glossary.md#ft) enabled on both APs
- Same FT [AKM](2026-07-08-glossary.md#akm)
- Same [MDID](2026-07-08-glossary.md#mdid)
- Same FT policy bits (over-the-air vs. over-the-DS)
- [R0KH/R1KH](2026-07-08-glossary.md#r0kh--r1kh) key-holder map configured
- Consistent key lifetimes / reassociation deadline ([TIE](2026-07-08-glossary.md#tie))

## 802.11k (RRM)

- [RRM](2026-07-08-glossary.md#rrm) enabled on both APs
- [Neighbor Report](2026-07-08-glossary.md#neighbor-report) on each AP lists the other AP
- (Optional) [AP Channel Report](2026-07-08-glossary.md#ap-channel-report) configured

## 802.11v (BTM)

- [BTM](2026-07-08-glossary.md#btm) enabled on both APs
- Steering daemon running and peered between APs (e.g. `usteer`/`dawn`, or a WLC) — a piece of software that decides when to push a client to roam; 802.11v defines the messages it sends, not this software itself
- RSSI thresholds tuned (associate / steer / kick) — settings you pick for testing, not part of the 802.11v standard

## Client (STA)

- Supplicant configured with matching FT AKM
- RRM support enabled
- BTM support enabled
- Roam aggressiveness / scan thresholds set

## Verification

- AP-side event log
- Client-side event log
- Per-station radio stats (RSSI trend)
- Packet capture (monitor mode)
- Continuous ping/iperf probe running through the roam

## Sanity Checks

- AKM + MDID match across client and both APs
- Backhaul carries PMK-R1 (no fallback to full EAP)
- Neighbor Report / BTM candidate list contains the other AP
- Observed roam time matches fast-roam range, not plain-roam range (see [Overview §3](2026-06-17-wifi-roaming-overview.md))

## Related pages

- [Glossary](2026-07-08-glossary.md)
- [Home WiFi Roaming Scenario (2-AP testbed)](2026-06-08-home-wifi-roaming-scenario.md)
- [WiFi Roaming — Overview & Fast-Roaming Methods](2026-06-17-wifi-roaming-overview.md)
- [802.11r](2026-06-08-80211r-fast-bss-transition.md) · [802.11k](2026-06-08-80211k-radio-resource-measurement.md) · [802.11v](2026-06-08-80211v-wireless-network-management.md)
- [Fast Roaming via Key Caching](2026-06-17-wifi-roaming-key-caching.md)
- [Common WiFi Roaming Issues](2026-06-08-wifi-roaming-common-issues.md)
