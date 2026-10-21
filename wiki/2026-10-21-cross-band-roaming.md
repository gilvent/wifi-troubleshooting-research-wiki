---
title: Cross-Band Roaming with 802.11k/v/r
tags: [multi-band, cross-band, 80211k, 80211v, 80211r, fast-bss-transition, btm, roaming]
sources: [references/80211-2024.pdf]
updated: 2026-10-21
---

# Cross-Band Roaming with 802.11k/v/r

*How a STA fast-roams between bands (5 GHz ↔ 6 GHz ↔ 2.4 GHz): 802.11k finds the other-band
target, 802.11v steers to it, 802.11r executes the move, and none of them treats band as a
parameter.*

> Sources: IEEE Std 802.11-2024 — mobility-domain definition **§3.2**; FT overview **§13.1**
> (p. 3141) and key hierarchy **§13.2.2**; **Neighbor Report element 9.4.2.35** (Fig. 9-416,
> BSSID Information Mobility Domain subfield, p. 1093); **OCI subelement** (Fig. 9-442,
> p. 1115); BSS Transition Management **§11.21.7.3–11.21.7.4** (pp. 2744–2747).

> Interactive: [cross-band FT roam message-sequence walkthrough](assets/ft-roam-multiband.html)
> — step through a 5 GHz → 6 GHz roam over-the-air and over-the-DS, with a per-band key
> ledger and a client-radio band indicator that shows when the radio leaves its serving
> band. Open the HTML file in a browser.

## 1. Band is not a parameter of FT

§13.1 scopes FT to "transitions between APs within the same mobility domain within the same
ESS," and a mobility domain is defined (§3.2) as "a set of basic service sets (BSSs), within
the same extended service set (ESS), that support fast BSS transitions between themselves
and that are identified by the set's mobility domain identifier (MDID)." Neither mentions
frequency. Two BSSes on different bands (two radios of one AP, or two separate APs) are just
two BSSIDs; if both advertise the same MDID, a move between them is an ordinary
[FT roam](2026-06-08-80211r-fast-bss-transition.md).

**The key hierarchy is keyed by holder identity, not band.** The R1KH is identified by a MAC
address, and the R0KH derives a PMK-R1 for each R1KH in the domain (§13.2.2). Each per-band
radio is its own R1KH holding its own PMK-R1, all derived from the single PMK-R0 from the
initial mobility domain association. Roaming to the other-band radio uses that radio's
PMK-R1, with no re-authentication, exactly as for a same-band roam.

## 2. Discovery — the Neighbor Report names the other band (802.11k)

The STA finds the cross-band target through the
[Neighbor Report element](2026-06-08-80211k-radio-resource-measurement.md#neighbor-report-element-format-fig-9-416)
(9.4.2.35, Fig. 9-416, p. 1093):

- **Operating Class** and **Channel Number** tell the STA the candidate is on 6 GHz, for
  example, rather than 5 GHz.
- The **Mobility Domain** subfield of BSSID Information is set to 1 when the candidate "is
  including an MDE in its Beacon frames and that the contents of that MDE are identical to
  the MDE advertised by the AP sending the report" (p. 1093). That tells the STA the
  candidate is reachable by FT, not only by a full 802.1X authentication.
- The colocated-AP bits (*Member Of ESS With 2.4/5 GHz Colocated AP*, *Colocated With 6 GHz
  AP*) are listed on the [802.11k page](2026-06-08-80211k-radio-resource-measurement.md#neighbor-report-element-format-fig-9-416).

## 3. Steering — BTM picks the other-band target (802.11v)

[BTM](2026-06-08-80211v-wireless-network-management.md) chooses *where* the STA goes; §11.21.7 never
mentions band or FT. A cross-band FT roam is an ordinary BTM steer whose chosen target
happens to be on another band.

- **The candidate list is Neighbor Report elements** (§11.21.7.3), so §2 above applies to
  each candidate verbatim.
- **Preference and Abridged steer the band.** A high Preference on the other-band BSSID
  (255 = most preferred) makes the steer. With **Abridged = 1**, every BSSID in the ESS that
  is missing from the list is treated as Preference 0 (§11.21.7.3). Listing only the 6 GHz
  radios therefore excludes the same-band neighbors.
- **Disassociation Imminent and the timer set the deadline.** The Disassociation Timer counts
  down in TBTTs of the *current* BSS, and the AP must not disassociate the STA before it
  reaches 0 (§11.21.7.3). The STA uses that window to run FT against the other-band target.
- **The Response is sent on the serving band.** The BTM Response goes to the associated AP.
  If both Disassociation Imminent and Preferred Candidate List Included are 1 and the STA
  answers **Status 0 (Accept)** with a Target BSSID, it must either disassociate or attempt
  (re)association with a non-excluded candidate (§11.21.7.4). That (re)association can be
  the FT roam of §4.

## 4. Execution — the FT roam (802.11r)

The operations are the standard FT roam, unchanged: FT authentication (PTK pre-computed),
FT reassociation, optional resource request (see
[802.11r §6](2026-06-08-80211r-fast-bss-transition.md#6-the-exchanges-in-detail)). What differs across
bands is only where the client radio is during the negotiation:

- **Over-the-air** — "the FTO communicates directly with the target AP" (§13.1), so the radio
  must already be tuned to the target's other-band channel before sending FT Authentication.
  It is off its serving band for the whole key exchange.
- **Over-the-DS** — FT Action frames go through the current AP (§13.1), so the radio stays on
  the serving band through the key exchange and only tunes to the target for Reassociation.

The [packet walkthrough](2026-06-17-80211r-ft-packet-walkthrough.md#5-over-the-air-vs-over-the-ds-at-a-glance)
compares the two methods frame by frame.

## 5. Channel validation — OCI

Because the target's channel and operating class genuinely differ per band, the **OCI
subelement** (Operating Channel Information, Fig. 9-442, p. 1115) carried in the FTE lets
Operating Channel Validation confirm which band and channel the roam actually landed on and
reject a MITM that misreports it. It authenticates the band; it does not restrict cross-band
roaming.

## 6. Requirements

- Every band's BSS advertises the **same MDID** and identical FT policy bits
  ([802.11r §4](2026-06-08-80211r-fast-bss-transition.md#4-capability-advertisement-133)).
- The out-of-scope **R0KH→R1KH backplane** delivers PMK-R1 to the target-band radio's R1KH.
- The STA is told about the other-band neighbor: 802.11k Neighbor Reports and BTM candidate
  lists include cross-band candidates.

## 7. Where cross-band roams fail

These are inferences from the rules above, not statements the standard makes.

- **BTM Status 2 ("insufficient Beacon or Probe Response frames from all candidates").**
  Typical when the STA has not scanned the target band. The AP *should* then send another BTM
  Request with both Disassociation Imminent and Preferred Candidate List Included set to 0,
  after giving the STA time to scan (§11.21.7.4).
- **Accept followed by a full authentication.** The target BSS does not share the MDE, so the
  Mobility Domain subfield is 0 and FT is not available; the roam falls back to slow 802.1X
  authentication. Check the MDID and FT policy on every band.
- **FT fails with `R0KH_UNREACHABLE`.** The target radio's R1KH could not fetch its PMK-R1
  ([802.11r §6.2](2026-06-08-80211r-fast-bss-transition.md#62-ft-protocol-over-the-air-in-an-rsn-1352-fig-13-5)).
  That is a fault in FT key distribution between key holders, not a BTM fault.
- **An expired candidate list.** After the Validity Interval, the STA must not use the
  candidate list (§11.21.7.3). A slow multi-band scan can outlast a short validity interval.
