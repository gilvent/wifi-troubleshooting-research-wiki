---
title: NIKA — Problem Specification Method
tags: [nika, benchmarking, network-troubleshooting, ai-agents, problem-specification]
sources: [references/nika-benchmarking-paper.pdf]
updated: 2026-06-08
---

# NIKA — Problem Specification Method

*How NIKA defines a reproducible troubleshooting "problem": an incident specification (N, I, T)
coupled with a troubleshooting goal (G), giving one structured ground truth scored across objectives.*

> Sources: *A Network Arena for Benchmarking AI Agents on Network Troubleshooting* (Wang et al., 2025), §3.2.

## Core idea

In NIKA, every benchmark problem is a self-contained, reproducible unit defined by **two coupled elements**:

1. an **incident specification** — *how to reproduce the failure* in a live network, and
2. a **troubleshooting specification** — *what task the agent must perform* and how it is scored.

This separation lets the same underlying fault be re-used across many scenarios and graded against several objectives, while keeping a single ground truth to evaluate against.

```
Problem  =  Incident specification (N, I, T)   +   Troubleshooting goal (G)
            └── how to reproduce the fault ──┘       └── what to solve / how scored ──┘
```

## 1. Incident specification — the tuple (N, I, T)

The incident is a **three-tuple** `(N, I, T)`:

### N — Network scenario
Describes the environment the incident lives in:
- network topology, devices, protocols, and initial configurations;
- the control plane needed to bring all nodes up;
- which **telemetry instrumentation** is deployed (INT, sketches, flow collectors, etc.), so users can restrict the agent to user-selected telemetry methods (e.g., legacy clusters that lack advanced switch features).

NIKA ships canonical, ready-to-use scenarios (Table 4): data center (CLOS), 3-tier campus, meshed ISP backbone, SDN-enabled cloud POP fabric, and a P4 programmable testbed. Four of the five are **scalable** to small/medium/large topologies (~11 / 27 / 101 nodes).

### I — Network issue (the ground truth)
The actual fault, and the **ground-truth evidence** against which agent output is judged. Each issue is formalized as a triple:

```
I = (Dev, Comp, RC)
    │     │     └── Root Cause       e.g. link_down, wrong_area
    │     └──────── affected Component   e.g. eth0, OSPF
    └────────────── Device              e.g. switch1, router2
```

Examples:
- `I = (switch1, eth0, link_down)` — switch1's eth0 interface is down due to a link failure.
- `I = (router2, OSPF, wrong_area)` — router2's OSPF is misconfigured with an incorrect area assignment.

The benchmark spans **54 representative issues** across six categories (Table 1 / Table 3): link failures, end-host failures, network node errors, misconfigurations (routing/ACL), resource contention, and network-under-attack — totaling **640 distinct incidents** when combined with scenarios and topology sizes.

### T — Traffic workload
Describes the traffic generated while reproducing the incident, in two parts:
- **regular traffic** — the normal-operation traffic matrix;
- **incident-triggering traffic** — patterns designed to exacerbate the issue `I` (e.g., overloading specific components so a misconfiguration manifests with severe symptoms).

By mixing these, domain experts can **modulate symptom visibility and severity**, controlling how hard the problem is.

### Incident variations (extensibility)
Because `(N, I, T)` is modular, NIKA generates many variants of one underlying issue:
- combine an issue `I` with different scenarios `N` and workloads `T`;
- swap affected devices/root causes via **parametric templates** (e.g., link failure on a different switch, or "link down" vs. "high error rate");
- compose **multiple issues** in one run (e.g., link failure + routing misconfiguration).

This yields "practically unlimited" incident variants from a small curated core.

## 2. Troubleshooting specification — the goal G

Each problem is paired with a **troubleshooting goal `G`**, following a hierarchical taxonomy of increasing difficulty (from prior cloud-diagnosis work):

| Goal | Task | Output form | Scoring |
|------|------|-------------|---------|
| **Detection** | Binary: is the network state normal or anomalous? | yes/no | confusion matrix → accuracy; disambiguates false positives/negatives |
| **Localization** | Identify responsible components | multi-class selection over `I_Dev` and `I_Comp` | boolean masks vs. ground truth → accuracy |
| **Root Cause Analysis (RCA)** | Additionally suggest the underlying root cause `I_RC` | boolean mask | accuracy from confusion matrix |

Because the ground truth `I = (Dev, Comp, RC)` is structured, each goal maps cleanly to a goal-specific evaluator. Beyond accuracy, NIKA also reports efficiency metrics (time-to-resolution, tool invocations, token usage) and tracks flow latency / packet loss during diagnosis to capture **side effects on network performance** and SLO violations.

## 3. How a specification becomes an executable problem

The specification is **declarative but executable** — it is realized in code that the orchestrator drives (Fig. 3):

- A problem is a class (e.g., `DataCenterIncast(IncidentBase)`) carrying:
  - a `prompt_detection` / agent role,
  - network-issue parameters (the `N`, `I`, `T` values),
  - an `inject_incident()` method (sets up issue-triggering traffic and the fault),
  - a `run()` method that encapsulates the **core logic to reproduce the incident** and manages its lifecycle,
  - an `eval()` hook keyed to the goal `G`.
- The NIKA **orchestrator** consumes `(N, I, T)` and *materializes* it into a live emulated network (built on Kathará): it deploys the topology, configs, and telemetry instrumentation, then injects the fault and generates traffic via runtime APIs (Linux TC for link issues, `stress-ng` for software contention, custom scripts for device failures/misconfigurations).
- Fault injection and traffic generation are exposed through **Runtime APIs**; the agent observes and acts only through the **Agent Access Layer (AAL)**, which provides diagnosis tools, telemetry collectors, access-control policies, and observability.

## Why this design matters

- **Reproducibility:** a tuple + run method makes each incident a deterministic, replayable scenario.
- **Separation of concerns:** issue (`I`) is decoupled from environment (`N`) and stress (`T`), so one fault generalizes across networks and difficulty levels.
- **Single ground truth, multiple objectives:** the structured `I = (Dev, Comp, RC)` simultaneously serves detection, localization, and RCA scoring.
- **Extensibility:** parametric templates and issue composition turn 54 curated issues into 640+ incidents (and beyond), without hand-authoring each one.
