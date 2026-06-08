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
