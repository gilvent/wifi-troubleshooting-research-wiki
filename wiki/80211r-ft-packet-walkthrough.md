---
title: 802.11r FT in Practice — Key Hierarchy, Initial Association & Roams (packet walkthrough)
tags: [80211r, fast-bss-transition, ft, key-hierarchy, over-the-air, over-the-ds, packet-capture, cisco, cwsp]
sources: [https://mrncciew.com/2014/09/05/cwsp-802-11r-key-hierarchy/, https://mrncciew.com/2014/09/06/cwsp-802-11r-ft-association/, https://mrncciew.com/2014/09/07/cwsp-802-11r-over-the-air-ft/, https://mrncciew.com/2014/09/08/cwsp-802-11r-over-the-ds-ft/]
updated: 2026-06-17
---

# 802.11r FT in Practice — Key Hierarchy, Initial Association & Roams (packet walkthrough)

*The capture-level companion to the spec page: the FT key hierarchy on a real
controller/AP split, the elements carried in each frame, and the actual over-the-air vs.
over-the-DS roam exchanges with observed timings.*

> Sources: mrn-cciew CWSP 802.11r series — **Key Hierarchy** (2014-09-05), **FT
> Association** (2014-09-06), **FT over-the-Air** (2014-09-07), **FT over-the-DS**
> (2014-09-08). Lab: Cisco WLC 4402 (7.0.116.0) + LAPs, ACS v5.2 RADIUS. For the
> standards definitions of every element and key below, see
> [802.11r — Fast BSS Transition (spec)](80211r-fast-bss-transition.md).

## 1. Key hierarchy on a controller/AP deployment

The three-level FT hierarchy maps onto the Cisco split as:

| Key | Level | Derived from | Held by (this deployment) |
|---|---|---|---|
| **PMK-R0** | 1st | MSK (from 802.1X/EAP) | WLC — the **R0KH** |
| **PMK-R1** | 2nd | PMK-R0 | AP — the **R1KH** (WLC distributes it) |
| **PTK** | 3rd | PMK-R1 + ANonce + SNonce + AA + SPA | AP ↔ client; encrypts data frames |

- The client side holds the matching **S0KH / S1KH**: it derives PMK-R0 from the MSK, then
  PMK-R1 from PMK-R0, caching both locally.
- `PTK = PRF(PMK-R1, ANonce + SNonce + AA + SPA)` — where **AA** is the authenticator
  (AP) address and **SPA** is the supplicant (client) address. Binding PMK-R1 into the PTK
  is what lets the new AP derive the PTK locally without re-contacting the AS.
- When the WLC also terminates encryption (centrally switched), it can act as **both** R0KH
  and R1KH.

## 2. FT initial mobility domain association (done once)

The first join to the mobility domain builds the hierarchy. It looks like a normal RSN join
plus FT elements:

```
Beacon                 : MDIE(54) + RSNE   — AP advertises FT capability
Open Auth Req/Resp     : 802.11 Open System
Assoc Req              : RSNE(AKM = FT/802.1X) + MDIE
Assoc Resp             : FTIE(55) [MIC, MIC-control, ANonce, SNonce, (R0KH-ID, R1KH-ID)]
802.1X / EAP           : creates the MSK
4-Way Handshake        : derives PTK, installs keys
```

Elements and contents observed:

- **MDIE (Element ID 54):** `MDID` (2 bytes) + FT Capability byte (FT-over-DS bit,
  FT-over-Air bit) + Resource Request capability. The beacon's RSNE lists two AKM suites:
  **00-0F-AC:1** (802.1X/EAP) and **00-0F-AC:3** (FT over 802.1X); the client's Assoc
  Request narrows to FT-over-802.1X only.
- **FTIE (Element ID 55):** MIC, MIC control, ANonce, SNonce, and optional **R0KH-ID /
  R1KH-ID**.
- **4-way handshake** carries the FT context: M2 includes SNonce + RSNIE + MDIE + FTIE;
  M3 (encrypted under the new PTK) includes ANonce + MIC + RSNIE + MDIE + **GTK** + FTIE.
- Endianness gotcha from the capture: WLC config shows MDID `0xac34` while the over-the-air
  capture shows it byte-swapped as `0x34ac`.

## 3. FT roam — over-the-air (4 frames)

The fast roam collapses the 8-frame (auth+reassoc+4-way) sequence into **4 frames**, all
**directly between client and target AP**:

```
1. Auth Request  (client → target) : RSNIE[PMKID list], MDIE, FTIE[SNonce, R0KH-ID]
2. Auth Response (target → client) : RSNIE, MDIE[FT-over-DS bit = 0], FTIE[ANonce, SNonce, R1KH-ID, R0KH-ID]
   ── both sides now have ANonce + SNonce → PTK computed BEFORE reassociation ──
3. Reassoc Req   (client → target) : RSNIE, MDIE, FTIE[MIC, ANonce, SNonce, R1KH-ID, R0KH-ID]
4. Reassoc Resp  (target → client) : RSNIE, MDIE, FTIE, GTK (MIC-protected)
```

- The **FT authentication algorithm** is used in the Auth frames (not Open System).
- The PTK is derived right after frame 2, so frames 3–4 just prove possession (MIC) and
  deliver the GTK — **no separate 4-way handshake**.
- Observed roam time: **~14 ms** (frames 88–95 in the source capture).
- To force over-the-air on the Cisco WLC: `no security ft over-the-ds`.

## 4. FT roam — over-the-DS (via the current AP)

Same key math, but the FT negotiation is carried in **FT Action frames** that the client
exchanges **with the target AP through its current AP**, over the DS — *before* it moves:

```
1. FT Action Request  (wired path) : 802.11 src = client, dst = target BSSID;
                                     current AP encapsulates into CAPWAP Data (UDP 5247)
                                     to the WLC. IEs: RSNIE, MDIE[FT-over-DS bit = 1],
                                     FTIE[SNonce, R0KH-ID]
2. FT Action Response (wired path) : 802.11 src = current AP, dst = client (relayed back);
                                     FTIE[ANonce, SNonce, R1KH-ID, R0KH-ID]
   ── PTK pre-computed while still on the current AP ──
3. Reassoc Request  (over-the-air to target) : FTIE[MIC, SNonce, ANonce, R1KH-ID, R0KH-ID]
4. Reassoc Response (over-the-air from target): status Successful; FTIE[ANonce, SNonce,
                                                R1KH-ID, R0KH-ID, GTK]
```

- The distinguishing trait: the FT exchange (frames 1–2) happens **before** the client
  leaves, with the **current AP relaying** to the target. Only the reassociation (frames
  3–4) is sent on the target's channel.
- The **FT-over-DS bit in the MDIE is set to 1** (vs. 0 for over-the-air).
- In this Cisco deployment the relay rides **CAPWAP Data on UDP 5247** to the WLC — a
  vendor transport detail, not part of 802.11 itself.
- Observed time: **~88 ms** (FT Action Request → Reassociation Response) in the source
  capture — slower than the over-the-air ~14 ms here, dominated by the DS round trip in
  this setup. (Over-the-DS is designed to help when the target's channel is hard to reach
  directly; relative numbers are deployment-dependent.)

## 5. Over-the-air vs. over-the-DS at a glance

| | Over-the-air | Over-the-DS |
|---|---|---|
| FT negotiation path | direct: client ↔ target AP | client ↔ target **via current AP** over the DS |
| Auth carrier | FT Authentication frames | **FT Action** frames |
| MDIE FT-over-DS bit | 0 | 1 |
| When the negotiation happens | as the client arrives at target | **before** the client leaves current AP |
| Observed roam (source lab) | ~14 ms | ~88 ms |

Both finish the actual move with the same **Reassociation Request/Response** carrying the
MIC-bearing FTIE and the GTK.
