# RepoOps P0-02 AgentTeams Capability Verification

This document records the reusable AgentTeams evidence for RepoOps issue
[#4](https://github.com/goai-hq/repo-ops/issues/4). It is a capability
verification, not a RepoOps integration or a change to workflow authority.

## Evidence Matrix

| Need | Evidence | Result / boundary |
| --- | --- | --- |
| Worker lifecycle and stable identity | `agentteams-controller/api/v1beta1/types.go:34-38` defines the stable Edge UUID label; `:183,334-341` define `WorkerName` and `EffectiveWorkerName`; `:375-396` defines the status projection. | Worker CR lifecycle is available and keeps a stable runtime identity key. RepoOps must still keep its own WorkItem, Dispatch, and NodeExecution identities. |
| Team, room, and member stable identifiers | `agentteams-controller/api/v1beta1/types.go:477-492,512-553` define `TeamRoomID`, `LeaderDMRoomID`, and per-member `RoomID`/`MatrixUserID`; `tests/test-18-team-config-verify.sh:143-235` waits for `Team.status.phase=Active` and verifies leader/worker room ownership. | Team/Room/member topology is available as durable controller state and can be bound by RepoOps as collaboration facts. |
| Project, DAG, Task, and event identifiers | `tests/test-21-team-project-dag.sh:449-618` retries `taskflow delegate_task` and verifies one stable `notification.eventId`, one assignment event, and exact Matrix `event_id` reuse; `tests/test-26-qwenpaw-teamharness-plugin-mode.sh:1343-1439` verifies stable `projectId`, `taskId`, `spec.md`, `result.md`, and deliverable paths. | AgentTeams provides stable Project/Task IDs plus deterministic assignment-event identity. RepoOps can treat these as bound collaboration IDs, not workflow authority. |
| Status source split: reconcile vs poll vs event | `docs/design/teamharness/boundary-and-contracts.md:24-35,47-62` excludes CR polling and worker lifecycle from TeamHarness; `docs/design/member-runtime-config-contract.md:22-29` assigns `runtime.yaml` polling to QwenPaw workers; `tests/test-17-worker-config-verify.sh:114-131,317-356` and `tests/test-26-qwenpaw-teamharness-plugin-mode.sh:831-897,1195-1219` verify controller projection and hot-update polling. | Controller-owned CR/status reconciliation and runtime-owned desired-state polling are explicitly separate. RepoOps must not infer success from one channel alone; it must reconcile event, task, and projected-state views. |
| Recovery and authoritative re-read | `docs/design/teamharness/project-task-runtime-design.md:229-267` defines `check_task` plus explicit `accept_task_result`, and `:273-281` defines the fixed event-resume order; `plugins/tests/teamharness/test_pull_project.py:261-400` proves mutating `taskflow`/`projectflow` actions pull authoritative shared state before acting and fail retryably when that pull fails; `plugins/teamharness/adapters/qwenpaw/task_trace.py:490-520` re-reads `meta.json` on every entry span so submit/cancel state stays authoritative. | Recovery is based on re-reading authoritative shared state, not trusting stale local memory or one-off tool return values. RepoOps can safely model uncertain collaboration state as waiting/reconcile-needed instead of completion. |
| Candidate result boundary | `tests/test-26-qwenpaw-teamharness-plugin-mode.sh:1433-1439` verifies `taskflow check_task` on a submitted worker result; `docs/design/teamharness/project-task-runtime-design.md:229-240` requires `accept_task_result` as the only project-advance step and keeps `check_task` read-only. | AgentTeams can produce and verify a candidate task result, but result acceptance is a separate explicit action. RepoOps remains the only authority that can promote a candidate into workflow/node completion. |

## Minimal Reproducible PoC

Lightweight repo-local validation, no embedded stack required:

```bash
python3 -m pytest plugins/tests/teamharness/test_trace.py \
  plugins/tests/teamharness/test_pull_project.py -q
ruby plugins/tests/teamharness/test-contracts.rb
ruby plugins/tests/teamharness/mcp/tools/test-projectflow.rb
ruby plugins/tests/teamharness/mcp/tools/test-taskflow.rb
```

These prove the TeamHarness contract, deterministic `delegate_task` behavior,
authoritative pull-before-mutate recovery guard, explicit
`check_task`/`accept_task_result` split, and task-trace re-read behavior.

Full-stack embedded validation, against a prepared AgentTeams environment:

```bash
bash tests/test-18-team-config-verify.sh
bash tests/test-21-team-project-dag.sh
bash tests/test-26-qwenpaw-teamharness-plugin-mode.sh
```

The first test creates temporary Worker and Team CRs, verifies Team/Room/role
artifacts and reconciler state, then removes its resources in its cleanup trap.
`test-21` adds the deterministic assignment-event contract for TeamHarness
project/task delegation. `test-26` creates a real QwenPaw TeamHarness team and
verifies task delegation, result storage, deliverables, and leader-side
`check_task`.

Capture the test logs, source revision, created CR names, Team/Room IDs,
Project/Task IDs, Matrix `event_id`, and resulting storage paths as the P0-02
evidence record.

## Observed Handoff Sequence

1. RepoOps binds its own Dispatch/NodeExecution to AgentTeams collaboration IDs.
2. AgentTeams controller reconciles Worker and Team CRs, then projects
   non-secret team/member/runtime facts into `runtime.yaml`.
3. Runtime workers poll `runtime.yaml`; TeamHarness tools create Project/DAG/Task
   state and emit deterministic assignment events.
4. Worker task completion produces `result.md`, deliverables, and a submitted
   task state. Leader-side `check_task` verifies that candidate result.
5. Only an explicit acceptance step may advance AgentTeams project state, and
   only RepoOps may advance RepoOps workflow/node state.

## RepoOps Boundary

These capabilities do not establish a RepoOps integration. The following are
explicitly deferred to RepoOps #9 (P1-03): `dispatchId`, role-slot mapping,
WorkerTemplate/manifest digest propagation, NodeDispatch/NodeResult schemas,
and the rule that task completion produces only a candidate result. RepoOps
alone decides whether a NodeExecution or WorkItem advances; no AgentTeams task
completion may advance it automatically.

## Acceptance Checklist

- [ ] `test-18` records a created Team with exactly one leader and expected Room/role facts.
- [ ] `test-21` or the repo-local TeamHarness MCP tests record deterministic Project/Task assignment identity and event reuse.
- [ ] `test-26` or the repo-local TeamHarness MCP tests record Project/Task delegation, a candidate result, and a deliverable.
- [ ] Evidence includes stable Worker/Team/Room/Project/Task/event identifiers and source revision.
- [ ] Missing or uncertain AgentTeams facts are reconciled or reported as waiting, never treated as RepoOps success.
- [ ] Candidate task completion is recorded as candidate-only and does not by itself advance RepoOps workflow state.
- [ ] Any required AgentTeams extension is proposed and reviewed through a fork PR only.

## Evidence Capture Template

```text
AgentTeams revision:
Test command and exit code:
Worker CR names / stable IDs:
Team and Room IDs:
Project / DAG / Task IDs:
Matrix assignment event IDs:
Result and artifact locations:
Observed reconciliation or polling signal:
RepoOps mapping decision (candidate only):
```
