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
