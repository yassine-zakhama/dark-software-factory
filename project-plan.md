# Dark Software Factory — Demo Project Plan

## 1. Goal

Build a demo for the **AI-Garage** showcase (Cloud Eng. pillar) that illustrates a "dark software factory": an autonomous AI pipeline where an incoming requirement moves through stations (spec → build → test → deploy) with minimal human intervention, visualized live in a dashboard.

The AI-Garage listing itself (content + link) is trivial and comes last — the demo is the actual deliverable.

## 2. Open questions to resolve with stakeholders (before or during Phase 0)

| # | Question |
|---|---|
| 1 | Is the IT Asset Management Tool confirmed as the target system, or should the demo stay fully stubbed? |
| 2 | Is there an existing AWS account/sandbox for this, or does one need to be requested? |
| 3 | Which credential covers LLM calls — the mentioned "AI license" (direct Anthropic API) or AWS Bedrock? Cost/billing implications differ. |
| 4 | Is "AWS or something" a hard requirement, or is any reachable hosting acceptable for a first demo? |
| 5 | What exactly triggers a requirement — a Jira ticket, a tagged ticket, a manual form submission? |
| 6 | Is a real integration into the Asset Management Tool expected, or is a simulated "would deploy X" sufficient for this first step? |

Assumption used for this plan until answered: **fully stubbed target system**, **Jira ticket as trigger**, **direct Anthropic API via the "AI license"**, **AWS Lambda + DynamoDB + React on S3/CloudFront** for hosting.

## 3. Architecture summary

**Design principle: the factory is a generic, pluggable engine — not tied to one AI provider, one target system, or one trigger source.** This is deliberate: the demo is meant to showcase a *pattern* for the AI-Garage Cloud Eng. pillar, not a one-off integration. Concretely, the engine is built against a small set of interfaces, each with exactly one real implementation for the demo plus a mock/in-memory implementation for local dev and safe fallback:

| Interface | Demo implementation | Also provides |
|---|---|---|
| `Reasoner` (AI provider) | Claude via `anthropic-sdk-go` | Swappable to another provider later without touching station logic |
| `TargetConnector` (system being extended) | `MockConnector` (logs intended action) | Asset Management Tool becomes a second implementation once confirmed — no redesign needed |
| `TriggerSource` (requirement entry point) | Jira webhook | Could add Slack/manual-form/GitHub-issue sources later with no engine change |
| `Store` (state persistence) | DynamoDB | In-memory implementation for local testing without AWS |
| Pipeline/station sequence | Data-driven config (`spec → build → test → deploy`) rather than hardcoded | Same engine could demo a different sequence live, on request |

- **Trigger**: `TriggerSource` (Jira webhook) → API Gateway → Lambda creates a requirement record.
- **Station agents** (Lambda, Go): spec, build, test, deploy — each depends on the `Reasoner` interface (not a specific SDK) where an LLM call is needed, writes its result, and advances `current_station` in the `Store`.
- **State store**: `Store` interface, backed by DynamoDB for the deployed demo — one item per requirement, holding `id`, `current_station`, `status`, `history`, `timestamps`.
- **API layer**: Lambda (REST) + WebSocket API Gateway for live push to the frontend.
- **Frontend**: React dashboard (S3 + CloudFront) — board/pipeline view showing requirements moving through stations in real time.
- **Deploy station**: uses `TargetConnector`, defaulting to `MockConnector` — logs the intended action rather than calling a real system, pending Q1/Q6 above. Swapping in a real Asset Management Tool connector later is a new implementation, not a pipeline change.

**Explicitly out of scope for this demo** (mentioned above as risk/future work, not built now): abstracting away AWS-specific hosting (Lambda/DynamoDB/EventBridge), abstracting the station hand-off mechanism, and a policy/approval engine for autonomy levels. These matter for a production dark factory but add cost with no payoff for a first-step demo.

## 4. Repo structure

Single monorepo, GitHub-hosted. Full layout:

```
dark-software-factory/
├── cmd/                          # one main package per deployable Lambda
│   ├── trigger-listener/
│   │   └── main.go
│   ├── spec-agent/
│   │   └── main.go
│   ├── build-agent/
│   │   └── main.go
│   ├── test-agent/
│   │   └── main.go
│   ├── deploy-agent/
│   │   └── main.go
│   └── api/
│       └── main.go               # REST + WebSocket handler
│
├── internal/
│   ├── pipeline/
│   │   ├── types.go              # Requirement, Station, StatusUpdate
│   │   ├── sequence.go           # data-driven station sequence config
│   │   └── sequence_test.go
│   ├── reasoner/
│   │   ├── reasoner.go           # Reasoner interface
│   │   ├── claude.go             # Claude implementation (anthropic-sdk-go)
│   │   └── claude_test.go
│   ├── connector/
│   │   ├── connector.go          # TargetConnector interface
│   │   ├── mock.go                # MockConnector implementation
│   │   └── mock_test.go
│   ├── trigger/
│   │   ├── trigger.go            # TriggerSource interface
│   │   ├── jira.go               # Jira webhook implementation
│   │   └── jira_test.go
│   └── store/
│       ├── store.go              # Store interface
│       ├── dynamodb.go           # DynamoDB implementation
│       ├── memory.go             # in-memory implementation (local dev/tests)
│       └── store_test.go
│
├── frontend/
│   ├── src/
│   │   ├── components/           # dashboard/board components
│   │   ├── hooks/                # e.g. useRequirementFeed (WebSocket)
│   │   ├── types.ts              # mirrors Go pipeline types
│   │   └── App.tsx
│   ├── public/
│   ├── package.json
│   └── tsconfig.json
│
├── infra/                        # IaC (SAM/CDK/Terraform — TBD)
│   ├── template.yaml             # or main.tf / cdk app, depending on choice
│   └── env/ 
│       ├── dev.yaml
│       └── demo.yaml
│
├── .github/
│   └── workflows/
│       ├── ci.yml                # go build/test/lint + frontend build/test
│       └── deploy.yml            # deploy to AWS on merge to main (or manual trigger)
│
├── docs/
│   └── architecture.md           # links back to this plan / diagrams
│
├── go.mod
├── go.sum
├── Makefile                      # common tasks: build, test, run-local, deploy
├── .golangci.yml                 # lint config
├── .gitignore
└── README.md
```

