# Wiki Log

Append-only record of operations. Newest at the bottom. One entry per operation.

Format:

```
## [YYYY-MM-DD] OP | <question or task>
- Pages: <created/updated page paths>
- Sources: <references used, with section/page>
- Notes: <optional — decisions, open questions, flags>
```

`OP` is one of: `QUERY`, `LINT`, `EDIT`.

---

## [2026-06-08] EDIT | Bootstrap wiki: three-layer structure + seed pages
- Pages: created `index.md`, `log.md`; moved `80211k-…`, `80211r-…`, `80211v-…` into `wiki/` and added frontmatter
- Sources: references/80211-2024.pdf
- Notes: Seed pages were generated earlier from a section-targeted read of IEEE Std 802.11-2024 (k=Clause 11.10, v=Clause 11.21, r=Clause 13). Practical/unsourced "troubleshooting checklist" sections were removed before bootstrap.

## [2026-06-08] EDIT | Conform NIKA page into the wiki
- Pages: moved `nika-problem-specification-summary.md` into `wiki/` with frontmatter; linked from `index.md` under "Benchmarking & methodology"
- Sources: references/nika-benchmarking-paper.pdf §3.2

## [2026-06-08] EDIT | Consolidate ambiguous schema
- Pages: rewrote `CLAUDE.md`; updated `index.md` heading; standardized `> Source:` → `> Sources:` across all pages and gave the NIKA page a `> Sources:` line
- Sources: n/a (schema only)
- Notes: Scope fixed to "network troubleshooting, WiFi-roaming focus." Provenance rule pinned to two places — frontmatter `sources` (files) + `> Sources:`/inline citations (locations). Added page-naming (kebab, topic-first) and `updated`-bump rules.

## [2026-06-08] EDIT | Add Research operation + home for /deep-research
- Pages: updated `CLAUDE.md` (Research op, `wiki/research/` location, URLs allowed in `sources`, `RESEARCH` log op); added "Research" category to `index.md`; created `wiki/research/`
- Sources: n/a (schema only)
- Notes: Reports go to `wiki/research/<YYYY-MM-DD>-<topic>.md` as one report+synthesis page; Lint allows external URLs in `sources` for research pages only.

## [2026-06-08] EDIT | Flatten wiki: drop research/ subfolder
- Pages: `CLAUDE.md` (removed Research op + `wiki/research/`; URLs now allowed on any page; reverted log OP set to QUERY/LINT/EDIT; added `wiki/assets/` for charts); removed empty "Research" category from `index.md`; deleted `wiki/research/`
- Sources: n/a (schema only)
- Notes: Synthesis pages (comparisons, deep-research reports) are normal flat `wiki/` pages, typed by index category + tags rather than by folder. Subfolders deferred until a category is unwieldy (~10+ pages).

## [2026-06-08] EDIT | Git: ignore source binaries, track names via manifest
- Pages: added `.gitignore` (ignores `references/*` except README, plus OS junk), `references/README.md` (source manifest); `CLAUDE.md` references/Lint rules updated to point at the manifest
- Sources: n/a (schema only)
- Notes: 80211-2024.pdf is licensed + 52 MB, so source binaries stay untracked; the manifest records their names/provenance. Lint now checks `sources` against `references/README.md`.

## [2026-06-08] QUERY | Define a NIKA-style home WiFi roaming scenario (2 APs)
- Pages: created `home-wifi-roaming-scenario.md`; added "Scenarios & issue catalogs" category to `index.md`
- Sources: references/nika-benchmarking-paper.pdf §3.2 (the `N` component of `(N, I, T)`); references/80211-2024.pdf via the k/v/r wiki pages (Clause 11.10 / 11.21.7 / 13); OpenWrt + hostapd (openwrt.org, w1.fi/hostapd) for the realization/telemetry layer
- Notes: Scenario = single L2 subnet, ap1 (gateway, QCA/OpenWrt) + ap2 (bridged dumb AP), roaming STA, optional wired target + monitor sniffer. Documents topology, shared-ESS + FT key-holder/MDID + 11k neighbor-list + 11v BTM (usteer/dawn) config, bring-up order, and a telemetry table (hostapd/wpa_supplicant events, station/survey dumps, mon-mode capture, data-plane probes, collectd/Influx/Grafana). Implementation/telemetry layer flagged as conventional, not drawn from the cited specs.

