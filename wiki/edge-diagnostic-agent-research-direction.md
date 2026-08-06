---
title: Cost-Aware Diagnostic Agents for Spec-Governed Edge Systems
tags: [research-direction, diagnostic-agent, sensing-policy, edge, roaming, decision-theoretic-troubleshooting, evaluation]
sources:
  - references/80211-2024.pdf
updated: 2026-07-14
---

# Cost-Aware Diagnostic Agents for Spec-Governed Edge Systems

*Research direction / charter page: Wi-Fi roaming / VoIP degradation is the instantiation; the subject is the diagnostic **policy** — cost-aware, falsification-driven evidence acquisition under measurement interference, where sensing consumes the same contended airtime whose scarcity produces the symptom.*

> Sources: Primary authored research direction (this wiki's charter for the diagnostic-agent work, not a summary of an external document). It engages the decision-theoretic-troubleshooting lineage (Heckerman, Breese & Rommelse, 1995) and the 802.11 self-label taxonomy — deauthentication reason codes, association status codes, BTM response codes — from references/80211-2024.pdf, covered in the [802.11k](80211k-radio-resource-measurement.md) / [802.11v](80211v-wireless-network-management.md) / [802.11r](80211r-fast-bss-transition.md) pages. §9 positions against named commercial systems (Juniper Mist, 7SIGNAL, Cisco DNA/ThousandEyes, Nile, Wyebot) and telemetry standards (Wi-Fi Alliance Data Elements, TR-369/TR-181, OpenSync, OpenWiFi); those are surveyed in [WiFi Troubleshooting Data Collection — Industry Mechanisms](2026-07-14-wifi-data-collection-mechanisms.md).

---

## 1. Thesis

An on-premise agent receives a natural-language symptom report and must localize and root-cause the fault by deciding **what to measure next**. The defining constraint is that measurement is not free and not neutral: in a wireless medium, the act of sensing consumes airtime — the same contended resource whose scarcity produces the symptom. Monitoring therefore degrades the very quality it is trying to measure, and pushed far enough, it manufactures the fault it is looking for.

This makes diagnosis a **sequential experimental design problem under measurement interference**, executed locally under a privacy constraint, and constrained by a formal protocol specification that is available as ground truth.

The research object is the **evidence-acquisition policy**. The ML components (anomaly detection, fault classification) are established techniques and are not the contribution. A platform-level contribution (plugin architecture, message bus, rules engine) would not be evaluable, because generality asserted through software structure produces no falsifiable prediction. A policy-level contribution is evaluable: it predicts a measurable frontier between diagnostic yield and network harm.

---

## 2. Problem class

The direction targets systems with all five properties:

1. **Causal distance.** The user-visible symptom (degraded call audio) is several layers removed from the protocol-level fault (a key-hierarchy misconfiguration causing full reauthentication on handoff).
2. **Distributed partial observability.** No single observer sees the whole event. The serving access point cannot see the client's roam decision; neighbor access points hold fragments of it.
3. **Privacy-bound telemetry.** MAC addresses, association histories, and traffic patterns are personal data. Processing is therefore local, which imposes hard compute and storage budgets.
4. **Specification as ground truth.** The system under diagnosis conforms (or fails to conform) to a published state machine, and emits self-describing diagnostic codes — 802.11 deauthentication reason codes, association status codes, BSS Transition Management response codes.
5. **Monitoring is not free.** Observation draws down the same contended resource that user traffic depends on.

Wi-Fi is the instantiation — realized concretely as the [2-AP home-roaming testbed](home-wifi-roaming-scenario.md). The class also contains BLE/Thread/Matter meshes, O-RAN and private 5G, and industrial fieldbus networks. Property 5 is what distinguishes this class from conventional distributed-systems observability, where instrumentation overhead is a nuisance rather than a confound.

---

## 3. The role of the AI agent

**The agent is a diagnostician whose only motor action is data collection.** Orchestration is the mechanism, not the subject. Every belief update the agent can make must be purchased through an evidence action — it cannot interrogate a witness or read a stack trace. The action space is therefore made entirely of collection decisions, but the agent performs four distinct functions:

1. **Interpretation** — translate the natural-language symptom report into a prior over hypotheses and a time window.
2. **Belief maintenance** — hold and revise a distribution over the hypothesis set.
3. **Evidence selection** — choose the next collection action. *This is the orchestration.*
4. **Attribution and termination** — rank causes when several fire together; decide when to stop.

Remove (1), (2) and (4) and what remains is a scheduler, not an agent. The distinction matters for how the contribution is stated: the **orchestrator** executes (fan out to access points, collect, stitch traces); the **policy** decides. Conflating the two is how a systems design comes to describe a scheduler as an agent.

### 3.1 Relationship to decision-theoretic troubleshooting

Framed only as "choose the cheapest sequence of tests to identify a fault from a known set," this problem is **decision-theoretic troubleshooting** (Heckerman, Breese & Rommelse, 1995) and the test-sequencing literature that follows it: a Bayesian network over faults, cost-annotated tests, and greedy expected-value-of-information selection. That machinery is mature, and an agent that does only this is a reimplementation of it.

The novelty of this direction lies in three specific ways the Wi-Fi problem **breaks** that formulation:

**(a) Tests cannot be applied to the past.** Classical test selection assumes tests are available on demand. Here the diagnostic question arrives after the evidence window has closed, and the evidence exists only if the agent had already decided to retain it. The problem acquires **irreversible information loss** and **speculative pre-payment** — a decision to pay now for the future answerability of a question not yet asked. The classical formulation has no representation for this.

**(b) Test cost is endogenous.** In classical troubleshooting, cost is exogenous — money, technician time. Here, the test consumes the resource whose scarcity is under investigation. Cost and evidence are coupled through the same variable. This changes the structure of the cost model, not merely its parameters.

**(c) The fault set is open.** Bayesian troubleshooting requires a closed, enumerated fault set. Firmware defects and vendor-specific client pathologies are not enumerable in advance.

### 3.2 Where the LLM earns its place

Point (c) locates the LLM precisely. Its role is **hypothesis generation over an open fault set**, interpretation of the natural-language report, reading of heterogeneous unstructured evidence, and explanation of the final diagnosis. Hypothesis *selection* — ranking, evidence weighting, cost-aware test choice — is better served by the classical machinery. The boundary between these two is itself a research question (RQ4).

---

## 4. Core research questions

### RQ1 — Temporality: what can a post-hoc query recover?

A symptom report ("calls were lagging just now") refers to a **completed** event. Active probing cannot reach it. This partitions the agent's action space into four modalities with different epistemic reach:

| Modality | Reaches the reported incident? | Cost |
|---|---|---|
| **Retrospective** — interrogate retained evidence | **Yes — the only one that does** | zero (already paid) |
| **Prospective** — instrument and await recurrence | No; awaits a second occurrence | deferred |
| **Reproductive** — re-elicit the fault | No; establishes the fault class is live | high |
| **Interventional** — alter configuration and observe | No; diagnosis by treatment response | highest |

**The question:** what must have been retained, before any query arrives, for a diagnosis to be reachable at all? This inverts the usual framing. The agent's competence is bounded not by its reasoning but by a **retention policy committed to before the symptom existed**. Probing establishes that a fault class is currently active; it cannot establish that this class caused Tuesday's dropped call. The limits of that inference are a research result in themselves.

### RQ2 — The sensing policy is the action space

Because of RQ1, the decision of what to record continuously, what to record speculatively, and what to record never **is** the agent's primary action space. An agent whose actions do not include *changing what it observes* is a retrieval interface over a fixed log.

The policy problem has an unusual structure: a **speculative** cost, paid in advance for an event that may not occur, against a **retrospective** benefit that only materializes if it does. The escalation decision — raise the sensing tier now, because a fault appears imminent — is a bet under uncertainty where the payoff is *the future answerability of a question not yet asked*.

**The question:** can this policy be learned from environment and deployment history, and does it outperform static sensing configurations on the frontier defined in §6?

### RQ3 — Probe selection under measurement interference

Classical sequential experimental design selects the next measurement to maximize expected information gain per unit cost. Here, cost is not latency or money but **interference**: a probe consumes airtime and can degrade the very quality metric being measured. In the limit, the diagnostic action manufactures the symptom.

Two sub-questions:

- **(a) Falsification versus confirmation.** A hypothesis-driven agent — particularly an LLM-planned one — tends to select evidence that supports its leading hypothesis. A diagnostic agent must select the probe that maximally *partitions* the hypothesis set. Whether LLM-based planners exhibit measurable confirmation bias in probe selection, and whether an explicit information-theoretic criterion corrects it, is directly testable.
- **(b) Stopping.** Diagnosis has no natural terminal state. A principled halting condition — expected information gain falls below the monitoring cost, or confidence becomes sufficient for the cheapest *reversible* remediation — is underexplored and is required for the frontier in §6 to be well-defined.

### RQ4 — Symbolic grounding against the specification

The protocol state machine is available as formal ground truth. This admits a constraint most agent domains cannot impose: a hypothesis that is **protocol-illegal given the observed trace** can be rejected mechanically, independent of the agent's confidence in it. A conformance checker functions as a hallucination guardrail.

**The question:** how much of the diagnostic loop must remain symbolic? Specifically, where is the boundary between what the specification decides and what a learned model decides?

The knowledge in this domain stratifies:

| Stratum | Source | Learning required |
|---|---|---|
| Protocol self-labels (reason/status/BTM codes) | The protocol labels its own failures | None |
| Specification-anchored thresholds (handoff latency budget, ITU G.114 mouth-to-ear limit) | Application and spec; site-independent | None |
| Client behavior; contextual normalization; attribution; residuals | Empirical only | Irreducible |

The third stratum is irreducible for a specific reason: **client roaming and power-save algorithms are proprietary and deliberately undocumented.** No specification states an iPhone's roam threshold or scan cadence. They can only be learned empirically. This is the strongest justification for the learned component, and it constrains where that component belongs.

The two strata that require no learning also mean a purely learned baseline detector will **normalize a chronically broken site** — if a fault has always been present, it becomes the baseline. Only absolute, specification-anchored constraints detect it.

### RQ5 — Attribution under co-occurrence

In a real degradation, multiple detectors fire simultaneously: a roam occurred, *and* channel utilization was elevated, *and* signal strength was marginal, *and* authentication-server round-trip time was high. Deterministic rules detect conditions; they do not rank causes.

Compounding this, several fault classes present identically to the user while having nothing to do with roaming — power-save mishandling, QoS misclassification at a trust boundary, airtime starvation by a legacy client, and radar-triggered channel changes all produce "the call broke." **Separating the roaming faults from their impostors is the classifier's actual task**, and it is why airtime and QoS telemetry are load-bearing even in a study nominally about roaming. (These concrete fault classes are cataloged in [Common WiFi Roaming Issues](2026-06-08-wifi-roaming-common-issues.md).)

### RQ6 — Memory, and the entrenchment failure

Memory has three functions worth studying, only one of which is well covered by existing agent-memory literature:

- **Probe amortization.** A fault diagnosed once at a cost of six probes should cost near-zero to diagnose again. This yields the direction's cleanest learning metric: **probes-per-diagnosis and monitoring-cost-per-diagnosis, decaying over deployment time.**
- **Learned affordances.** Memory should store which probes were **uninformative at this site** — a beacon-report request is worthless if the site's clients do not implement the relevant radio-measurement extensions. Existing agent-memory work accumulates successes; in a cost-constrained loop, the negative results are what reduce cost. This is the escalation policy of RQ2 learning itself.
- **Precursor signatures.** Learned patterns that precede a fault, enabling the speculative escalation of RQ2.

**The failure mode, which is the sharpest claim in the direction:** memory entrenches. An agent that has learned "at this site, call-quality complaints are the switch uplink" ceases to probe, and becomes blind to a *novel* fault presenting with the same surface symptom. Memory converts the agent into a confirmation-bias machine that grows more confident as it grows more wrong.

This implies an **exploration budget dedicated to hypothesis maintenance**: probes spent deliberately on hypotheses the memory deems unlikely, to keep priors calibrated. This is exploration–exploitation where exploitation is not merely suboptimal but **epistemically corrosive**.

The claim is falsifiable and cheap to test: train memory on fault A, inject fault B with an identical symptom, and measure whether the memory-augmented agent underperforms the memoryless one.

---

## 5. Scope: what is research, what is engineering

| Research | Engineering (deferred; design space open) |
|---|---|
| The escalation **policy** — when to raise sensing tier, what to arm, what to spend | Capture **mechanism** — buffering strategy, trigger semantics, storage layout |
| Probe **selection criterion** | Probe **implementation** |
| Where the symbolic/learned boundary sits | Which detector library implements the symbolic half |
| The cost model's *structure* (airtime cost is hardware-conditional and action-specific, not uniform across "monitoring") | The measured cost *values* for a given radio configuration |
| Whether the site-invariant / site-specific model split holds | Serving stack, model format, quantization |

The monitoring cost model is the one place these interact, and it must be settled early at the *structural* level: cost is not a property of a capability but of a capability-on-hardware pairing. Passive monitoring on a radio dedicated to sensing and passive monitoring on a radio simultaneously serving clients are the same capability with categorically different costs. A cost model indexed only by action type is therefore wrong, and no downstream policy can be correct on top of it. The [industry data-collection survey](2026-07-14-wifi-data-collection-mechanisms.md) corroborates this: vendors physically separate sensing from serving radios precisely because the two pairings cost differently.

---

## 6. Evaluation

### Primary metric

**Unanswerable-query rate** — the fraction of symptom reports for which retained evidence was insufficient to reach a diagnosis — minimized subject to a **monitoring budget** (degradation inflicted on the network by sensing).

### The frontier

Plotting unanswerable-query rate against inflicted quality degradation yields a Pareto frontier over sensing policies. Two corners are trivially known: exhaustive always-on sensing (perfect evidence, degraded network) and no sensing (pristine network, no diagnostic capability). **The claim under test is that a learned, escalating policy occupies the knee — that it approaches the diagnostic yield of exhaustive sensing at a fraction of the harm.** If this frontier does not separate the policy from static configurations, the direction fails, and it fails early and cheaply. This experiment should be built first.

### Required arms

- **Control** — no monitoring. Establishes baseline quality. Without it there is no denominator for inflicted harm.
- **Oracle** — exhaustive monitoring. Establishes maximum achievable diagnostic yield and maximum harm.
- **Classical baseline** — cost-sensitive Bayesian test selection over the enumerated fault set, without an LLM (§3.1). This arm is mandatory. If the agent does not beat it, the LLM is not earning its cost; if it does, the margin isolates which of the three departures in §3.1 is doing the work.
- **Policy** — the agent.

### Methodological constraint

**Monitoring overhead cannot be measured with the instruments causing it.** If sensing consumes airtime and quality is measured via telemetry from the sensing radios, the measurement is confounded. Evaluation requires a **non-participating observer** — a measurement path permanently outside the agent's action space, used only to score the arms.

### Ground truth

Fault injection supplies labeled traces: the protocol's own reason and status codes label the fault, and the injection is reproducible. This resolves the labeling problem that otherwise blocks supervised root-cause classification, and it is what makes the direction tractable at testbed scale. This is the `I` (injected fault) component of a [NIKA-style `(N, I, T)`](nika-problem-specification-summary.md) problem specification over the [home-roaming scenario](home-wifi-roaming-scenario.md).

### Secondary metrics

Probes-per-diagnosis over deployment time · localization accuracy (time window and topological element) · time-to-diagnosis · precision, weighted above recall — a confident wrong diagnosis destroys operator trust faster than a missed one.

---

## 7. Generality: what is claimed and how it is demonstrated

The claim is **not** that the software works for arbitrary problems. It is that the **policy** — cost-aware, falsification-driven evidence acquisition under measurement interference, with specification-based symbolic grounding — transfers to the problem class of §2, while the following remain domain-supplied:

- the conformance specification and its state machine
- the event schema and self-label taxonomy
- the hypothesis set
- the action set, annotated with cost **and network impact**
- the fault-injection harness

Demonstration requires **one deep instantiation plus one shallow but genuine second** — a second protocol in the class (a BLE/Thread/Matter mesh shares the distributed-observer, privacy-bound, specification-governed structure and would run on the same edge host). The generality artifact is the **interface plus two instantiations**, not a framework.

Sequencing matters: the Wi-Fi instantiation must be built deeply *before* the interface is extracted. Abstraction here is a refactoring, not a design phase — the requirement that action cost encode *airtime consumption* is discoverable only from the domain, and an interface designed in advance of that discovery will not have a place to put it.

---

## 8. Open questions to settle before committing

- **Subject identification.** "My calls" must resolve to a station. MAC randomization makes the user-to-device binding a genuine research dependency, not a lookup. If the symptom report arrives through an application on the affected device, the binding is free — which is an argument about the query channel, not the query interface.
- **The report as a prior, not an observation.** "Just now" is a distribution over time windows; "lagging" is a distribution over symptom classes. The natural-language report should enter the system as a **prior over (time window × symptom class × hypothesis subset)**, not as a timestamped fact. How to calibrate that prior is unresolved.
- **Complaint-driven sampling bias.** Reports arrive only when a user is annoyed. Silence is not evidence of health, and the query-generating mechanism systematically skews the observed fault distribution.
- **Authentication mode.** WPA2-PSK and 802.1X produce entirely different roaming-latency profiles, and the interesting key-hierarchy failures exist only under 802.1X. This choice determines which fault classes are even reachable in the testbed.
- **Client population.** Real handsets, softphones, or synthetic traffic generators determine how much client-side ground truth is available for validating an access-point-side-only model.
- **Remediation coexistence.** If the agent acts, it acts against vendor radio-resource-management and steering daemons that are making their own decisions. Whether the agent's scope stops at diagnosis is a scoping decision with consequences for the whole action space.

---

## 9. Positioning

Existing commercial systems occupy adjacent ground and should be read for what they **could not** do: Juniper Mist (whose diagnostic advantage rested on client-side telemetry, unavailable to an access-point-side system), 7SIGNAL (sensor-based active testing — closest to the measurement-interference problem, without the cost-aware policy), Cisco DNA/ThousandEyes, Nile, Wyebot. On the telemetry-standards side: Wi-Fi Alliance Data Elements, TR-369/TR-181, OpenSync, OpenWiFi. These systems and standards, and the two-tier (cheap always-on + gated deep capture) sensing structure they share, are surveyed in [WiFi Troubleshooting Data Collection — Industry Mechanisms](2026-07-14-wifi-data-collection-mechanisms.md).

The relevant academic lineage is **decision-theoretic troubleshooting and test sequencing** (§3.1), and **active learning / optimal experimental design** for the evidence-selection criterion. Neither treats monitoring cost as endogenous to the fault being diagnosed, and neither addresses evidence that must be retained before the question is asked.

Commercial systems do not treat monitoring cost as a decision variable at all. Those two gaps, jointly, are the direction.
