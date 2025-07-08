# Code Crew – Project Document

## Version 2.0 · 08 July 2025

---

### Table of Contents

1. Context
2. Key Objectives
3. Project Scope
4. Stakeholders
5. Main Assumptions
6. High‑Level Architecture
7. Components (Conductor CLI, Sandbox Pool, Observability)
8. Detailed Workflow
9. `code‑crew` CLI – Design & Usage
10. KPI & Dashboard
11. Security
12. Governance & Alerts
13. Technical Roadmap (local system)
14. Open‑Source Strategy
15. OSS + CLI Roadmap
16. Risks & Mitigations
17. Attachments

---

## 1 · Context

This project targets **CTOs, staff engineers, and open‑source maintainers** who want to run a *team* of specialised **AI agents** for software development. The agents are powered by Claude Code and can collaborate across **multiple, even inter‑connected GitHub repositories** (federated monorepos, shared libraries, micro‑services). They tackle features, bug‑fixes, refactors, and optimisations with automatic coordination.

A human remains the final decision‑maker (PR merge, policies); the system delivers operational autonomy, observability, and a CLI to orchestrate the agents.

## 2 · Key Objectives

1. **Agent autonomy** with ready‑to‑review pull requests.
2. **Human governance** (manual PR approval).
3. **Progressive quality** (upward trends in tests & linting, downward debt).
4. **Local‑first**: everything runs on a single machine.
5. **Live observability** with 4‑h stalled‑task alerts.
6. **Open‑source friendly**: Apache 2.0 licence and CLI usable by third parties.

## 3 · Project Scope

| In scope                                             | Out of scope                       |
| ---------------------------------------------------- | ---------------------------------- |
| Vertical agents backend / frontend / test / refactor | Generic multitask agents           |
| GitHub Issues ↔ PR integration                       | Running *agents* on GitHub Actions |
| Local Docker sandbox                                 | Languages other than TS/JS         |
| Local Prometheus + Grafana                           | Advanced regulatory compliance     |
| Agent management CLI                                 | —                                  |

## 4 · Stakeholders

| Role                           | Responsibility                      |
| ------------------------------ | ----------------------------------- |
| Product Owner                  | Functional roadmap                  |
| **Tech Lead / Human Reviewer** | Backlog, PR reviews, KPI monitoring |
| Part‑time Dev Ops              | Docker & observability stack        |
| OSS Community                  | Issues & improvement PRs            |

## 5 · Main Assumptions

* Anthropic API key available.
* Laptop/host with ≥ 16 GB RAM; Docker Desktop with 4 GB allocated.
* GitHub PAT with `contents`, `issues`, `pull_requests` scopes.

## 6 · High‑Level Architecture

```
┌──────────────┐        ┌──────────────────────┐
│ GitHub Issue │◄──────►│ Conductor (Node/TS)  │
└──────────────┘        └───────┬──────────────┘
        ▲  CLI cmd              │   /metrics
        │                       ▼
   code‑crew               ┌──────────────┐
      (npm)                │ Sandbox Pool │  (Docker, 1–N)
                           └──────────────┘
                                  ▲
                                  │  logs/stdout
                                  ▼
                           ┌──────────────┐
                           │ Grafana ← Prom + Loki │
                           └──────────────┘
```

## 7 · Components in Detail

### 7.1 Conductor (core)

* **Node 22 + TypeScript**
* Modules: `github.ts`, `scheduler.ts`, `sandbox‑driver.ts`, `metrics.ts`
* Config file: `conductor.config.yml` (poll interval, concurrency, tokens, agents list)

### 7.2 Sandbox Pool

| Image            | Stack                                            | Label    |
| ---------------- | ------------------------------------------------ | -------- |
| `agent-backend`  | node:22‑slim, Claude Code, ts‑node, Jest, Prisma | backend  |
| `agent-frontend` | node:22‑slim, Claude Code, Next 14, Playwright   | frontend |
| `agent-test`     | node:22‑slim, Claude Code, Jest, nyc             | test     |
| `agent-refactor` | node:22‑slim, Claude Code, ESLint, SonarJS       | refactor |

* Default `--network=none`; hard timeout 3 h 45 m.

### 7.3 Observability (docker‑compose)

* **Prometheus** scraping `/metrics`
* **Loki** for JSON logs
* **Grafana** dashboard for KPIs

## 8 · Detailed Workflow

1. **Issue** labelled `agent:*` is opened.
2. **Scheduler** enqueues the task (FIFO).
3. **Clone / update** repo in `~/agents`.
4. Create **branch** `agent/<type>/<slug>`.
5. **Sandbox** executes `claude fix "<prompt>"`.
6. Run local **tests** (`npm test`), generate tests if missing.
7. **Success** → commits & pushes; opens PR labelled `needs‑human‑review`.
8. **Fail** → comments Issue with log link.
9. **Human review** → merge.
10. **Metrics** recorded to Prometheus.