## [2026-06-08] QUERY | Curate common WiFi roaming issues (deep-research, 10 issues)
- Pages: created `2026-06-08-wifi-roaming-common-issues.md`; linked from `index.md`
- Sources: Juniper Mist, Cisco (221671, AireOS 8.10 client roaming, Enterprise Mobility 8.5 RF, C9800 RX-SOP, 802.11r FT DG), Aruba/HPE VRD, OpenWrt commit 3cc56a5, Cisco Meraki L3 Roaming, mrn-cciew 802.11k; protocol layer anchored to references/80211-2024.pdf via the k/v/r pages
- Notes: 10 issues in (Category, Root Cause, Key Signals) form across 6 categories (client-side, 11k/v steering, FT key/MDID, client compat, RF design, L2/L3 backhaul), mapped to scenario `(Dev, Comp)`. Built via /deep-research: 5 angles, 21 sources, 25 claims 3-vote adversarially verified (24 confirmed, 1 refuted/excluded). Honest gaps recorded: no pure RADIUS-unreachable root cause found; only deauth Reason 4 verified (no "code 15" mapping → omitted); −70 dBm threshold device-dependent; Meraki/Aruba rows single-source.

## [2026-07-01] LINT | Validate wiki health
- Pages: fixed `index.md` (registered 3 orphan pages); `references/README.md` (added mrn-cciew 2014-09-11 11k post to the series enumeration)
- Sources: n/a (health check)
- Notes: 3 orphans found — `wifi-roaming-overview.md`, `wifi-roaming-key-caching.md`, `80211r-ft-packet-walkthrough.md` (added 2026-06-17 in commit 9d8421b) were never linked from `index.md`. Fixed: added a "Roaming fundamentals" category (overview hub + key caching) and filed the FT packet walkthrough under "802.11 amendments" beside the 802.11r spec page. All frontmatter complete, all H1/summary/`> Sources:` lines present, all internal links + anchors resolve, cross-page timings consistent (no contradictions). Flag only (not auto-fixed): the 3 pages added on 2026-06-17 have no corresponding QUERY/EDIT entry in this log — not backfilled to avoid fabricating history.

## [2026-07-08] QUERY | Checklist of configs/setups for a simple WiFi roaming simulation
- Pages: created `wifi-roaming-simulation-checklist.md`, `glossary.md` (linked from `index.md`); updated `home-wifi-roaming-scenario.md`
- Sources: distilled from existing wiki pages (`home-wifi-roaming-scenario.md`, `wifi-roaming-overview.md`, `80211k/v/r` pages, `wifi-roaming-key-caching.md`), themselves grounded in references/80211-2024.pdf, references/nika-benchmarking-paper.pdf, openwrt.org, w1.fi/hostapd, mrn-cciew posts
- Notes: no new source material — a checklist-form distillation of prior QUERY answers, per user request for terse checklist output (no spec-clause citations, no long explanations). Glossary created alongside so checklist terms (AKM, MDID, R0KH/R1KH, RRM, BTM, RADIUS, etc.) link to one-line definitions instead of re-explaining inline. Follow-up question ("why different channels, can't we roam same-channel?") caught an imprecise claim in `home-wifi-roaming-scenario.md`'s topology table (implied different channels were needed "so both are receivable during the roam") — checked against the 80211k/v/r pages (no channel-match requirement anywhere; Neighbor Report carries a per-candidate channel number precisely because neighbors can be same- or different-channel) and corrected both that page and the checklist to say the different-channel choice is lab convenience, not an 802.11k/v/r requirement. Second follow-up asked to mark, throughout the checklist, which items are testbed/experiment convenience vs. genuine WiFi-roaming requirements — added short plain-language notes to: wired backhaul (§Environment), same subnet/L2 (§Setup & Connection), tx power tuning (§Setup & Connection), steering daemon and RSSI thresholds (§802.11v). Left protocol-required items (SSID/AKM/MDID match, R0KH/R1KH, RRM, FT policy bits, etc.) and the self-evidently experiment-side Verification/Sanity-Checks sections unannotated.

