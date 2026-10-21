---
title: 802.11r — Fast BSS Transition (FT)
tags: [80211r, fast-bss-transition, ft, key-hierarchy, roaming, security, multi-band]
sources: [references/80211-2024.pdf]
updated: 2026-10-21
---

# 802.11r — Fast BSS Transition (FT)

*Pre-establishes the key hierarchy once and pre-computes the new PTK before the STA leaves,
so a roam completes in one round trip instead of a full 802.1X/EAP + 4-way handshake.*

> Sources: IEEE Std 802.11-2024, **Clause 13 "Fast BSS transition"** (pp. 3141–3190),
> FT elements in **9.4.2.45–9.4.2.49** (pp. 1112–1117), FT Action frames in **9.6.8** (p. 1721),
> and the FT key hierarchy in Clause 12 (Security).

> Interactive: [FT roam message-sequence walkthrough](assets/ft-roam.html) — step through the
> frames for initial association vs. over-the-air and over-the-DS roams, with a live
> key-hierarchy ledger. Open the HTML file in a browser.

## 1. What problem FT solves

A normal (re)association in an RSN requires a full IEEE 802.1X/EAP exchange plus a
4-way handshake **after** the STA has already moved to the new AP. That adds hundreds of
milliseconds of data-plane outage on every roam — fatal for voice/video.

**Fast BSS Transition** reduces the connectivity gap by:

1. Establishing a **key hierarchy once** at the first association (the *FT initial mobility
   domain association*), so subsequent roams never repeat full 802.1X/EAP.
2. Letting the STA **pre-compute the new PTK before it leaves** the current AP (the FT
   authentication sequence runs over Authentication frames or over the DS), so the
   Reassociation Request/Response pair completes the key install and (optionally) QoS
   resource setup in a single round trip.

FT only applies to roams **between APs in the same mobility domain within the same ESS**
(§13.1). It is the reassociation service — not initial association.

## 2. Core terminology

| Term | Meaning |
|---|---|
| **FTO** (FT Originator) | The non-AP STA performing the transition |
| **Mobility Domain (MD)** | Group of APs that share an FT key hierarchy, named by the **MDID** |
| **MDE** | Mobility Domain element — advertises MDID + FT policy |
| **FTE** | Fast BSS Transition element — carries nonces, MIC, key-holder IDs, wrapped keys |
| **TIE** | Timeout Interval element — reassociation deadline / key lifetime |
| **R0KH / R1KH** | Authenticator key holders (level 0 / level 1), live on the AP-side SME |
| **S0KH / S1KH** | Supplicant key holders (level 0 / level 1), live in the STA |
| **PMK-R0 / PMK-R1 / PTK** | The three-level FT key hierarchy |
| **RIC** | Resource Information Container — bundles resource requests/responses for QoS |

## 3. Key hierarchy (§13.2, Clause 12)

FT distributes the 802.1X Authenticator into a two-level key-holder architecture so that
roaming does not require re-contacting the AS:

```
        MSK / PSK / PMK / FILS-FT
                 │  (derived at first association)
                 ▼
   ┌──────── PMK-R0  ───────┐     held by R0KH (and S0KH in the STA)
   │  R0KH-ID bound in       │     R0KH-ID ≤ 48 octets (fits FTE)
   ▼                         ▼
 PMK-R1  PMK-R1  PMK-R1 ...        one per R1KH; held by R1KH (and S1KH)
   │  R1KH-ID bound in PMK-R1
   ▼
  PTK   ← derived mutually by R1KH and S1KH from PMK-R1, PMKR1Name,
          ANonce (from R1KH), SNonce (from S1KH)
```

- **R0KH** receives the MSK/PSK/PMK/FILS-FT, derives **PMK-R0** and **PMKR0Name**, and
  derives one **PMK-R1 per R1KH** in the mobility domain. It must be colocated with the
  NAS Client / 802.1X Authenticator. R0KH-ID is the NAS-Identifier.
- **R1KH** is identified by a MAC address. It derives **PTK** with the S1KH and
  distributes GTK / IGTK (and BIGTK if beacon protection, WIGTK if WUR protection are on).
- The **R0KH→R1KH channel** that transports PMK-R1 + context is **out of scope** of the
  standard (typically a vendor/controller backplane) — a common interop pain point.
- Key naming binds identity: **R0KH-ID is bound into PMK-R0**, **R1KH-ID into PMK-R1**.
- Lifetimes: PMK-R1 and PTK lifetimes equal the PMK-R0 lifetime (from MSK lifetime, or
  `dot11FTR0KeyLifetime` for PSK). On expiry the holder deletes the SA and the STA must
  redo the FT *initial* mobility domain association.

