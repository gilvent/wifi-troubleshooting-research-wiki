---
title: Fast Roaming via Key Caching (PMK Caching, OKC, CCKM)
tags: [roaming, fast-roaming, key-caching, pmk-caching, okc, cckm, pmkid, 8021x, cwsp]
sources: [https://mrncciew.com/2014/09/02/cwsp-802-11-roaming-basics/, https://mrncciew.com/2020/12/10/fast-roaming-lessons/]
updated: 2026-06-17
---

# Fast Roaming via Key Caching (PMK Caching, OKC, CCKM)

*Key caching speeds roaming by reusing a PMK from an earlier 802.1X/EAP exchange, so a roam
can skip the (slow) EAP step. The variants differ in how widely the cached PMK can be
reused, and whether the 4-way handshake is also avoided.*

> Sources: mrn-cciew CWSP **802.11 Roaming Basics** (2014-09-02) and **Fast Roaming
> Lessons** course index (2020-12-10). Standardized methods are from 802.11-2007; OKC and
> CCKM are vendor mechanisms. See [WiFi roaming overview](wifi-roaming-overview.md) for how
> these compare to 802.11r FT.

## 1. The idea

A full RSN roam re-runs **802.1X/EAP** (to mint a fresh MSK→PMK) and then the **4-way
handshake** (to derive the PTK) on the new AP. EAP is the dominant cost (~90 ms in the
source capture). If the client and AP already share a valid **PMK**, EAP can be skipped and
the roam jumps straight to the 4-way handshake.

The client signals a cached PMK by including a **PMKID Count** and **PMKID List** in the
**Reassociation Request** (carried in the RSNE). If the AP recognizes a listed PMKID, it
proceeds without EAP.

## 2. PMK Caching (SKC / "sticky")

- **Standard:** 802.11-2007.
- After the first full 802.1X auth to an AP, both sides **cache the PMK** (keyed by PMKID).
- On *return* to that **same AP**, the client presents the PMKID; the AP skips EAP and runs
  only the 4-way handshake.
- Limitation: the PMK is valid **only on the AP that minted it**. Roaming to a *new* AP
  still triggers full EAP. Hence "sticky" — it only helps when you come back.

## 3. OKC — Opportunistic Key Caching

- **Vendor mechanism** (widely implemented, not in the base standard).
- The infrastructure (WLC, or APs sharing context) **pre-distributes / derives the cached
  PMK for every AP** in the mobility group, so the client's single 802.1X PMK is usable on
  *any* AP it roams to.
- The client computes the PMKID it expects for the target AP and presents it in the
  Reassociation Request; the AP already holds the matching PMK → **EAP skipped on first
  roam to a previously-unvisited AP**.
- Still performs the **4-way handshake** on each roam (PTK is per-AP).

## 4. CCKM — Cisco Centralized Key Management

- **Cisco proprietary**; requires CCX-capable clients.
- A central key manager (WLC) caches credentials so a roam skips **both** the EAP exchange
  **and** the 4-way handshake — the client "continues to pass traffic as soon as the AP
  sends the Reassociation Response."
- Fastest of the caching family in the source lab: **~7 ms** roam vs ~119 ms plain.
- Cost: vendor lock-in (Cisco + CCX). 802.11r FT is the standards-based way to get a
  comparable single-round-trip roam across vendors.

## 5. Where this sits vs. 802.11r

Key caching removes the **authentication** cost but still does a handshake per roam (except
CCKM). 802.11r **FT** goes further: it pre-establishes a key hierarchy once and
**pre-computes the PTK before the client leaves**, so the roam completes in one round trip
without a separate 4-way handshake — and does so as an open multi-vendor standard. See
[802.11r (spec)](80211r-fast-bss-transition.md) and the
[FT packet walkthrough](80211r-ft-packet-walkthrough.md).

| | EAP skipped | 4-way handshake skipped | PMK reuse scope | Standard |
|---|---|---|---|---|
| **PMK caching (SKC)** | on return only | no | same AP | 802.11-2007 |
| **OKC** | yes | no | all APs in group | vendor |
| **CCKM** | yes | yes | all APs (Cisco) | Cisco/CCX |
| **802.11r FT** | yes | yes (FT replaces it) | mobility domain | 802.11r-2008 |