**Notes on the layout:**
- `internal/` is intentionally used (not a top-level `pkg/`) since none of this is meant to be imported by other repos — it's a self-contained demo engine.
- Each interface package (`reasoner`, `connector`, `trigger`, `store`) holds the interface **and** its implementations together, so `import "…/internal/store"` gives you both the contract and the concrete types — common Go convention, avoids interface-only packages that add indirection for no benefit at this scale.
- `cmd/*/main.go` stays thin — just wiring (pick implementations, construct dependencies, hand off to a Lambda handler). All real logic lives in `internal/`, which keeps the station agents testable without spinning up Lambda.
- `Makefile` targets worth having from day one: `make test`, `make lint`, `make run-local` (spins up station agents against in-memory `Store` + a fake `Reasoner` for fast iteration), `make deploy`.
- CI (`ci.yml`) should run on every PR: `go vet`, `golangci-lint`, `go test ./...`, and a frontend `npm run build` + `npm test`. Keep `deploy.yml` separate and either manually triggered or gated to `main`, so you're not deploying to AWS on every branch push during early development.
- `docs/architecture.md` is a good place to drop the pipeline diagram once you've settled on final station names — keeps the repo self-documenting for anyone who stumbles on it via the AI-Garage link.

## 5. Phases

### Phase 0 — Clarify & scope (0.5–1 week)
- Get answers to open questions above.
- Confirm AWS account access and Anthropic API key availability.
- Lock the list of stations for v1 (recommend: trigger → spec → build → test → deploy, 5 stations, matches the diagram already discussed).

### Phase 1 — Backend skeleton (1–1.5 weeks)
- Define shared types (`Requirement`, `Station`, `StatusUpdate`) in `/internal/pipeline`, plus the data-driven station sequence config.
- Define the four core interfaces before writing any concrete implementation: `Reasoner`, `TargetConnector`, `TriggerSource`, `Store`.
- Implement one real implementation of each: Claude `Reasoner` (`anthropic-sdk-go`), `MockConnector`, Jira `TriggerSource`, DynamoDB `Store` — plus an in-memory `Store` for local dev/tests.
- Implement one station agent end-to-end (e.g. spec agent) against the `Reasoner`/`Store` interfaces, as a template for the rest.
- Local testing against in-memory `Store` + mock `Reasoner` — no AWS or live API dependency required yet.

### Phase 2 — Remaining stations + orchestration (1 week)
- Implement build, test, and deploy (stubbed) agents following the Phase 1 template.
- Decide and implement station hand-off mechanism (direct Lambda invoke, SQS, or EventBridge — EventBridge recommended for loose coupling).
- Implement API layer (REST for initial state fetch, WebSocket for live updates).

### Phase 3 — Frontend dashboard (1 week, can overlap with Phase 2)
- React app: pipeline/board view showing requirements and their current station.
- WebSocket (or polling fallback) integration to the API layer.
- Basic animation/transition when a requirement moves stations (the "wow" factor for the showcase).

### Phase 4 — AWS deployment (0.5–1 week)
- IaC for Lambdas, API Gateway (REST + WebSocket), DynamoDB, S3/CloudFront.
- Wire environment variables/secrets (Anthropic API key) via AWS Secrets Manager or SSM Parameter Store.
- End-to-end test: real Jira ticket → visible progression on deployed dashboard.

### Phase 5 — Polish & demo readiness (0.5 week)
- Seed a few realistic-looking requirements for a smooth live demo (don't rely solely on a live Jira ticket during the actual showcase).
- Write short demo script / talking points.
- Record a fallback video walkthrough in case live demo fails during the showcase.

### Phase 6 — AI-Garage integration
- Write short content blurb for the Cloud Eng. pillar.
- Add link to the deployed demo.

## 6. Rough timeline

~4.5–5.5 weeks total, assuming no major blockers on AWS access or the Asset Management Tool decision. Phases 2 and 3 can run in parallel if more than one person is available.

## 7. Risks

- **Target system integration undecided** — no longer a blocker: the `TargetConnector` interface means the demo ships with `MockConnector` regardless, and a real Asset Management Tool connector can be added later as a second implementation with no pipeline redesign.
- **AI provider lock-in** — mitigated the same way via the `Reasoner` interface; switching providers later is a new implementation, not a rewrite.
- **AWS access/setup delays** — mitigate by developing and testing locally (SAM local, DynamoDB Local) before needing real AWS access.
- **LLM output reliability for build/test stations** — for a demo, favor deterministic/narrow scenarios over open-ended code generation to avoid unpredictable failures during a live showcase.
- **Scope creep from "real" integration ambitions** — keep Phase 0–5 focused on the demo; treat any real Asset Management Tool integration as a distinct follow-up project.