## 4. Capability advertisement (§13.3)

- The AP advertises FT support by including the **MDE** in **Beacon** and **Probe
  Response** frames: `MDID` + **FT Capability and Policy** field.
- FT Capability and Policy field (1 octet, Fig. 9-435):
  - **B0 — Fast BSS Transition over DS**: 0 = over-the-air only; 1 = over-the-DS allowed.
  - **B1 — Resource Request Protocol Capability**: 1 = AP supports the FT resource request
    protocol (pre-reassociation QoS resource setup).
- The FT policy bits **must be identical across the whole mobility domain**.
- FT capability is also surfaced in the **Neighbor Report** element (BSSID Information →
  Mobility Domain bit), letting an 802.11k-capable STA discover FT-capable neighbors.

## 5. Two protocols × two methods

FT defines **two protocols**, each runnable via **two methods**:

| Protocol | When | QoS resources |
|---|---|---|
| **FT protocol** (§13.5) | Plain transition | Negotiated as part of reassociation (RIC in Reassoc Req/Resp) |
| **FT resource request protocol** (§13.6) | When resources must be confirmed *before* reassociating | Negotiated up front via Auth-Confirm/Auth-Ack (or FT Confirm/Ack) carrying RIC |

| Method | Path |
|---|---|
| **Over-the-Air** | FTO talks **directly to the target AP** using 802.11 Authentication frames with the **FT authentication algorithm** (FTAA) |
| **Over-the-DS** | FTO talks to the target AP **through the current AP**, using **FT Action** frames (FT Request/Response/Confirm/Ack); current AP relays to the target via the RRB (§13.10). Requires MDE bit B0 = 1 |

## 6. The exchanges in detail

### 6.1 FT initial mobility domain association (§13.4) — done once

This is the *first* association in the MD; it builds the key hierarchy. In an RSN
(Fig. 13-2):

```
STA → AP : Authentication-Request  (Open System)
AP → STA : Authentication-Response (Open System)
STA → AP : (Re)Association Request  (MDE, RSNE, RSNXE)
AP → STA : (Re)Association Response (MDE, FTE[R1KH-ID, R0KH-ID], RSNXE)
... 802.1X EAP authentication (skipped for PSK) ...
... FT 4-way handshake (EAPOL-Key) installs PTK/GTK/IGTK, delivers
    TIE[ReassociationDeadline], TIE[KeyLifetime] ...
802.1X Controlled Port opened; FT key hierarchy established.
```

Variants: **non-RSN** (§13.4.3) is just Open auth + (Re)Assoc with MDE, no 4-way
handshake; **over FILS in an RSN** (§13.4.4) establishes the hierarchy inside the FILS
Authentication frames (AKM 00-0F-AC:16/17), avoiding the separate 4-way handshake.