## 9 · `code‑crew` CLI – Design & Usage

### 9.1 Installation

```bash
npm i -g code-crew
code-crew init    # wizard (API key, PAT, workspace path)
```

### 9.2 Main Commands

| Command                    | Alias  | Action                                |
| -------------------------- | ------ | ------------------------------------- |
| `start [-d]`               | `run`  | Start / daemonise Conductor           |
| `stop`                     | —      | Stop Conductor                        |
| `status`                   | `stat` | Snapshot of sandbox pool, queue, KPIs |
| `agent add <name> <image>` | `a+`   | Register a new agent                  |
| `agent rm <name>`          | `a-`   | Disable an agent                      |
| `config set <key> <val>`   | `cfg`  | Change runtime config (hot‑reload)    |
| `doctor`                   | `diag` | Environment diagnostics               |

### 9.3 Configuration Files

* **Global**: `~/.code-crew/conductor.config.yml`
* **Per‑repo override**: `<repo>/.code-crew.yml`

### 9.4 Typical Workflow

```bash
code-crew init
code-crew agent add backend ghcr.io/org/agent-backend:1.0
code-crew start -d
code-crew status
# … review PR on GitHub …
code-crew stop
```

## 10 · KPI & Dashboard

| KPI                       | Source                  | Grafana panel | Threshold    |
| ------------------------- | ----------------------- | ------------- | ------------ |
| **Cycle Time (h)**        | prom `cycle_time_hours` | Histogram     | > 4 h        |
| **Pass Rate (%)**         | success/total           | Gauge         | < 95 %       |
| **Coverage Δ**            | nyc diff                | Bar           | +2 pp target |
| **Review Iterations**     | PR comment count        | Heatmap       | > 3          |
| **Throughput (tasks/wk)** | pipeline                | Line chart    | trend        |

## 11 · Security

* Credentials stored locally with `chmod 600`.
* Sandboxes run in isolated network.
* No data egress.

## 12 · Governance & Alerts

* 4‑h stalled‑task alert via `notify-send` and Slack.
* Auto‑disable an agent after two consecutive failures.
* Weekly KPI summary to `#eng‑metrics`.

## 13 · Technical Roadmap (local system)

| Week | Deliverable                    | KPI gate              |
| ---- | ------------------------------ | --------------------- |
| 1    | Conductor polling + `/metrics` | Cycle‑time logging OK |
| 2    | 5 sandbox images               | Build OK              |
| 3    | Grafana dashboard v1           | KPIs visible          |
| 4    | Pilot repo mapping             | Hotspots report       |
| 5    | First 3 agent PRs              | Pass‑rate > 90 %      |
| 6    | Prompt tuning + concurrency 2  | Cycle < 4 h           |
| 7    | Multi‑repo rollout             | Throughput ↑          |
| 8    | Hardening & escalation rules   | Zero stalled tasks    |

## 14 · Open‑Source Strategy

* **Licence**: Apache 2.0
* **Public repo**: `github.com/linosorice/code-crew`
* Structure: `packages/` (conductor, sandbox‑images, plugins), `examples/`, `docs/`
* Public CI: lint & unit tests only; agents do **not** run in Actions.
* Governance: CODEOWNERS, CLA‑bot, semantic‑release.

## 15 · OSS & CLI Roadmap

| Sprint | Deliverable                                           |
| ------ | ----------------------------------------------------- |
| 1      | Public repo with LICENSE, README, docs skeleton       |
| 2      | NPM package `code-crew` v0.1 (`init`, `start`)        |
| 3      | Commands `agent add/rm`, `config set` + plugin schema |
| 4      | `status` command + Prometheus scraper                 |
| 5      | `doctor` command + demo repo `todos‑app`              |
| 6      | Blog post announcement + call for contributors        |

## 16 · Risks & Mitigations

| Risk                   | Prob. | Impact | Mitigation                              |
| ---------------------- | ----- | ------ | --------------------------------------- |
| Laptop RAM saturation  | M     | M      | Concurrency 2, 4 GB Docker limit        |
| Prompt exceeds context | M     | L      | `contextWindow = 90k`                   |
| Breaking NPM deps      | L     | M      | Lockfile maintenance script             |
| OSS support overhead   | M     | L      | Contrib guidelines, `help‑wanted` label |

## 17 · Attachments

**A1** `agent_task.yml` issue template · **A2** `docker-compose.yml` observability · **A3** 4‑h Alertmanager rule (all stored in the `code‑crew` repo).

---

Released under **Apache 2.0**

