# Network Troubleshooting Wiki — Index

Registry of all wiki pages, grouped by category. Updated whenever a page is added or renamed.

## Research direction

- [Cost-Aware Diagnostic Agents for Spec-Governed Edge Systems](edge-diagnostic-agent-research-direction.md) — the charter: WiFi-roaming diagnosis framed as cost-aware sequential evidence acquisition under measurement interference; the contribution is the sensing/probe-selection policy, evaluated on an unanswerable-query-rate vs. inflicted-harm frontier.

## Roaming fundamentals

- [WiFi Roaming — Overview & Fast-Roaming Methods](wifi-roaming-overview.md) — the roaming hub: what a roam is, the latency gap, and the menu of fast-roaming methods (k/v/r, key caching, CCKM).
- [Fast Roaming via Key Caching](wifi-roaming-key-caching.md) — PMK caching (SKC), OKC, and CCKM — reusing a PMK to skip the slow EAP step on roam.
- [Glossary — WiFi Roaming Terms](glossary.md) — one-line definitions for terms used across the roaming pages (AKM, MDID, R0KH/R1KH, RRM, BTM, etc.).

## 802.11 amendments (roaming protocols)

- [802.11k — Radio Resource Measurement](80211k-radio-resource-measurement.md) — measurement layer; Neighbor Report and Link Measurement feed roam decisions.
- [802.11v — Wireless Network Management](80211v-wireless-network-management.md) — BSS Transition Management (BTM) steers a STA to a better AP with a ranked candidate list.
- [802.11r — Fast BSS Transition](80211r-fast-bss-transition.md) — pre-computes keys so the actual roam completes in one round trip.
- [802.11r FT in Practice — Packet Walkthrough](80211r-ft-packet-walkthrough.md) — capture-level companion to the 802.11r spec page: key hierarchy on a controller/AP split and the over-the-air vs. over-the-DS roam exchanges with observed timings.

## Benchmarking & methodology

- [NIKA — Problem Specification Method](nika-problem-specification-summary.md) — how NIKA frames a reproducible network-troubleshooting problem: incident (N, I, T) + goal (G).

## Telemetry & data collection

- [WiFi Troubleshooting Data Collection — Industry Mechanisms](2026-07-14-wifi-data-collection-mechanisms.md) — survey of the collection mechanisms (infrastructure streaming, scanning radios, triggered capture, synthetic testing, client agents, 802.11k, and the independent passive sniffer node) across commercial AIOps vendors and open standards, mapped to the edge-diagnostic-agent research direction.

## Scenarios & issue catalogs

- [Home WiFi Roaming Scenario (2-AP testbed)](home-wifi-roaming-scenario.md) — NIKA-style `N`: two QCA/OpenWrt APs, one ESS + mobility domain, with topology, per-device 802.11k/v/r config, and telemetry instrumentation.
- [Common WiFi Roaming Issues (curated issue catalog)](2026-06-08-wifi-roaming-common-issues.md) — 10 documented roaming faults as (Category, Root Cause, Key Signals), the candidate `I` faults for the scenario above.
- [WiFi Roaming Simulation — Minimal Setup Checklist](wifi-roaming-simulation-checklist.md) — checklist form of the scenario above: environment, SSID/security, 802.11k/v/r config, client, verification.
