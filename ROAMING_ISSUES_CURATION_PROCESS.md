# Roaming Issues Curation Process

How this wiki was built — from setup to the curated roaming-issue catalog
(`wiki/2026-06-08-wifi-roaming-common-issues.md`).

Two directories are involved:

- **The LLM wiki repo** — this repo, `wifi-troubleshooting-research-wiki`
  (originally created as `fast-roaming-troubleshooting-research`, later renamed).
- **The NIKA codebase** — `../nika`, the network-troubleshooting benchmarking tool whose
  fault/scenario model and "Network Issues" taxonomy this wiki borrows.

Each step below shows the conversation, in order, with the directory it ran in. Three
entities appear throughout: the **User** (the typed prompt), the **Assistant** (the LLM
acting on it), and **Claude Code** (the harness that injects skill text and attaches `@`
files).

## 1. Set up the LLM wiki repo (Karpathy structure)

Define the three-layer architecture, consolidate the schema, then bootstrap-commit the repo.

**Directory:** the LLM wiki repo

**User:**

```
I am building a simple wiki for my research about wifi fast roaming troubleshooting.
Refer to Karpathy's idea at https://gist.githubusercontent.com/karpathy/.../llm-wiki.md

For now, I don't need advanced workflows like automated ingest, the search tools, etc.
You should just focus on maintaining a consistent wiki structure, particularly:

1. The three layer architecture concept: references/ (raw sources), wiki/ (generated
   pages), and prepare CLAUDE.md. Add only *NECESSARY* info — no over-explained details,
   unnecessary tables, etc
2. Operations: no automated "Ingest"; wikis generated manually following my "Query"; do "Lint"
3. Indexing and Logging: follow "index.md" and "log.md". Propose the structure of log

Clarify to me for any specifics
```

**Summary of Assistant actions:**

1. Set up the three-layer structure (`references/`, `wiki/`, `CLAUDE.md`).
2. Authored `CLAUDE.md` with the Query/Lint operations and page schema.
3. Proposed the `index.md` registry and append-only `log.md` format.
4. Asked the User clarifying questions about scope.

**Directory:** the LLM wiki repo

**User:**

```
This wiki is for network troubleshooting research focusing on wifi roaming issues.
Consolidate any schemas that is still ambiguous (CLAUDE.md, index.md, etc)
```

**Summary of Assistant actions:**

1. Reviewed `CLAUDE.md` and `index.md` for ambiguity.
2. Fixed the frontmatter fields, page structure, and provenance rules.

**Directory:** the LLM wiki repo

<a id="step1-flat-wiki"></a>

**User:**

```
Using /deep-research skill is just an example. I am just asking for a consistent
organization to store pages that are synthesized across many sources (e.g comparisons,
deep research reports, charts, etc). The question: Would organizing to @wiki/research/
be necessary or not?
```

**Summary of Assistant actions:**

1. Recommended keeping `wiki/` flat (no `wiki/research/` subfolder).
2. Filed synthesized pages as normal wiki pages under `index.md` categories.
3. Bootstrap-committed the repo.

## 2. Distill the NIKA framework

Understand NIKA's fault-injection and scenario model; produce `../nika/docs/` and the wiki's
`nika-problem-specification-summary.md`. This is where the
`(Category, Root Cause, Key Signals)` and scenario framing come from.

**Directory:** the NIKA codebase

**User:**

```
This repo is an AI agent benchmarking tool for network troubleshooting tasks.
Analyze how they reproduce certain faults or incidents.

Context:
- The paper mentioned that an incident can involve faulty configurations, traffics, etc
```

**Summary of Assistant actions:**

1. Explored the NIKA repo's fault-injection code.
2. Explained how incidents (faulty configs, traffic, component failures) are reproduced.

**Directory:** the NIKA codebase

**User:**

```
Document this summary in ./docs/
```

**Summary of Assistant actions:**

1. Wrote the analysis to `docs/fault-reproduction.md`.

**Directory:** the NIKA codebase

**User:**

```
Summarize in markdown file about how topology scenario is defined,
created, and provided to the agent in @docs/
```

**Summary of Assistant actions:**

1. Wrote `docs/topology-scenarios.md` covering how scenarios are defined, built, and
   exposed to the agent (topology, configs, telemetry).

**Directory:** the LLM wiki repo

**User:**

```
Read and understand @references/nika-benchmarking-paper.pdf .
Then, create a summary about problem specification method in markdown file
```

**Summary of Assistant actions:**

1. Read `references/nika-benchmarking-paper.pdf`.
2. Produced `nika-problem-specification-summary.md`, capturing the `(Dev, Comp, RC)`
   problem-specification formalism.

## 3. Seed wiki content from the 802.11 standard

Generate the 802.11k/v/r pages from the spec, then build a first knowledge-based prototype of
the scenario and curated issues.

