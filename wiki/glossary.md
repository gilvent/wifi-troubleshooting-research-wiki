---
title: Glossary — WiFi Roaming Terms
tags: [glossary, terminology, 80211k, 80211v, 80211r, security]
sources: [references/80211-2024.pdf, https://mrncciew.com/2014/09/02/cwsp-802-11-roaming-basics/, https://mrncciew.com/2020/12/10/fast-roaming-lessons/]
updated: 2026-07-08
---

# Glossary — WiFi Roaming Terms

*Short definitions for terms used across the roaming pages, for linking.*

> Sources: distilled from this wiki's 802.11k/v/r and key-caching pages, grounded in IEEE
> Std 802.11-2024 (Clause 11.10, 11.21, 13) and the mrn-cciew CWSP roaming posts.

Link a term as `glossary.md#term` (lowercase).

### 4-Way Handshake
Derives the PTK/GTK from a PMK. See [Overview](wifi-roaming-overview.md).

### 802.1X
Port-based auth framework carrying EAP. See [Overview](wifi-roaming-overview.md).

### AKM
Authentication and Key Management suite (e.g. FT-PSK `00-0F-AC:4`). See [802.11r](2026-06-08-80211r-fast-bss-transition.md).

### AP Channel Report
Channel list an AP sends so a STA can prune its scan. See [802.11k](2026-06-08-80211k-radio-resource-measurement.md).

### BSS
One AP's radio cell; an ESS is a set of BSSs sharing one SSID.

### BSSID
MAC address of an AP's radio/BSS.

### BSS Max Idle Period
Keep-alive interval before AP disassociates an idle STA. See [802.11v](2026-06-08-80211v-wireless-network-management.md).

### BTM
BSS Transition Management — AP steers/forces a STA to roam. See [802.11v](2026-06-08-80211v-wireless-network-management.md).

### BTM Query / Request / Response
The three BTM frames: ask, steer, accept/reject. See [802.11v](2026-06-08-80211v-wireless-network-management.md).

### CCKM
Cisco proprietary key caching; skips EAP + 4-way handshake. See [Key caching](wifi-roaming-key-caching.md).

### Disassociation Imminent
BTM Request bit signaling forced disassoc is coming. See [802.11v](2026-06-08-80211v-wireless-network-management.md).

### EAP
Extensible Authentication Protocol; mints the PMK. See [Overview](wifi-roaming-overview.md).

### ESS
Set of BSSs sharing one SSID a client roams between.

### FT
Fast BSS Transition (802.11r) — pre-computed PTK, one round-trip roam. See [802.11r](2026-06-08-80211r-fast-bss-transition.md).

### FTE
Fast BSS Transition Element — nonces/MIC/key-holder IDs. See [802.11r](2026-06-08-80211r-fast-bss-transition.md).

### FTO
FT Originator — the roaming STA. See [802.11r](2026-06-08-80211r-fast-bss-transition.md).

### GTK
Group Temporal Key, protects broadcast/multicast traffic.

### IGTK
Integrity group key, protects management frames.

### MD
Mobility Domain — APs sharing an FT key hierarchy. See [802.11r](2026-06-08-80211r-fast-bss-transition.md).

### MDE
Mobility Domain Element — advertises MDID + FT policy. See [802.11r](2026-06-08-80211r-fast-bss-transition.md).

### MDID
Mobility Domain ID — must match across all FT-roam APs. See [802.11r](2026-06-08-80211r-fast-bss-transition.md).

### Neighbor Report
Candidate-AP list a STA uses to scope scanning. See [802.11k](2026-06-08-80211k-radio-resource-measurement.md).

### OKC
Opportunistic Key Caching — cached PMK usable on any AP in the group. See [Key caching](wifi-roaming-key-caching.md).

### PMK
Pairwise Master Key — seeds the 4-way handshake.

### PMK-R0 / PMK-R1
FT key hierarchy below the PMK. See [802.11r](2026-06-08-80211r-fast-bss-transition.md).

### PMKID
ID for a cached PMK, presented to skip EAP. See [Key caching](wifi-roaming-key-caching.md).

### PTK
Pairwise Transient Key — session key for unicast traffic.

### R0KH / R1KH
Authenticator-side FT key holders. See [802.11r](2026-06-08-80211r-fast-bss-transition.md).

### RADIUS
AAA server for 802.1X/EAP — only needed for Enterprise, not PSK/SAE.

### RCPI / RSNI
Standardized signal power / SNR metrics. See [802.11k](2026-06-08-80211k-radio-resource-measurement.md).

### Reassociation
Frame exchange that moves an existing association to a new AP. See [Overview](wifi-roaming-overview.md).

### RIC
Resource Information Container — bundles QoS setup with FT. See [802.11r](2026-06-08-80211r-fast-bss-transition.md).

### RRM
Radio Resource Measurement (802.11k). See [802.11k](2026-06-08-80211k-radio-resource-measurement.md).

### RSNE
RSN Element — carries negotiated AKM/cipher suite.

### RSSI
Received Signal Strength Indicator, drives roam decisions. See [Overview](wifi-roaming-overview.md).

### S0KH / S1KH
Supplicant-side FT key holders (STA side). See [802.11r](2026-06-08-80211r-fast-bss-transition.md).

### SAE
WPA3 password-based key establishment; has its own FT AKM.

### SKC
Sticky Key Caching / PMK Caching — skips EAP only on return to same AP. See [Key caching](wifi-roaming-key-caching.md).

### SSID
Network name; must match across all APs in the ESS.

### STA
Station — a client device.

### TIE
Timeout Interval Element — reassoc deadline / key lifetime. See [802.11r](2026-06-08-80211r-fast-bss-transition.md).

### WNM
Wireless Network Management (802.11v). See [802.11v](2026-06-08-80211v-wireless-network-management.md).

### WPA2 / WPA3
Security certs on RSN; both support Personal (PSK/SAE) and Enterprise (802.1X) modes.
