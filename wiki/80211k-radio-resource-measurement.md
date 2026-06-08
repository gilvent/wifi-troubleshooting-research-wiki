---
title: 802.11k — Radio Resource Measurement (RRM)
tags: [80211k, rrm, neighbor-report, link-measurement, roaming]
sources: [references/80211-2024.pdf]
updated: 2026-06-08
---

# 802.11k — Radio Resource Measurement (RRM)

*The measurement layer: STAs and APs measure the RF environment and share results
(Neighbor Report, Link Measurement) so roaming decisions use data instead of blind scanning.*

> Sources: IEEE Std 802.11-2024. Procedures in **Clause 11.10 "Radio measurement
> procedures"** (pp. 2589–2621). Key elements: **RM Enabled Capabilities** (9.4.2.43,
> p. 1107), **Measurement Request / Report** (9.4.2.19 / 9.4.2.20, pp. 954 / 989),
> **Neighbor Report** (9.4.2.35, p. 1093), **AP Channel Report** (9.4.2.34, p. 1092).
> Action frames: **9.6.6 Radio Measurement Action frame details** (p. 1653).

## 1. Purpose

802.11k gives STAs and APs a standard way to **measure the RF environment and share the
results**, so roaming/load decisions are made from data instead of blind scanning. The two
mechanisms most relevant to fast roaming are the **Neighbor Report** (discover roam
candidates without a full off-channel scan) and **Link Measurement** (quantify the current
link). RRM is the measurement layer that 802.11v (BTM) and 802.11r (FT) build on.

## 2. Capability negotiation — RM Enabled Capabilities element (9.4.2.43)

`Element ID | Length | RM Enabled Capabilities (5 octets)`

A 40-bit string where each bit advertises one capability (Table 9-218). The roaming-relevant
bits:

| Bit | Capability |
|---|---|
| 0 | **Link Measurement** |
| 1 | **Neighbor Report** |
| 2 | Parallel Measurements |
| 3 | Repeated Measurements |
| 4–6 | Beacon **Passive / Active / Table** Measurement |
| 7 | Beacon Measurement Reporting Conditions |
| 8 | Frame Measurement |
| 9 | Channel Load Measurement |
| 10 | Noise Histogram Measurement |
| 11 | Statistics Measurement |
| 12–13 | LCI / LCI Azimuth |
| 14–15 | Transmit Stream/Category (+ Triggered) Measurement |
| 16 | **AP Channel Report** |
| 17 | RM MIB |
| 18–20 | Operating Channel Max Measurement Duration |
| 21–23 | Nonoperating Channel Max Measurement Duration |
| 24–26 | Measurement Pilot |
| 27 | Measurement Pilot Transmission Information |
| 28 | Neighbor Report TSF Offset |
| 29–30 | RCPI / RSNI Measurement |

`dot11RadioMeasurementActivated = true` is the master switch; it forces
`dot11OperatingClassesRequired` etc. to true.

## 3. Measurement request/report mechanics (§11.10.1–11.10.9)

- A STA can measure itself, or **request other STAs in the same BSS to measure on its
  behalf** (§11.10.6). Requests/reports ride in **Radio Measurement Action** frames
  (9.6.6) carrying one or more **Measurement Request / Measurement Report** elements.
- **Measurement start time & randomization** (§11.10.3): each request may carry a
  randomization interval; the STA picks a uniform random delay in `[0, randomization]` TUs
  before measuring — avoids synchronized "measurement storms." Value 0 = no delay.
- **Measurement duration** (§11.10.4): governed by `dot11RMMaxMeasurementDuration` and the
  request's *Duration Mandatory* bit:

  ```
  Maximum Measurement Duration (TUs) = 2^(dot11RMMaxMeasurementDuration − 4) × BeaconInterval
  ```

  | MaxMeasDuration | Duration Mandatory | Behavior |
  |---|---|---|
  | 0 | 1 | Measure for requested duration |
  | 0 | 0 | May measure for shorter |
  | nonzero | 0 | Measure for min(requested, max) |
  | nonzero | 1, requested > max | **Reject** (Report Mode = "refused") |
  | nonzero | 1, requested ≤ max | Measure for requested duration |

- **Station responsibility** (§11.10.5): a STA must decode every request and assess impact;
  it **may refuse** (Refused bit = 1) for QoS/power/scheduling reasons. It cancels all
  in-process measurements on Disassociation or on (re)association to a different BSSID.
- **Operating vs nonoperating channels** (§11.10.2): off-channel measurements may interrupt
  data service; the STA chooses spacing. Nonoperating-channel duration is bounded by
  `dot11RMNonOperatingChannelMaxMeasurementDuration` (0 ⇒ STA rejects off-channel requests
  with Incapable = 1).

### Common measurement report types (§11.10.9)
Beacon report, Frame report, Channel load report, Noise histogram report, STA statistics
report, **LCI (location) report**, Location Civic report, Location Identifier report,
Transmit Stream/Category report, **FTM Range report**.