**Directory:** the LLM wiki repo

**User:**

```
Read @references/80211-2024.pdf and create summary in markdowns about these topics.
The PDF is 5000-pages long so you should just use optimized approach to find
respective sections without brute forcely reading all documents:

802.11k Radio Resource Measurement, Neighbor Report, RRM
802.11v Wireless Network Management, BSS Transition Management, WNM
802.11r Fast BSS Transition, FT, Mobility Domain
```

**Summary of Assistant actions:**

1. Located the relevant clauses without reading the whole 5000-page spec.
2. Drafted the 802.11k, 802.11v, and 802.11r summary pages.

**Directory:** the LLM wiki repo

**User:**

```
You can create multiple markdowns. Continue
```

**Summary of Assistant actions:**

1. Finished the three pages — `80211k-radio-resource-measurement.md`,
   `80211v-wireless-network-management.md`, `80211r-fast-bss-transition.md`.

**Directory:** the LLM wiki repo

**User:**

```
Two tasks for you:
1. Define a new network scenario under the framing of @nika-problem-specification-summary.md .
   - Home wireless network scenario consisting of 2 wireless access points
   - Cover possible roaming issues related to the 3 roaming topics (11r/11k/11v)
2. Curate roaming issues under the frame of Network Issues section in @../nika/README.md
   (Category, Root Cause, Key Signals)
   - You can create new category if it does not fit into any of existing categories
   - State the sources of where you come up with the root cause (e.g. Opus 4.8 knowledge)
   - Infer the key signals from the 3 roaming markdown documents you created
   - Collect total of 10 issues across different categories
```

**Summary of Assistant actions:**

1. Drafted a first 2-AP home scenario.
2. Curated a knowledge-based prototype of 10 roaming issues in
   `(Category, Root Cause, Key Signals)` form, sourced from LLM knowledge and the 802.11 pages.

## 4. Produce the curated catalog

Define the scenario and run `/deep-research` to turn the prototype into a source-verified
10-issue catalog.

**Directory:** the LLM wiki repo

<a id="step4-user-prompt"></a>

**User:**

```
Two tasks:
1. Define a new network scenario under the framing of @wiki/nika-problem-specification-summary.md
   - Home wireless network of 2 wireless access points
   - For troubleshooting wifi roaming issues (11r/11k/11v pages)
   - Define topology (nodes, links, per-device configs); assume a QCA OpenWrt router AP
   - List common telemetry/monitoring for this kind of home network
2. Use /deep-research to collect and curate common issues related to wifi roaming.
   - Use the "Network Issues" structure in @../nika/README.md (category, root cause, key signals)
   - Create new category if it does not fit the available categories
   - State the sources of how you come up with the list of root causes and key signals
   - Wifi roaming resources in this wiki can be used as references too
   - Collect total of 10 issues across different categories
```

**Summary of Assistant actions:**

1. Wrote `home-wifi-roaming-scenario.md` (topology, per-device configs, telemetry).
2. Composed a detailed research question for task 2 from the requirements.
3. Invoked the `/deep-research` skill (see the expansion below).

### How the `/deep-research` invocation reached the model

Task 2 of [the User message above](#step4-user-prompt) only *mentions* `/deep-research` in
prose. The prompt the LLM actually receives is assembled by the Assistant and the Claude Code
harness — the User never types it. The flow:

1. The User references the `/deep-research` skill inside a natural-language request.
2. The Assistant composes a detailed research question from the requirements and invokes the
   skill, passing that question as the skill's `args`.
3. The Claude Code harness injects the skill's own instruction text — its description, phases,
   and an example invocation — into the conversation as a hidden block (shown below).
4. The Assistant runs the workflow with the composed `args`, executing the multi-angle research.

The harness-injected block:

```
Run the "deep-research" workflow.
Deep research harness — fan-out web searches, fetch sources, adversarially verify claims,
synthesize a cited report.
... (skill description + phases) ...
Invoke: Workflow({ name: "deep-research", args: "Collect and curate the most common,
well-documented WiFi roaming problems (802.11k/v/r-era enterprise & home WLANs ...
10 distinct issues, each as (Category, Root Cause, Key Signals) ... prioritize IEEE
802.11-2024, Cisco/Aruba/Mist/Juniper/Ruckus, hostapd/OpenWrt, Wi-Fi Alliance, mrn-cciew
... drop into a (Category | Root Cause | Key Signals | Source) table." })
```

In short, the conversion is: *User prose requirement → Assistant-authored skill `args` →
harness-injected skill instructions.* A plain `@file` reference is simpler — the harness just
attaches the file's contents inline.

**Result:** deep-research ran (5 search angles, 21 sources, 25 claims adversarially verified →
24 confirmed, 1 refuted) and synthesized the final `2026-06-08-wifi-roaming-common-issues.md`
catalog with per-row sources.