Reject status codes worth knowing: `STATUS_INVALID_MDE` (MDE doesn't match beacon),
`STATUS_INVALID_AKMP` (RSNE doesn't indicate a single FT AKM).

### 6.2 FT protocol, over-the-air in an RSN (§13.5.2, Fig. 13-5)

This is the canonical fast roam:

```
FTO → Target AP : Authentication-Request  (FTAA, 0, RSNE[PMKR0Name], MDE, FTE[SNonce, R0KH-ID])
Target AP → FTO : Authentication-Response (FTAA, Status, RSNE[PMKR0Name], MDE,
                                           FTE[ANonce, SNonce, R1KH-ID, R0KH-ID])
   ── PTK is computed by both sides NOW from PMK-R1, PMKR1Name, ANonce, SNonce ──
FTO → Target AP : Reassociation Request  (RSNE[PMKR1Name], MDE,
                                          FTE[MIC, ANonce, SNonce, R1KH-ID, R0KH-ID],
                                          RIC-Request, RSNXE)
Target AP → FTO : Reassociation Response (RSNE[PMKR1Name], MDE,
                                          FTE[MIC, ANonce, SNonce, R1KH-ID, R0KH-ID],
                                          GTK[N], IGTK[M], BIGTK[Q], RIC-Response, RSNXE)
802.1X Controlled Port unblocked → data flows.
```

- The Auth exchange exchanges nonces and proves liveness; **PTK is ready before the STA
  leaves**. The MIC in the FTE of the Reassociation frames proves possession of the PTK.
- The **R1KH of the target AP** computes PMKR1Name and, if it lacks PMK-R1, pulls it from
  the R0KH (the out-of-scope backplane).
- **Reassociation Deadline Timer**: a successful reassociation only happens if the time
  between Auth-Request and Reassoc-Request does not exceed the reassociation deadline; the
  target AP deletes the PTKSA otherwise.

Over-the-air reject codes: `STATUS_INVALID_PMKID` (bad PMKR0Name),
`R0KH_UNREACHABLE` (target can't reach the R0KH — classic backplane failure),
`STATUS_INVALID_FTE`, `STATUS_INVALID_PAIRWISE_CIPHER`.

### 6.3 FT protocol, over-the-DS in an RSN (§13.5.3, Fig. 13-6)

Identical key math, but the Authentication exchange is carried as **FT Request / FT
Response** Action frames sent **via the current AP** before the STA physically moves.
The current AP relays to the target via the Remote Request Broker (RRB). MLME interface is
`MLME-REMOTE-REQUEST`. Then the STA still does the Reassociation Req/Resp directly with the
target. Forbidden if the target's MDE has B0 (FT-over-DS) = 0.

### 6.4 FT resource request protocol (§13.6, Figs. 13-10..13)

Adds an extra confirm/ack round (Authentication-Confirm / Authentication-Ack over-the-air,
or FT Confirm / FT Ack over-the-DS) **carrying a RIC-Request / RIC-Response** so QoS
resources are committed *before* reassociation. Only available if the target advertises
Resource Request Protocol Capability = 1. The Auth-Ack carries `TIE[ReassociationDeadline]`.

### 6.5 FT reassociation (§13.7)

The final Reassociation Req/Resp. The R1KH verifies the FTE MIC and discards the request if
it is wrong. If an RSNXE is advertised and the FTE's RSNXE-Used bit is 1 but the Reassoc
Request omits the RSNXE, the request is discarded (downgrade protection).

## 7. FT elements (frame body)

### MDE — Mobility Domain element (9.4.2.45, p. 1112)
`Element ID | Length | MDID (2) | FT Capability and Policy (1)`

### FTE — Fast BSS Transition element (9.4.2.46, p. 1112)
`Element ID | Length | MIC Control (2) | MIC (var) | ANonce (32) | SNonce (32) | Optional Parameter(s) (var)`

- **MIC Control** (Fig. 9-437): *RSNXE Used* (1 bit), *MIC Length* (3 bits — for AKM
  00-0F-AC:25: 0→16, 1→24, 2→32 octets), *Element Count* (8 bits, number of elements in
  the MIC). When Element Count = 0 the MIC field is all zeros.
- **ANonce** is chosen by the R1KH; **SNonce** by the S1KH.
- **Optional Parameter(s)** are subelements (Table 9-221):

  | ID | Subelement | Purpose |
  |---|---|---|
  | 1 | R1KH-ID | identifies R1KH for PMK-R1 derivation |
  | 2 | GTK | wrapped group key (Key Info, Key Length, RSC, Wrapped Key) |
  | 3 | R0KH-ID | identifies R0KH |
  | 4 | IGTK | wrapped integrity group key (mgmt frame protection) |
  | 5 | OCI | Operating Channel Info (integrity-protected, anti-channel-spoof) |
  | 6 | BIGTK | beacon protection key |
  | 7 | WIGTK | WUR group key |

### TIE — Timeout Interval element (9.4.2.47, p. 1116)
`Element ID | Length | Timeout Interval Type (1) | Timeout Interval Value (4)`

| Type | Meaning | Units |
|---|---|---|
| 1 | **Reassociation deadline interval** | TUs |
| 2 | **Key lifetime interval** | seconds |
| 3 | Association comeback time | TUs |
| 4 | Time-to-Start | TUs |
| 5 | Peer-to-peer TWT agreement lifetime | TUs |

A reassociation-deadline value of 0 = no deadline; a key-lifetime value of 0 is reserved.

### RDE / RIC Descriptor (9.4.2.48–49, pp. 1116–1117)
A **RIC** is a sequence of Resource Requests or Resource Responses. Each starts with an
**RDE** (`RDE Identifier`, `Resource Descriptor Count`, `Status Code`) followed by elements
describing the resource (e.g., TSPEC). Used to set up QoS reservations atomically with the
roam.

## 8. FT Action frames (9.6.8, p. 1721)

For over-the-DS only: **FT Request**, **FT Response**, **FT Confirm**, **FT Ack**
(distinguished by the FT Action field). Carried between FTO and current AP; current AP
encapsulates to the target AP over the DS (§13.10.3, RRB).

## 9. Multi-band (cross-band) roaming

The operating band is not a parameter of FT: §13.1 scopes it to APs "within the same mobility
domain within the same ESS," and each per-band radio is its own R1KH with its own PMK-R1
(§13.2.2), so a move between bands is an ordinary FT roam. Discovery, BTM steering, OCI, and
cross-band failure modes are covered in
[Cross-Band Roaming with 802.11k/v/r](2026-10-21-cross-band-roaming.md).
