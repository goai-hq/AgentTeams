# RepoOps P0-02 AgentTeams Capability Verification

This document records the reusable AgentTeams evidence for RepoOps issue
[#4](https://github.com/goai-hq/repo-ops/issues/4). It is a capability
verification, not a RepoOps integration or a change to workflow authority.

## Evidence Matrix

| Need | Evidence | Result / boundary |
| --- | --- | --- |
| Worker lifecycle and stable identity | `agentteams-controller/api/v1beta1/types.go:170-206` defines `Worker`, its desired lifecycle state, and reconciliation toward actual state; `:34-38` defines the stable Edge UUID label. | Worker CR lifecycle is available. RepoOps must retain its own WorkItem and dispatch identity. |
| Team member role and leader boundary | `tests/test-18-team-config-verify.sh:1-7` verifies three Worker CRs, one `team_leader`, Team Room status, role APIs, and controller reconciliation; `:84-140` applies the Team/Worker CRs. | Team/Room topology and leader/worker roles are available. |
| Runtime facts and recovery input | `docs/design/member-runtime-config-contract.md:22-29` defines controller-written, worker-polled non-secret desired state; `:48-69` defines team/member facts. | Runtime config is a projected AgentTeams fact, not the RepoOps source of truth. |
| Project, task delegation, result evidence | `docs/design/teamharness/boundary-and-contracts.md:14-22` assigns project/task MCP tools to TeamHarness; `tests/test-26-qwenpaw-teamharness-plugin-mode.sh:1247-1430` performs real Team work and asserts task specification, result, and deliverable storage. | Project/task collaboration and candidate results are available. |
| Events, polling, reconciliation | `docs/design/teamharness/boundary-and-contracts.md:24-35,47-62` excludes CR polling and worker lifecycle from TeamHarness; `docs/design/member-runtime-config-contract.md:22-29` assigns config polling to QwenPaw workers. | AgentTeams reconciliation and worker polling must be independently reconciled by RepoOps. |

## Minimal Reproducible PoC

Run against a prepared embedded AgentTeams stack:

```bash
bash tests/test-18-team-config-verify.sh
bash tests/test-26-qwenpaw-teamharness-plugin-mode.sh
```

The first test creates temporary Worker and Team CRs, verifies Team/Room/role
artifacts and reconciler state, then removes its resources in its cleanup trap.
The second test creates a real QwenPaw TeamHarness team and verifies task
delegation, result storage, and deliverables. Capture the test logs, created
CR names, Team/Room IDs, Project/Task IDs, and resulting storage paths as the
P0-02 evidence record.

## RepoOps Boundary

These capabilities do not establish a RepoOps integration. The following are
explicitly deferred to RepoOps #9 (P1-03): `dispatchId`, role-slot mapping,
WorkerTemplate/manifest digest propagation, NodeDispatch/NodeResult schemas,
and the rule that task completion produces only a candidate result. RepoOps
alone decides whether a NodeExecution or WorkItem advances; no AgentTeams task
completion may advance it automatically.

## Acceptance Checklist

- [ ] `test-18` records a created Team with exactly one leader and expected Room/role facts.
- [ ] `test-26` records Project/Task delegation, a result, and a deliverable.
- [ ] Evidence includes stable Worker/Team/Room/Project/Task identifiers and source revision.
- [ ] Missing or uncertain AgentTeams facts are reconciled or reported as waiting, never treated as RepoOps success.
- [ ] Any required AgentTeams extension is proposed and reviewed through a fork PR only.

## Evidence Capture Template

```text
AgentTeams revision:
Test command and exit code:
Worker CR names / stable IDs:
Team and Room IDs:
Project / DAG / Task IDs:
Result and artifact locations:
Observed reconciliation or polling signal:
RepoOps mapping decision (candidate only):
```
