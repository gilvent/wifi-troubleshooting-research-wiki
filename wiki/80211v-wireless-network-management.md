---
title: 802.11v — Wireless Network Management (WNM)
tags: [80211v, wnm, bss-transition-management, btm, roaming]
sources: [references/80211-2024.pdf]
updated: 2026-06-08
---

# 802.11v — Wireless Network Management (WNM)

*Network-management services; for roaming the centerpiece is BSS Transition Management (BTM)
— the AP-driven mechanism to suggest or force a STA to roam, with a ranked candidate list.*

> Sources: IEEE Std 802.11-2024. Procedures in **Clause 11.21 "Wireless network management
> procedures"** (pp. 2656–2776), with **BSS Transition Management in 11.21.7** (pp. 2744–2747).
> WNM Action frames: **9.6.13** (p. 1741). BTM frames: **9.6.13.8–10** (pp. 1745–1749).
> WNM elements: **9.4.2.59–9.4.2.89** (pp. 1136–1198). WNM Sleep Mode: **11.2.3.15** (p. 2492).

## 1. Scope

802.11v is a grab-bag of **network-management services** that let the infrastructure and
STAs exchange operational info and steer behavior. For fast roaming the centerpiece is
**BSS Transition Management (BTM)** — the AP-driven mechanism to *suggest or force a STA to
roam to a better AP*, optionally with a ranked candidate list. WNM is enabled per
`dot11WirelessManagementImplemented`; a non-AP HE STA must support BTM.

## 2. WNM service catalog (Clause 11.21)

| Service | Subclause | What it does |
|---|---|---|
| **BSS Transition Management (BTM)** | 11.21.7 | Steer/force STA roaming with a candidate list — **the roaming service** |
| Event request/report | 11.21.2 | STA reports transition, RSNA, peer-link, WNM-log, BSS-color events |
| Diagnostic request/report | 11.21.3 | Config profile, manufacturer info, association & 802.1X-auth diagnostics |
| Location track | 11.21.4 | Location configuration request/response, location track notification |
| Timing measurement / FTM | 11.21.5–6 | Fine timing measurement (ranging) |
| FMS (Flexible Multicast Service) | 11.21.8 | Alternate multicast delivery intervals/rates |
| Colocated interference reporting | 11.21.9 | Report interference from a colocated radio |
| TFS (Traffic Filtering Service) | 11.21.12 | AP filters frames for a sleeping STA |
| **BSS Max Idle Period** | 11.21.13 | Keep-alive interval before AP disassociates an idle STA |
| Proxy ARP | 11.21.14 | AP answers ARP/ND on behalf of STA (lets STA sleep) |
| Channel usage | 11.21.15 | Channel info for off-network / candidate selection |
| DMS (Directed Multicast Service) | 11.21.16 | Deliver group-addressed frames as unicast |
| WNM notification | 11.21.17 | Generic vendor notification request/response |
| **WNM Sleep Mode** | 11.2.3.15 | Extended sleep beyond the listen interval, skipping group-key updates |
| TIM Broadcast | 11.2.3.14 | Deliver TIM info outside the normal beacon to save power |

## 3. BSS Transition Management (§11.21.7) — the roaming service

### 3.1 Capability (§11.21.7.1)
- Advertised via the **BSS Transition** bit in the **Extended Capabilities** element.
- `dot11BSSTransitionActivated = true` ⇒ STA shall support BTM. A non-AP HE STA must have it.
- Enables load balancing and "your session is ending" alerts. **Does not apply in an IBSS.**

### 3.2 Three frames (WNM Action category)
| Frame | Sender | Role |
|---|---|---|
| **BTM Query** (9.6.13.8) | non-AP STA | "What are my transition candidates?" — may include the STA's own preferred candidate list |
| **BTM Request** (9.6.13.9) | AP | Solicited (reply to Query) or **unsolicited/broadcast** steer; carries Request Mode + candidate list |
| **BTM Response** (9.6.13.10) | non-AP STA | Accept (with Target BSSID) or Reject (with status code) |

### 3.3 BTM Query frame (Fig. 9-1274)
`Category | WNM Action | Dialog Token | BSS Transition Query Reason | BSS Transition Candidate List Entries (optional)`
- *Query Reason* codes (Table 9-238); reason **19 = "Preferred BSS Transition Candidate List
  Included"** when the STA is volunteering its own preferences.
- Candidate list entries are **Neighbor Report elements** (9.4.2.35) collected via the STA's
  scanning — this is where 802.11k and 802.11v meet.

### 3.4 BTM Request frame (Fig. 9-1275) — the steer
`Category | WNM Action | Dialog Token | Request Mode (1) | Disassociation Timer (2) | Validity Interval (1) | BSS Termination Duration (0/12, opt) | Session Information URL (var, opt) | BSS Transition Candidate List Entries (var, opt)`

**Request Mode field (Fig. 9-1276)** — the key control bits:

| Bit | Field | Meaning |
|---|---|---|
| B0 | **Preferred Candidate List Included** | 1 ⇒ the list is a *preferred* list the STA should use; 0 ⇒ STA may ignore the list |
| B1 | **Abridged** | 1 ⇒ any BSSID *not* in the list is treated as preference 0 (excluded) |
| B2 | **Disassociation Imminent** | 1 ⇒ STA *will* be disassociated from this AP (see Disassociation Timer) |
| B3 | **BSS Termination Included** | 1 ⇒ this BSS is shutting down; BSS Termination Duration present |
| B4 | **ESS Disassociation Imminent** | 1 ⇒ STA will be disassociated from the whole ESS; Session Information URL present |

- **Disassociation Timer**: number of TBTTs until the AP sends a Disassociation frame
  (counts down each Beacon). While Disassoc Imminent = 1 the AP won't disassociate until the
  timer hits 0 — the STA's window to roam gracefully.
- **Validity Interval**: TBTTs for which the candidate list / preference values stay valid.
- **BSS Termination Duration**: BSS Termination TSF + minutes the BSS will be absent
  (same subelement as Neighbor Report ID 4).
- **Session Information URL**: shown to the user when leaving the ESS (RFC 3986).
- **Candidate List Entries**: zero or more Neighbor Report elements, each carrying a **BSS
  Transition Candidate Preference** subelement (Table 9-213): **255 = most preferred, 1 =
  least, 0 = excluded**.

### 3.5 BTM Response frame (Fig. 9-1279)
`Category | WNM Action | Dialog Token | BTM Status Code (1) | BSS Termination Delay (1) | Target BSSID (0/6, opt) | BSS Transition Candidate List Entries (var, opt)`

**BTM Status Code (Table 9-539)** — how the STA answered:

| Code | Meaning |
|---|---|
| 0 | **Accept** — will transition; Target BSSID present |
| 1 | Reject — unspecified |
| 2 | Reject — insufficient Beacon/Probe Response from all candidates |
| 3 | Reject — insufficient available capacity from all candidates |
| 4 | Reject — BSS termination undesired |
| 5 | Reject — BSS termination delay requested (see BSS Termination Delay) |
| 6 | Reject — STA BSS Transition Candidate List provided (STA counter-proposes) |
| 7 | Reject — no suitable BSS transition candidates |
| 8 | Reject — leaving ESS |

### 3.6 Behavioral rules that bite in the field
- AP **must** respond to an individually addressed Query with a Request (if it supports BTM).
- Unsolicited/broadcast Requests only to STAs that advertised BTM support.
- If **Preferred Candidate List Included = 0**, the STA *may ignore* the candidate list.
- If **Abridged = 1**, every in-ESS BSSID absent from the list is treated as preference 0
  (excluded) — a strong steer.
- On **Accept**, if the STA sent Status 0 it shall either disassociate from the current AP
  or attempt (re)association with one of the **non-excluded** candidates.
- Before transitioning to an **excluded** BSS, the STA must first send a BTM Response giving
  the reject reason.
- The AP must not send a Disassociation frame while Disassoc Imminent = 1 until the timer
  reaches 0.

### 3.7 Planned ESS (§11.21.7.5) — the ESS Report element (9.4.2.255)
Lets the AP tell STAs the ESS is **planned with overlapping BSSs**, whether the AP is at the
**edge of the ESS**, and a **Recommended BSS Transition RSSI Threshold Within ESS** (beacon
RSSI below which a STA should start roaming). STAs may use this to tune *when* they scan/roam.

## 4. How 802.11v ties into k and r (roaming pipeline)

```
802.11k  ──►  802.11v  ──►  802.11r
Neighbor       BTM Request    FT key hierarchy +
Report /        ranks the     pre-computed PTK →
candidate       candidates    fast reassociation
discovery       (Disassoc     to the chosen AP
(scan-pruning)  Imminent,     (no full 802.1X)
                preference)
```

The **Neighbor Report element** is the shared data structure: produced under 802.11k, used
verbatim as the BTM "candidate list," and the chosen target is reached fast via 802.11r.

## 5. Selected supporting WNM services

- **WNM Sleep Mode (11.2.3.15, element 9.4.2.80)**: STA sleeps for many beacon intervals,
  optionally **without receiving GTK/IGTK updates**, and resyncs keys on wake. Reduces power
  but interacts with key rotation — a roaming/key-state troubleshooting area.
- **BSS Max Idle Period (11.21.13, element 9.4.2.77)**: AP tells the STA the **keep-alive
  interval** before it gets disassociated for inactivity. Mis-set values cause "silent"
  disassociations that look like roam failures.
- **TFS / DMS / FMS / Proxy ARP**: offload/filtering services that let STAs sleep longer;
  relevant when sleep + roaming interact.
- **Directed Multicast / FMS multicast rate (11.21.8/16)**: control multicast delivery,
  rate, and reliability.
