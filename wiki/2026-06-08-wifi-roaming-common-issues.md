---
title: Common WiFi Roaming Issues (curated issue catalog)
tags: [roaming, issues, sticky-client, 80211r, 80211k, 80211v, rf-design, l3-mobility, deep-research]
sources: [references/80211-2024.pdf, https://www.juniper.net/documentation/us/en/software/mist/mist-wireless/topics/topic-map/rssi-fast-roaming.html, https://www.cisco.com/c/en/us/support/docs/wireless/catalyst-9800-series-wireless-controllers/221671-understand-802-11r-11k-11v-fast-roams-on.html, https://www.cisco.com/c/en/us/td/docs/wireless/controller/8-10/config-guide/b_cg810/client_roaming.html, https://www.cisco.com/c/en/us/td/docs/wireless/controller/8-5/Enterprise-Mobility-8-5-Design-Guide/Enterprise_Mobility_8-5_Deployment_Guide/wlanrf.html, https://www.cisco.com/c/en/us/td/docs/wireless/controller/9800/17-9/config-guide/b_wl_17_9_cg/m_rx_sop_ewlc.html, https://www.cisco.com/c/dam/en/us/td/docs/wireless/controller/technotes/80211r-ft/b-80211r-dg.html, https://higherlogicdownload.s3.amazonaws.com/HPE/MigratedAttachments/9B71B25A-80A1-4851-8B5F-21FDC9AB083F-2-VRD_Optimizing-WLAN-for-Roaming-Devices.pdf, https://github.com/openwrt/openwrt/commit/3cc56a5534b8b49a7e9ba57edf9878ec32bdd27a, https://documentation.meraki.com/Architectures_and_Best_Practices/Cisco_Meraki_Best_Practice_Design/Best_Practice_Design_-_MR_Wireless/Wireless_Layer_3_Roaming_Best_Practices, https://mrncciew.com/2014/09/11/cwsp-802-11k-ap-assisted-roaming/]
updated: 2026-06-08
---

# Common WiFi Roaming Issues (curated issue catalog)

*Ten well-documented WiFi-roaming faults, framed as `(Category, Root Cause, Key Signals)` like the
NIKA "Network Issues" taxonomy — the candidate `I = (Dev, Comp, RC)` faults to inject into the
[Home WiFi Roaming Scenario](home-wifi-roaming-scenario.md).*

> Sources: curated via `/deep-research` (5 search angles, 21 sources fetched, 25 falsifiable claims
> 3-vote adversarially verified → 24 confirmed, 1 refuted/excluded). Primary vendor docs (Cisco,
> Aruba/HPE, Juniper Mist, Meraki), OpenWrt/`hostapd`, and the CWNP-grade mrn-cciew write-ups; the
> protocol layer is anchored in IEEE Std 802.11-2024 via this wiki's
> [802.11k](80211k-radio-resource-measurement.md) / [802.11v](80211v-wireless-network-management.md) /
> [802.11r](80211r-fast-bss-transition.md) pages. Per-row source tags resolve in the
> [Sources legend](#sources-legend). Each root cause and its signals carry the supporting source(s)
> inline.

## How to read this

Each row is one injectable issue. **Key Signals** are the observable symptoms — log lines, frame
exchanges, RSSI/metric thresholds — an operator or agent would see, not the fix. The
`(Dev, Comp)` column maps the issue onto the scenario's device/component coordinates (see
[scenario §0](home-wifi-roaming-scenario.md#0-where-this-fits-in-nika)).

## The catalog

### Client-side roaming behavior

| # | Root Cause | Key Signals | (Dev, Comp) | Source |
|---|---|---|---|---|
| 1 | **Sticky client.** The roam decision is client-owned in 802.11; conservative client firmware (high RSSI thresholds, no standardized roam logic) does not initiate a roam to a better target AP, so it clings to its current/far AP as signal weakens — the single most common roaming problem. | Client stays associated to a far AP while a neighbor offers stronger signal; current-AP RSSI falls below the device's roam trigger (≈ −70 dBm typical, *device-dependent* — macOS ≈ −75 dBm) yet no roam; throughput degradation + high retries; operator sees the client riding a far AP at ≈ −80 dBm. | `sta`, client-roam-logic | [Juniper-Mist], [Cisco-221671], [Aruba-VRD] |

### 802.11k / 802.11v steering faults

| # | Root Cause | Key Signals | (Dev, Comp) | Source |
|---|---|---|---|---|
| 2 | **Missing / unhonored 802.11k Neighbor Report.** Without a Neighbor Report (AP not returning one, or client not requesting/acting on it), the client scans blindly across all channels instead of being pre-aware of candidate target APs — slow, late roams. | No Neighbor Report Request/Response action-frame exchange in capture; full passive/active channel scan before each roam (long scan dwell) instead of a targeted scan; roam latency dominated by scan time. | `ap1/ap2`, 11k-neighbor-list | [mrn-11k], [Cisco-221671] |
| 3 | **Over-aggressive forced disassociation (Optimized Roaming / BTM disassoc-imminent mis-tuned).** Controller-side sticky-client mitigation kicks the client off the far AP; if the CHDM/RSSI threshold or disassoc-imminent timer is too aggressive it drops still-usable clients or causes ping-pong. | Disassociation **Reason 4 — Timeout** sent when data RSSI < CHDM threshold (default ≈ −80 dBm); re-association denied until RSSI rises **6 dB** above the disassoc threshold (hysteresis, e.g. −80 → rejoin at −74 dBm); 802.11v **BTM Request** issued before the forced disconnect (C9800/IOS-XE); 11v-capable client force-disconnected when it doesn't transition before the disassociate-imminent timer expires. | `ap1/ap2`, 11v-BTM / steering-daemon | [Cisco-221671], [Cisco-8.10], [Cisco-8.5-RF] |

### 802.11r / FT key & mobility-domain faults

| # | Root Cause | Key Signals | (Dev, Comp) | Source |
|---|---|---|---|---|
| 4 | **Cached-key miss at the target AP (no PMKID / PMK-R1 / OKC entry; or all fast-roam methods disabled).** The fast-roam shortcut breaks, forcing a **full EAP re-authentication + fresh 4-way handshake** on every roam. | C9800 RA-trace: `Failed to Dot11 validate dot11i pmkids. No matching pmkid for the pmk available in cache`; full EAP exchange (EAP-Identity, EAP method, RADIUS Access-Challenge/Accept) then complete 4-way handshake (M1–M4); roam latency in the **hundreds of ms** (voice/video impact); target AP can't acknowledge the FT/reassoc because it doesn't hold the client's PMKID. Working FT instead shows `Dot11 validate dot11r pmkid. 11r PMKID match found` → `Fast roam = True`. | `ap2`(target), FT-PMK-R1 / OKC cache | [Juniper-Mist], [Cisco-221671], [Cisco-11r-DG] |
| 5 | **Misconfigured key-holder lists / inconsistent mobility-domain ID.** Missing or mismatched `r0kh`/`r1kh` entries, or a different MDID across APs, breaks the FT key hierarchy so the STA reauthenticates instead of fast-roaming. (In OpenWrt/`hostapd` FT-PSK, the documented fix is `ft_psk_generate_local=1` + a consistent `mobility_domain` derived from the SSID's md5sum rather than the old hardcoded `4f57`; **does not apply to FT-EAP**, which still needs R0KH→R1KH push/pull.) | FT fails when `r0kh`/`r1kh` entries are missing/mismatched; clients reauthenticate (lose fast roam) when MDIDs differ across APs; FT begins working after local PSK key generation + a shared md5sum-derived mobility domain. (Backplane-unreachable analog in the standard: `R0KH_UNREACHABLE`, see [802.11r §6.2](80211r-fast-bss-transition.md#62-ft-protocol-over-the-air-in-an-rsn-1352-fig-13-5).) | `ap1/ap2`+`backhaul`, hostapd-FT R0KH/R1KH + MDID | [OpenWrt-FT], [Cisco-11r-DG], [Aruba-VRD] |
| 6 | **Reassociation timeout set below the ~20 s default.** On an L2 roam the next AP cancels the cached PMK-R1 if the client doesn't reassociate within the window, so FT clients fall back to full reauthentication. | 802.11r clients reauthenticate (no fast roam) after an L2 roam when the reassociation timeout < 20 s; full reauth observed at a controller boundary on L2 roam. | `ap2`(target), FT reassoc-timeout | [Cisco-11r-DG] |

### Client compatibility / fast-roam enablement

| # | Root Cause | Key Signals | (Dev, Comp) | Source |
|---|---|---|---|---|
| 7 | **Legacy/IoT client incompatibility with 802.11r.** Enabling FT can break clients that can't parse FT AKMs — older 802.11n devices, handheld scanners, printers — and FT is only offered on WPA2/OPEN SSIDs. | Older 11n devices / scanners / printers fail to associate after 802.11r is enabled on the WLAN; FT setup options greyed out unless the SSID is WPA2 or OPEN. *(Version caveat: Cisco "adaptive 11r," Release 8.0+, lets non-FT clients join an FT WLAN; the dual-SSID — separate non-FT SSID — workaround is still the documented mitigation.)* | `sta`, FT-AKM negotiation | [Aruba-VRD], [Cisco-11r-DG] |

### RF / coverage design

| # | Root Cause | Key Signals | (Dev, Comp) | Source |
|---|---|---|---|---|
| 8 | **Insufficient cell overlap / oversized cells / weak cell edge.** Poor RF design means there is no good target to roam to, so a client holds a far AP after entering another cell. RX-SOP tuning is the documented remediation (raising it shrinks the receiver cell). | Client holds an AP well after moving into a neighbor's coverage; survey/heatmap overlap below **10–15 %** (data) / **15–20 %** (voice); cell-edge RSSI worse than **−67 dBm**, or SNR < 25 dB / noise floor above −92 dBm; client distribution improves after raising RX-SOP in high-density areas. | `ap1/ap2`, RF/channel/coverage | [Cisco-8.5-RF], [Cisco-RXSOP] |
| 9 | **Low basic/legacy data rates left enabled.** With 1/2/6 Mbps basic rates active, cell-edge clients sustain a slow link instead of roaming, worsening sticky behavior and degrading the whole BSS. | Edge clients connected at 1 Mbps or 6 Mbps, degrading AP airtime; clients fail to roam until their rate drops, then roam once the minimum basic rate is raised (Aruba: client initiates roaming once its rate drops below 18 Mbps when 6/9/12 are cut); improved roam initiation after trimming lower rates from the SSID profile. | `ap1/ap2`, basic-rate-set | [Aruba-VRD] |

### L2 / L3 mobility & backhaul / infrastructure

| # | Root Cause | Key Signals | (Dev, Comp) | Source |
|---|---|---|---|---|
| 10 | **Distributed L3-roaming inter-AP backhaul break.** In distributed Layer-3 roaming the post-roam path depends on inter-AP IP reachability to the anchor AP; if the target AP can't reach the anchor, the L3 roam times out and the client is forced to re-DHCP on the new VLAN. (Meraki: APs health-check the anchor every second over **UDP 9358**.) | L3 roam timeout when the target AP can't reach the anchor AP; forced **DHCP renew on the new VLAN** after roam (IP change / connectivity gap); loss of UDP 9358 reachability / failed 1-second anchor ping; packet-loss spike + session drop coinciding with the roam. | `backhaul`+`ap2`, L3-mobility-tunnel | [Meraki-L3] |

## Sources legend

| Tag | Source | Quality |
|---|---|---|
| [Juniper-Mist] | Juniper Mist — *RSSI and Fast Roaming* | primary (vendor) |
| [Cisco-221671] | Cisco — *Understand 802.11r/11k/11v Fast Roams on Catalyst 9800* (doc 221671) | primary (vendor) |
| [Cisco-8.10] | Cisco — *AireOS 8.10 Client Roaming config guide* (Optimized Roaming) | primary (vendor) |
| [Cisco-8.5-RF] | Cisco — *Enterprise Mobility 8.5 Design Guide*, WLAN RF chapter | primary (vendor) |
| [Cisco-RXSOP] | Cisco — *Catalyst 9800 17.9 config guide*, RX-SOP | primary (vendor) |
| [Cisco-11r-DG] | Cisco — *802.11r FT Deployment Guide* | primary (vendor) |
| [Aruba-VRD] | Aruba/HPE — *VRD: Optimizing WLAN for Roaming Devices* | primary (vendor) |
| [OpenWrt-FT] | OpenWrt commit `3cc56a5` — FT-PSK `ft_psk_generate_local` + md5sum mobility domain | primary (project) |
| [Meraki-L3] | Cisco Meraki — *Wireless Layer 3 Roaming Best Practices* | primary (vendor) |
| [mrn-11k] | mrn-cciew — *CWSP: 802.11k AP-assisted roaming* | blog (CWNP-grade, corroborated by Cisco docs) |

## Caveats & gaps

- **Device-dependent thresholds.** The −70 dBm roam trigger is typical, not universal (macOS ≈ −75 dBm); this was the one finding with a split (2-1) verification vote.
- **Vendor/version drift.** Optimized Roaming differs between AireOS (Reason-4 disassoc) and Catalyst 9800/IOS-XE (adds an 802.11v BTM Request first); legacy-client FT incompatibility (#7) is mitigated by adaptive 11r in Release 8.0+; Aruba ClientMatch sticky-client SNR default changed across ArubaOS versions.
- **Single-source rows.** #9 (data-rate trimming) rests on the Aruba VRD; #10 (Meraki UDP 9358 / re-DHCP) on the Meraki best-practices doc — both high-confidence vendor behavioral claims but vendor-specific.
- **Not found / excluded.** No source-grounded claim surfaced for a *pure* authentication/RADIUS root cause (e.g. unreachable RADIUS or CoA latency causing roam delay) beyond the EAP-reauth/PMKID-miss findings (#4); among deauth/disassoc reason codes only **Reason 4 — Timeout** was verifiable as a roaming signal (no source confirmed a "code 15 / 4-way-handshake-timeout" roaming mapping, so it is intentionally omitted). One refuted claim — a specific over-the-air vs over-the-DS FT *preference* rule — was excluded (0-3 vote).

## Related pages

- [Home WiFi Roaming Scenario](home-wifi-roaming-scenario.md) — the `N` these issues inject into.
- [802.11k](80211k-radio-resource-measurement.md) · [802.11v](80211v-wireless-network-management.md) · [802.11r](80211r-fast-bss-transition.md) — the standard mechanisms behind issues #2–#7.
- [NIKA — Problem Specification Method](nika-problem-specification-summary.md) — the `(Dev, Comp, RC)` issue formalism this catalog follows.