## 4. Neighbor Report — the roaming workhorse (§11.10.10, element 9.4.2.35)

A STA asks its associated AP for a list of neighbor APs so it can **scope/optimize its
scan** instead of sweeping all channels.

### Flow
- STA → AP: **Neighbor Report Request** frame (9.6.6.6) — may include an SSID element
  (wildcard = all neighbors; specific = that ESS).
- AP → STA: **Neighbor Report Response** frame (9.6.6.7) containing zero or more **Neighbor
  Report elements**, one per neighbor AP. Requires `dot11RMNeighborReportActivated = true`,
  else the AP ignores the request.
- Contents are advisory and may be stale; live Beacon/Probe info takes precedence
  (§11.10.10.1 NOTE).

### Neighbor Report element format (Fig. 9-416)
`Element ID | Length | BSSID (6) | BSSID Information (4) | Operating Class (1) | Channel Number (1) | PHY Type (1) | Optional Subelements (var)`

**BSSID Information field (Fig. 9-417)** — tells the STA what the candidate supports:

| Bits | Field | Notes |
|---|---|---|
| B0–B1 | **AP Reachability** | 0 reserved, 1 Not Reachable, 2 Unknown, 3 Reachable (for preauth) |
| B2 | Security | same security provisioning as current AP |
| B3 | Key Scope | same Authenticator as reporting AP |
| B4–B9 | Capabilities | Spectrum Mgmt, QoS, APSD, **Radio Measurement**, … |
| B10 | **Mobility Domain** | candidate advertises the same MDE ⇒ **FT-capable** (ties 802.11k → 802.11r) |
| B11–B14 | HT / VHT / FTM / HE | PHY/feature support |
| B15 | ER BSS | |
| B16 | Colocated AP | same physical device as another reported AP |
| B17 | Unsolicited Probe Responses Active | |
| B18 | Member Of ESS With 2.4/5 GHz Colocated AP | 6 GHz discovery hint |
| B19 | OCT Supported With Reporting AP | |
| B20 | Colocated With 6 GHz AP | |
| B22 | DMG Positioning | |

### Roaming-relevant subelements (Table 9-212)

| ID | Subelement | Why it matters for roaming |
|---|---|---|
| 1 | **TSF Information** | TSF Offset + Beacon Interval → STA can time-align to the target (faster reassoc); only sent if AP guarantees ≤ 1.5 TU TSF accuracy |
| 3 | **BSS Transition Candidate Preference** | network's preference value (1–255, 255 = most preferred; 0 = excluded) — shared with 802.11v BTM (Table 9-213) |
| 4 | **BSS Termination Duration** | candidate BSS will shut down at a TSF time for N minutes |
| 5 | Bearing | direction/distance/relative height to neighbor |
| 6 | Wide Bandwidth Channel | |
| 39 | Measurement Report (LCI / Location Civic) | neighbor location |
| 45/61/62 | HT Capabilities / HT Operation / Secondary Channel Offset | |
| 66 | Measurement Pilot Transmission | |
| 70 | RM Enabled Capabilities | neighbor's RRM support |
| 71 | Multiple BSSID | |
| 191–198 | VHT/HE Capabilities/Operation, (HE) BSS Load, SSID, HE 6 GHz | candidate PHY + load |

The same Neighbor Report element is reused as the **BSS Transition Candidate List Entry**
in 802.11v BTM frames — so 802.11k and 802.11v share the candidate representation.

## 5. AP Channel Report element (9.4.2.34, §11.10.18)

`Element ID | Length | Operating Class (1) | Channel List (var)`

A compact list of channels (within one operating class) where the STA is **likely to find
an AP** of this ESS. Lets a STA prune its active-scan channel set. Multiple elements appear
when more than one operating class is reported.

## 6. Link Measurement (§11.10.11)

- STA → peer: **Link Measurement Request** frame (9.6.6.4).
- Peer → STA: **Link Measurement Report** frame (9.6.6.5) containing a **TPC Report**
  element (transmit power used + link margin), antenna ID, and signal quality as **RCPI**
  (received channel power indicator) and **RSNI** (received signal-to-noise indicator).

Gives an instantaneous, bidirectional view of link quality — useful to decide *when* a roam
is warranted. Requires `dot11RMLinkMeasurementActivated`.

## 7. Measurement Pilot frames (§11.10.15)

Compact Action frames an AP transmits **pseudo-periodically at a sub-beacon interval** to
let STAs (a) discover a BSS, (b) gather neighbor signal strength via passive scanning, and
(c) trigger Probe Requests — all faster than waiting for full Beacons.

## 8. RCPI / RSNI (9.4.2.36 / 9.4.2.39)

- **RCPI** — total received channel power (per dBm scale), the standardized "RSSI."
- **RSNI** — (signal+noise+interference) to (noise+interference) ratio.
These are the standardized metrics carried in Beacon/Link/Neighbor reports so roaming logic
is comparable across vendors.
