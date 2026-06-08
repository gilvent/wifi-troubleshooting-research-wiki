---
title: Home WiFi Roaming Scenario (2-AP testbed)
tags: [scenario, home-wifi, roaming, openwrt, qca, hostapd, 80211k, 80211v, 80211r, telemetry, nika]
sources: [references/nika-benchmarking-paper.pdf, references/80211-2024.pdf, https://openwrt.org/docs/guide-user/network/wifi/basic, https://w1.fi/hostapd/]
updated: 2026-06-08
---

# Home WiFi Roaming Scenario (2-AP testbed)

*A NIKA-style network scenario `N`: a single-subnet home WLAN with two QCA/OpenWrt access points
sharing one ESS and mobility domain, instrumented end-to-end so 802.11k/v/r roaming faults can be
injected and graded.*

> Sources: NIKA problem-specification method — incident tuple `(N, I, T)`, esp. the `N` component
> (topology + devices + protocols + initial configs, control plane, deployed telemetry),
> *NIKA paper* §3.2 (see [NIKA — Problem Specification Method](nika-problem-specification-summary.md)).
> Protocol/config layer grounded in IEEE Std 802.11-2024 via
> [802.11k](80211k-radio-resource-measurement.md) (Clause 11.10),
> [802.11v](80211v-wireless-network-management.md) (Clause 11.21.7, BTM),
> [802.11r](80211r-fast-bss-transition.md) (Clause 13, FT). Implementation/realization and
> telemetry tooling are standard OpenWrt/`hostapd` practice (openwrt.org, w1.fi/hostapd), named at
> the convention level rather than drawn from the cited specs.

## 0. Where this fits in NIKA

This page defines only the **`N`** of a NIKA incident `(N, I, T)`: the environment a roaming fault
lives in. It is deliberately fault-agnostic so that an issue `I = (Dev, Comp, RC)` and a traffic
workload `T` can be plugged in later (NIKA §3.2). For this scenario the issue coordinates resolve to:

- **`Dev`** ∈ `{ap1, ap2, sta, gw, backhaul}`
- **`Comp`** ∈ `{hostapd-FT/MDID, 11k-neighbor-list, 11v-BTM, steering-daemon, RF/channel, dhcp/dns, backhaul-bridge}`
- **`RC`** — a roaming root cause (the curated set lives in the companion issues page).

## 1. Topology

A flat **single L2 broadcast domain / single IP subnet** (e.g. `192.168.1.0/24`) — the home norm.
One AP is the gateway; the second is a bridged "dumb AP." The client roams between the two cells,
whose coverage overlaps.

```
                 Internet (WAN)
                      │
              ┌───────┴────────┐
              │      ap1        │   QCA + OpenWrt — GATEWAY
              │  router/AP      │   NAT, firewall, DHCP, DNS, NTP
              │  radio: ch 36   │   hostapd (ESS "home", MD "home-md")
              └───┬─────────┬───┘
       wired      │ LAN     │ radio (5 GHz, BSS-A)
     backhaul ────┤ bridge  ╎ ........ overlap ........
       (Ethernet) │         ╎                          ╎
              ┌───┴─────────┴───┐                  ┌────┴─────┐
              │      ap2        │  radio (BSS-B)   │   sta    │  roaming client
              │  dumb AP        │  ch 149          │ (phone/  │  wpa_supplicant
              │  bridged L2     │ ........overlap...│  laptop) │  the device under test
              └─────────────────┘                  └──────────┘
                      │
              ┌───────┴────────┐ (optional)
              │   host_srv     │  wired iperf/ping target for data-plane KPIs
              └────────────────┘
              ┌────────────────┐ (optional)
              │   mon (sniffer)│  3rd radio in monitor mode — 802.11 mgmt capture
              └────────────────┘
```

### Nodes

| Node | Role | Notes |
|---|---|---|
| **ap1** | Gateway + AP (QCA chipset, OpenWrt) | WAN uplink; owns NAT/firewall/DHCP/DNS/NTP; advertises BSS-A |
| **ap2** | Dumb AP (QCA chipset, OpenWrt) | No routing/DHCP; L2-bridged to ap1; advertises BSS-B |
| **sta** | Roaming client | The mobile node; its roam timing/decisions are what the experiment grades |
| **host_srv** | Wired traffic target (optional) | Anchors active probes (iperf/ping/VoIP-like UDP) for the connectivity gap |
| **mon** | Passive 802.11 sniffer (optional) | Captures Beacon/Probe/Auth/(Re)Assoc/Action/Deauth — roam ground truth |

### Links

| Link | Type | Purpose |
|---|---|---|
| WAN → ap1 | Ethernet (uplink) | Internet egress |
| ap1 ⇄ ap2 | **wired Ethernet backhaul**, same bridge | Single broadcast domain; carries DHCP/ARP and the R0KH→R1KH PMK-R1 transport for FT |
| ap1 ⇄ sta / ap2 ⇄ sta | 802.11 (5 GHz) | The two cells the STA roams between; **overlapping coverage**, **different channels** (36 / 149) so both are receivable during the roam |
| ap1 ⇄ host_srv | Ethernet | Wired KPI anchor |

## 2. Per-device configuration

### 2.1 Shared ESS (both APs identical where required)

- **SSID** `home`, identical **security**: WPA2-PSK *or* WPA3-SAE (PSK home case), *or*
  WPA2/3-Enterprise against a RADIUS server (enterprise-at-home / `802.1X` case).
- Same **country/regulatory domain**, same **mode/band** (5 GHz), **different channels** with
  overlapping coverage; tx power and min-basic-rate tuned to size the cells so a roam is actually
  provoked when the STA moves.
- **BSS Max Idle Period** (802.11v keep-alive) set sanely — a too-short idle period produces
  "silent" disassociations that masquerade as roam failures
  ([802.11v §11.21.13](80211v-wireless-network-management.md)).

### 2.2 802.11r — Fast BSS Transition (FT)

Enables single-round-trip roams instead of a full 802.1X/EAP + 4-way handshake on every move
([802.11r §13](80211r-fast-bss-transition.md)). Both APs configured with:

- A single **FT AKM** advertised in the RSNE (FT-PSK `00-0F-AC:4`, FT-SAE `…:9`, or FT-802.1X
  `…:3`) — must be *identical* across the MD or association is rejected (`STATUS_INVALID_AKMP`).
- Same **mobility domain ID (MDID)** advertised in the MDE of Beacons/Probe Responses; FT policy
  bits identical MD-wide (§13.3).
- **R0KH/R1KH key-holder map**: each AP's `nas_identifier`/R0KH-ID and the MAC-keyed R1KH list of
  the *other* AP, so PMK-R1 can be derived/pushed/pulled. The **R0KH→R1KH PMK-R1 transport** rides
  the wired backhaul — this channel is *out of scope* of the standard and a classic interop failure
  point (`R0KH_UNREACHABLE`) (§13.2).
- **FT-over-DS** on/off (MDE bit B0) — chooses whether the FT auth runs through the current AP
  (Action frames + RRB) or over-the-air to the target (§13.5).
- **Reassociation deadline** (TIE) and PMK-R0/R1 lifetimes consistent across APs.

### 2.3 802.11k — Radio Resource Measurement (RRM)

The measurement layer that feeds roam decisions with data instead of blind scans
([802.11k Clause 11.10](80211k-radio-resource-measurement.md)). Both APs:

- `ieee80211k=1` equivalent: advertise **RM Enabled Capabilities** with **Neighbor Report** and
  **Beacon/Link Measurement** bits set.
- **Neighbor Report list**: each AP is given the *other* AP's BSSID + operating class + channel +
  the **Mobility Domain bit** (so the candidate is flagged FT-capable, tying 11k→11r). This lets the
  STA scope its scan to the sibling channel rather than sweeping all channels (§11.10.10).
- Optionally an **AP Channel Report** so the STA prunes its active-scan set to {36, 149}.

### 2.4 802.11v — Wireless Network Management (BTM)

The AP-driven steer ([802.11v §11.21.7](80211v-wireless-network-management.md)):

- `bss_transition=1` (WNM/BTM bit in Extended Capabilities) on both APs.
- A **roam-steering controller** is the *control plane for roaming* in a 2-AP home setup. On OpenWrt
  the de-facto options are **`usteer`** or **`dawn`**: they peer over the LAN, share each client's
  RSSI as seen by *both* APs, and issue **BTM Requests** (with a ranked candidate list /
  Disassociation-Imminent) — and a hard `kick`/deauth as a fallback — to pull a "sticky" client to
  the better AP. RSSI thresholds for *associate / steer / kick* are the main tunables.

### 2.5 Gateway services (ap1 only)

- **DHCP + DNS** (e.g. `dnsmasq`), **NAT + firewall** (`nftables`), **NTP**.
- **Time sync matters twice**: NTP keeps both APs' clocks aligned so per-AP roam logs correlate, and
  the backhaul carries the TSF/neighbor info the steering daemon compares.

### 2.6 STA

- `wpa_supplicant` configured for the ESS with the matching FT AKM; RRM + BTM client support enabled
  so it acts on Neighbor Reports and BTM Requests. The STA's own scan/roam thresholds are part of
  the scenario (driver/`iwd`/`wpa_supplicant` `bgscan`).

## 3. Control plane (bring-up order)

To "bring all nodes up" (NIKA `N` requirement):

1. Backhaul **L2 bridge** up; ap1 gateway services (DHCP/DNS/NAT/NTP) serving the subnet.
2. Both **`hostapd`** instances up with the shared ESS, FT key-holders, 11k neighbor lists, 11v BTM.
3. **Steering daemon** (`usteer`/`dawn`) running on *both* APs and peered over the LAN.
4. **sta** supplicant associates to the ESS (FT *initial* mobility-domain association, building the
   key hierarchy); optional **host_srv** and **mon** brought up for KPIs and capture.

## 4. Telemetry instrumentation

NIKA's `N` explicitly records *which telemetry is deployed*, so an experiment can restrict an agent
to a chosen subset (§3.2). What a home WLAN / roaming experiment commonly establishes:

| Layer | Instrument | What it yields for roaming |
|---|---|---|
| **AP control-interface events** | `hostapd` ctrl iface / `logread`: `AP-STA-CONNECTED/DISCONNECTED`, `BSS-TM-RESP`, `FT-*`, auth/assoc/reassoc, EAPOL | Per-AP record of who joined/left and BTM/FT outcomes |
| **STA supplicant events** | `wpa_supplicant`/`iwd`: `CTRL-EVENT-SCAN-*`, `-CONNECTED/-DISCONNECTED`, roam/`FT` events | The STA's scan→decide→roam timeline |
| **Per-STA radio stats** | `iw dev wlanX station dump` (RSSI, tx/rx rate, inactivity, connected time); `iw … survey dump` (channel busy/noise); scan results | RF view: signal trend that should trigger a roam; airtime/contention |
| **802.11 frame capture** | `mon` radio in monitor mode + `tcpdump`/`tshark` (or `airmon-ng`) | **Ground truth** for roam timing: Beacon/Probe/Auth/(Re)Assoc/Action(BTM, Neighbor Report, FT)/Deauth/Disassoc |
| **Steering-daemon state** | `usteer`/`dawn` `ubus`/status page | Cross-AP per-client RSSI, steer decisions, BTM accept/reject |
| **Data-plane / SLO probes** | Continuous `ping`, `iperf3`, or VoIP-like UDP from sta → gw/`host_srv` | The **connectivity gap**: latency spike, packet loss, throughput dip *during* the roam — distinguishes "roamed but hurt" from "roamed cleanly" |
| **L3 mobility** | gateway DHCP/ARP logs (re-DHCP, gratuitous ARP after roam) | Catches L3-side roam problems (lease churn, ARP/neighbor staleness) |
| **Metrics pipeline** | `collectd`/`node_exporter`/SNMP → **InfluxDB/Prometheus** → **Grafana** | Interface counters, CPU, channel utilization over time — mirrors NIKA's INT→InfluxDB telemetry pattern, enabling dashboards and post-hoc correlation |

The capture layer (`mon` + supplicant/`hostapd` events) is the **roam ground truth**; the data-plane
probes are the **KPI**; the metrics pipeline is the **longitudinal context**. An experiment can hand
an agent any subset (e.g. logs-only, or logs + station dump but no sniffer) to vary difficulty.

## 5. Related pages

- [NIKA — Problem Specification Method](nika-problem-specification-summary.md) — the `(N, I, T)` + `G` framing this scenario instantiates.
- [802.11k](80211k-radio-resource-measurement.md) · [802.11v](80211v-wireless-network-management.md) · [802.11r](80211r-fast-bss-transition.md) — the protocol layer configured above.