## [2026-07-14] QUERY | Data collection mechanisms in industry AI/cloud WiFi troubleshooting
- Pages: created `2026-07-14-wifi-data-collection-mechanisms.md`; linked from `index.md` under new "Telemetry & data collection" category
- Sources: references/edge-diagnostic-agent-research-direction.md (§3.1, §5, §6, §9 for framing); vendor docs — Juniper Mist (SLE, dynamic pcap, ap-stats, Marvis Client), Cisco Meraki (Health, Proactive PCAP, management-traffic doc), Cisco Catalyst Center (iCAP, AI Analytics admin guide, 9800 MDT), HPE Aruba (Central live troubleshooting, AirMatch, UXI/Cape FAQ), Extreme CloudIQ, Nile (SLA doc, NXI); sensor vendors — 7SIGNAL (Sapphire Eye/Mobile Eye), Wyebot FAQ, ThousandEyes endpoint-agent docs, NetBeez; standards/open source — Wi-Fi Data Elements Jan-2024 overview, TR-369/TR-181/TR-157, OpenSync repo + whitepaper, TIP OpenWiFi PROTOCOL.md, prplMesh ap_metrics.md + issues, Cisco 802.11k blog
- Notes: built from three parallel web-research passes (cloud-WLAN vendors / sensor+client agents / open standards), per-claim URLs inline. Key synthesis: industry universally runs two-tier sensing (cheap always-on streaming + gated deep capture) with hardcoded escalation triggers and static cost controls (Meraki first-failure-per-5-min, Cisco AP-side anomaly aggregation + byte caps) — no learned/cost-aware policy anywhere, corroborating the research direction's §9 gap. No vendor quantifies its own active-test airtime overhead. Unverified claims listed in the page's §7 rather than asserted.

## [2026-07-14] EDIT | Port edge-diagnostic-agent research direction into the wiki
- Pages: created `edge-diagnostic-agent-research-direction.md` (from the former `references/edge-diagnostic-agent-research-direction.md`, now deleted); added a "Research direction" category to `index.md`; updated `2026-07-14-wifi-data-collection-mechanisms.md` (dropped the stale `references/…` frontmatter source, repointed its two body links + `> Sources:` link to the new sibling page)
- Sources: references/80211-2024.pdf (the spec-as-ground-truth anchor: reason/status/BTM codes and state machine, §4 RQ4); the doc is primary authored content, so no external source to summarize
- Notes: the file was an untracked authored research-direction document sitting in the immutable `references/` layer — really wiki-layer content. Ported verbatim with wiki scaffolding (frontmatter, italic summary, `> Sources:`) plus cross-links to the home-roaming scenario (§2/§6 testbed = NIKA `N`/`I`), the common-issues catalog (§4 RQ5 impostor faults), the NIKA page (§6 ground truth), and the data-collection survey (§5/§9 industry corroboration). Also resolves a latent lint issue: the data-collection page had cited `references/edge-diagnostic-agent-research-direction.md`, which was never listed in `references/README.md` (a file-source-not-in-manifest violation) — now a wiki-internal link instead.

## [2026-08-07] QUERY | Extend the data-collection survey with independent monitor/sniffer-node methods
- Pages: updated `2026-07-14-wifi-data-collection-mechanisms.md` (added §6 "The independent passive sniffer node" as a new seventh mechanism; added the mechanism to §1 framing; added a remote-capture transport bullet to §3; wove the sniffer into §7 Relevance as the one existing non-participating observer; renumbered Relevance §6→§7 and Flagged-Unverified §7→§8 with new sniffer-specific entries; added 13 sources + bumped `updated`); updated `index.md` one-line summary
- Sources: Cisco 802.11 sniffing fundamentals (doc 200527), Cisco Catalyst 9800 Sniffer Mode (doc 217057), HPE Aruba AOS-8 AP packet-capture CLI, LiveAction/Omnipeek driver page, Ekahau-pcap and WPA2-decrypt posts (mrn-cciew), WLAN Pi user guide, WLANPiShark2 repo, Intuitibits Raspberry-Pi-remote-sensor and Airtool-2-multichannel posts, Wireshark WLAN capture wiki, Meraki wireless-capture analysis guide, IEEE multi-channel-capture paper (ResearchGate)
- Notes: produced by the deep-research workflow (5 angles, 20 sources fetched, 91 claims → 25 adversarially verified, 24 confirmed / 1 refuted). Core synthesis: the sniffer is the only truly-zero-airtime, network-independent vantage — a separate device (sniffer-mode AP, Ekahau Sidekick 2 / WLAN Pi appliance, or laptop+USB-adapter in monitor mode) that passively records all-SSID mgmt/control/data frames at full radiotap detail; placement near the client is its defining property for roaming, and best practice is a simultaneous AP-side + near-client pair. One refuted claim deliberately excluded (Cisco 9800 export via UDP 5555 / PEEKREMOTE dissector — failed 1-of-3 verification; page states only "forwards to a remote analyzer"). Under-sourced dimensions (WPA2/WPA3/FT decryption limits, multi-sniffer time-sync, 80/160 MHz + spatial-stream capture gaps) and untouched named products (Profitap, NetAlly/AirMagnet, CommView, Nubium) recorded in the page's §8 rather than asserted.
