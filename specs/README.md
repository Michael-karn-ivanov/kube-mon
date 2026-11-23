# Time API on EKS with C++ Node Telemetry: spec set

This folder replaces the earlier single `SPEC.md`. There is one spec per agent, and each agent can work on its own.

| File | Who reads it | What it is |
|---|---|---|
| `00-overview.md` | **every agent** | Architecture, goals, non-goals, assumptions, workstream map, repo layout |
| `01-contracts.md` | **every agent** | Fixed interfaces: path ownership, names and ports, the HTTP and metrics contracts, labels, build/Helm/Terraform interfaces, versions policy, hand-off format |
| `10-infra.md` | WS-10 agent | Terraform: state bucket, VPC, EKS, node group, ECR, IAM, LB Controller, StorageClass |
| `20-time-api.md` | WS-20 agent | Python service, image, Helm chart (with ServiceMonitor) |
| `30-node-agent.md` | WS-30 agent | C++ agent, image, Helm chart (with PodMonitor) |
| `40-observability.md` | WS-40 agent | kube-prometheus-stack values, rules with tests, the 2 Grafana dashboards, a fake agent for development |
| `50-integration.md` | WS-50 agent | Top-level Makefile, scripts, CI, **deployment runbook**, acceptance tests |

## How to run the agents in parallel

### 0. One-time preparation
```bash
mkdir time-telemetry && cd time-telemetry && git init -b main
unzip ../time-telemetry-specs.zip                    # creates specs/ (or copy the folder in)
git add specs && git commit -m "Add specs"          # the specs must be on main BEFORE you create worktrees
for ws in 10-infra 20-time-api 30-node-agent 40-observability 50-integration; do
  git worktree add "../tt-ws${ws%%-*}" -b "ws/$ws"
done
```
Start each agent **inside its own worktree directory**: `../tt-ws10`, `../tt-ws20` and so on. Never start two agents in the same directory.

Decide in advance:
- **AWS during phase 1:** the recommendation is *no*. WS-10 stops at `terraform plan`.
- **Machine:** WS-20, WS-30, WS-40 and WS-50 need Docker and kind for their local tests, so run them where Docker is available. If they run without Docker, their kind and image checks will be reported as skipped.

### 1. Phase 1: five agents in parallel
Use the launch prompt below, with `<NN>`, `<spec>` and `<phase line>` filled in. `01-contracts.md` §12 keeps the agents from colliding with each other (kind cluster names, ports, Docker, git).

### 2. Review and merge
For each branch:
1. Read its `HANDOFF.md`, especially *Deviations* and *Contract Change Requests*.
2. Optionally run a fresh reviewer agent: "Review branch `ws/NN-…` against `specs/NN-….md` and `specs/01-contracts.md`. Check every Definition of Done item by running the commands yourself. Report gaps. Don't fix anything."
3. Merge into `main`. The paths don't overlap, so the order doesn't matter.

If you accept a Contract Change Request, edit `01-contracts.md` on `main` and tell the affected agents (in their worktrees, after `git merge main`).

### 3. Phase 2: integration
Start one agent on `main` (or a new `ws/50-integration-p2` worktree branched from `main`) with the WS-50 prompt and the phase 2 line. Allow AWS only when you're ready to spend money; the agent then runs `make up acceptance destroy`.

## Launch prompt

```text
You are one of five agents building this system in parallel. You implement workstream WS-<NN>.

Read, in this order:
  specs/README.md, specs/00-overview.md, specs/01-contracts.md  - global goal, components, fixed interfaces
  specs/<spec>                                                  - YOUR task

Rules:
- Create or edit files ONLY under the paths your spec says you own (01-contracts.md §1).
- Follow the parallel-work rules in 01-contracts.md §12 exactly (your own git branch, kind cluster tt-ws<NN>,
  local ports 1<NN>00-1<NN>99, no destructive Docker commands, no sudo).
- 01-contracts.md is fixed. If it blocks you, record a Contract Change Request in HANDOFF.md and carry on.
- Pin every dependency to the latest stable version and record the pins in your VERSIONS.md.
- Don't stop to ask questions: choose the option most consistent with the specs and record it under Deviations.
- Commit to your branch at meaningful milestones. Never push or merge.
- <phase line>
- You are done when every Definition of Done item in your spec is met (or explicitly marked as skipped with a reason)
  and HANDOFF.md is written. Finish with a short summary of the results and anything I need to decide.
```

Phase lines:
- WS-10: `Do NOT run terraform apply or create AWS resources. Go as far as fmt/validate/tflint/trivy, and terraform plan if credentials are present.`
- WS-20, WS-30, WS-40: `Do not create AWS resources. Local Docker/kind testing is expected.`
- WS-50, phase 1: `You are in PHASE 1 (scaffolding) as defined in 50-integration.md §8. The other components do not exist yet. Do not touch AWS.`
- WS-50, phase 2: `You are in PHASE 2 as defined in 50-integration.md §8. WS-10..40 are merged into main. AWS use is <ALLOWED, account/profile X | NOT allowed>.`

## Where to find the deployment instructions
`50-integration.md` §5 is the operator runbook: prerequisites, `deploy.env`, first deployment with a check after each step, access, day-2 operations, local kind, teardown and troubleshooting. WS-50 also turns it into the quick start in the root `README.md`.
