# Drone Delivery Simulation Codex Implementation Guide

Version 1.0 | 23 October 2026 | Project owner Makomane Tau

## 1 Mission for Codex

Implement a local drone delivery simulation using PX4 SITL, Gazebo, Python, and a browser interface. Start with one vehicle, a generic flat world, two pads 100 m apart, a 250 g physically modelled package, operator approval, outbound flight, landing and disarming, simulated handover, and an explicitly approved return flight. The target developer machine is Windows with 16 GB RAM, using Ubuntu through WSL2.

Read `AGENTS.md` and `docs/SOFTWARE_DESIGN.md` before changing code. The full design defines interfaces, state ownership, coordinate frames, requirements FR01 through FR13, and tests T01 through T17. This guide defines build order and evidence. The Markdown files are the source for future edits; the Word document is a review copy of the full design.

This package contains design documents, not a working simulator or application. Create the implementation in a suitable repository. If the destination already contains code or instructions, inspect them first and preserve unrelated work. Do not initialize a second project over existing source without understanding it.

## 2 Implementation constraints

- One active run and vehicle. Use sequential batch experiments.
- Keep flight control in PX4; use uploaded missions for the first release.
- Use a modular FastAPI API plus one separate durable Python gateway. PostgreSQL stores business data; SQLite journals live execution locally.
- Use React and TypeScript with a simple SVG site view. Gazebo provides the 3D scene.
- Keep raw SDK types inside adapters. Pure domain code must remain simulator independent.
- Never connect to hardware, scan serial devices, expose unrestricted MAVLink, or enable remote flight endpoints.
- Clearly identify MOCK, SITL, and REPLAY modes in logs, reports, and UI.
- Never report SITL as tested if only fake adapter tests ran.
- Introduce ROS 2, a message broker, cloud hosting, perception, and fleets only through a later justified design change.
- Treat all numeric limits as versioned experiment settings. Record approved changes with their reasons.

## 3 First response and repository inspection

Inspect the directory, git status, existing `AGENTS.md`, manifests, and runtime configuration. Identify available OS, Python, Node, WSL, graphics, disk, and memory information. Report existing work and gaps briefly. Do not print secrets or install dependencies just to inspect the environment.

If running on a Linux coding host rather than the owner's Windows laptop, distinguish what can be built there from what still needs laptop verification. Do not claim that creating a WSL setup guide installed WSL on the user's computer. If the simulator is unavailable, proceed with permitted domain work and make the missing integration evidence explicit.

Create `docs/IMPLEMENTATION_STATUS.md` with milestone status, evidence paths, remaining blockers, and next step. Initial statuses are NOT_STARTED, IN_PROGRESS, BLOCKED, and VERIFIED. A checklist tick requires actual evidence. Keep `docs/DECISIONS.md` as the ADR index.

## 4 Required repository skeleton

Create these directories as they become necessary, not as empty filler: `apps/api`, `apps/gateway`, `apps/web`, `packages/domain`, `packages/adapters`, `sim/worlds`, `sim/models`, `sim/plugins`, `config`, `scenarios`, `scripts`, `tests/unit`, `tests/integration`, `tests/sitl`, `tests/e2e`, `docs`, and ignored `runs`.

Prefer one Python workspace and one browser package. Use a Python lockfile, exact runtime files, and a JavaScript lockfile. Choose package tooling compatible with the host and document it. Keep PX4 in a pinned external checkout or submodule; do not copy its full build tree into the project. Preserve third party licenses.

Implement these public project targets by M7. Their spelling is a contract; the underlying implementation may be Python scripts or Make targets.

| Target | Required behavior |
|---|---|
| make doctor | Read only diagnostics with actionable missing dependency messages |
| make setup | Prepare project dependencies from locks and explain privileged prerequisites |
| make sim | Start the pinned simulator and selected scenario assets |
| make gateway | Start the single vehicle executive with its journal |
| make api | Start API and configured local dispatch worker |
| make web | Start browser development server |
| make test | Run non SITL checks and exit nonzero on failure |
| make test-sitl SCENARIO=calm_delivery | Execute actual PX4 integration and save run evidence |
| make evaluate RUN_ID=... | Calculate metrics from an existing run |
| make export-run RUN_ID=... | Package manifest and referenced artifacts |
| make stop | Stop only processes owned by the selected run |

Before a target exists, do not tell the user to run it as though it were implemented. Startup must never arm automatically. A simulator start and a mission dispatch are separate operations.

## 5 M0 Environment and integration spike

**Goal:** prove the selected flight toolchain before building platform features.

1. Add read only diagnostics and `docs/SETUP_WINDOWS_WSL.md`. Capture CPU, graphics, RAM, SSD space, Windows, WSL, Linux, and actual rendering support where observable. Start with one lightweight world and modest build parallelism.
2. Check the official release matching PX4, Gazebo, and SDK instructions. Candidate baseline is Ubuntu 24.04, Gazebo Harmonic, a stable PX4 release in the 1.17 documentation family, and Python 3.12. Exact versions become fixed only after the spike works.
3. Start the stock X500 using the upstream command `make px4_sitl gz_x500`. Validate a headless launch separately. Save logs and actual versions.
4. Connect QGroundControl and verify simulated identity and telemetry. Then prove programmatic telemetry, arm, takeoff, observed hover, landing, and observed disarming with the selected MAVSDK API generation.
5. Prove upload and execution of a mission with explicit landing semantics using the chosen SDK. Verify readback or normalized mission fingerprint reconciliation. Resolve MissionRaw availability before writing executive code around it.
6. Test a separate flight from B back to A with rearming. Record home before and after each arm and the observed failsafe target. This is the early check for misleading return to home assumptions.
7. Resolve the geodetic world origin, ENU and NED conversion, mission altitude datum, actual UDP ports, expected system IDs, and simulator namespaces. Add small numeric conversion fixtures.
8. Commit `toolchain.lock.yaml` with exact versions, SHAs, API generation, model hashes, parameter export reference, and official reference URLs. Record capabilities such as detach observation and failure injection as tested, unsupported, or pending.

**Exit:** real simulator logs and an evidence record for takeoff, land, mission upload, rearm, and home behavior. If any necessary feature fails, fix it or record a justified toolchain decision before marking M0 VERIFIED. A screen capture alone is insufficient evidence of command outcome handling.

## 6 M1 Domain model and planning

**Goal:** implement behavior that can be verified without a simulator.

Create typed values for mass, dimensions, position frames, altitude datums, velocity, identities, timestamps, and plan revisions. Reject nonfinite numeric values and unknown configuration fields. Add immutable world, route, vehicle, payload, approval, mission, leg, and command schemas.

Implement route checking against expanded obstacle geometry, bounds, pads, and payload limits. Validate outbound, descent, and return segments. Compute a canonical plan hash from resolved configuration. Approval covers that hash, specific action, operator, and expiry. A changed plan invalidates approval.

Implement the state machine from full design section 10 as a pure transition reducer. Each transition has explicit guards and produces events. Orthogonal confidence does not mutate assumed aircraft location. Terminal outcomes cannot be overwritten by a late duplicate event. Add a fake clock, fake flight adapter, and fake payload adapter.

**Exit tests:** FR01, FR02, FR03, FR11, and frame checks through T01, T02, T03, T04, T12, T15. Include negative cases for stale approvals, NaN, unknown units, invalid transitions, route collisions, and concurrent vehicle reservations at the persistence layer when available. Unit tests should check behavior, not copy reducer implementation line by line.

## 7 M2 Durable mission gateway

**Goal:** fly both legs with a durable, reconcilable execution owner.

Create `FlightAdapter`, `PayloadAdapter`, `Clock`, `MissionJournal`, and `EventSink` protocols. The flight interface exposes connection and identity, observation subscription, mission upload and inspection, arm and start, supported recovery requests, and disconnect. SDK exceptions become typed domain results; they must not leak into API behavior.

Implement a single writer gateway process with SQLite WAL transactions and a process lock. Persist each intent before actuation. Keep command receipt, transport acknowledgement, and observed completion separate. Serialize commands per vehicle. Track source boot and run identity, sequence events, and persist checkpoints.

Add explicit command deadlines, telemetry freshness, observed landing and disarming guards, and separate return approval. Do not use arbitrary sleeps as evidence of landing. Dwell timers may wait only while validating a continuing observed condition.

On restart, monitor and reconcile actual state with stored mission and command outcomes. Unknown actions remain blocked until a recorded resolution; never blindly replay arm, start, or payload operations. API outages must not halt monitoring of an active uploaded leg. An expired command cannot become executable when the network returns.

**Exit tests:** T05 without physical package, T07 using a disconnected event sink, and T08 across crash boundaries. Preserve actual PX4 logs for two flight legs. Document the gateway crash outcome under the configured PX4 failsafe with and without QGroundControl heartbeats.

## 8 M3 World and physical payload

**Goal:** add the delivery experiment with measurable physical effects.

Build the baseline world and model revisions from full design section 4. Include collision geometry, pads, declared origin, fence, ground contact evidence, and obstacle definitions shared by validation and evaluation. Keep visual detail low.

Attach the separate 0.25 kg box with verified dimensions, inertia, mount clearance, and no overlapping initial collision shapes. Use stock DetachableJoint behavior where sufficient. Add a minimal plugin only if direct status observation is missing. Verify total model mass and the centre of mass contribution. Keep the original aircraft dynamics configuration and record each changed parameter.

Implement simulated removal only when landed and disarmed. Persist a handover request, confirm joint detachment, and clear the detached package without resetting the aircraft. A jam scenario keeps it attached. Duplicate handovers query or return the existing outcome. Record the transition from loaded to empty aircraft assembly.

Create an evaluator that reads ground truth separately from estimated flight telemetry. Produce collision, landing error, minimum swept path clearance, and estimated energy fields. Introduce a transparent load dependent energy estimate with versioned coefficients and validity limits; label it uncalibrated until measured calibration exists. Do not substitute changing a battery percentage for modelling payload mass.

**Exit tests:** T05, T06, T12, T15, T16 with physical payload and stored truth. Include a loaded and empty comparison that reports physical mass and measured outputs without claiming hardware endurance.

## 9 M4 API and persistent delivery platform

**Goal:** turn the local executive into an authorized delivery workflow.

Implement database migrations from full design section 15, local identities, requester ownership, operator roles, strict input schemas, mission revisions, approval expiry, vehicle reservation, unique keys, command records, and event projections. Keep client input free of arbitrary flight instructions.

Implement the public and gateway contracts in full design sections 16 and 17. Publish generated OpenAPI. Use one API transaction for mission intent, state version, reservation, and dispatch outbox. The worker delivers at least once with lease renewal. The gateway deduplicates by stable command ID and body hash. Expired worker leases do not authorize duplicate physical actions.

Replay gateway events by durable per mission sequence. Store unique event IDs and acknowledgements. Display projection lag and confidence. Return 202 for accepted asynchronous requests and provide a command status resource; never label a receipt as completed delivery.

**Exit tests:** T01 through T04, T07 through T10, T13, and T17 contracts. Use a real test PostgreSQL instance for unique constraint, migration, transaction, and concurrency checks. Test wrong requester access, stale versions, changed payload approvals, DB restart, duplicate event replay, and a timeout with an unknown action outcome.

## 10 M5 Browser interfaces

**Goal:** demonstrate request to return through an understandable interface.

Build the request form, mission queue, mission detail, SVG site view, telemetry and alerts, and run results. Generate types from OpenAPI. Show units, altitude reference, observed mode, arm status, payload attachment, stale telemetry, and approval validity. Keep command receipt and observed result visually separate.

Requester accounts can inspect their own orders. Operator controls approve and start outbound, confirm removal after disarm, approve and start return, cancel where permitted, and request valid phase dependent recovery. Disable unavailable controls with a reason. Do not create a public emergency motor cut action.

WebSocket reconnect fetches snapshot and durable events after the saved cursor. Bound queues and downsample telemetry. A disconnected browser does not pause the simulator. Site map rendering uses estimated positions; any optional truth overlay is explicitly labelled EVALUATION TRUTH and read only.

**Exit tests:** one complete mock mode Playwright flow for fast regression plus an actual SITL browser demonstration. Test unauthorized controls, expiry, stale status, recovery choice, dropped connection, and result retrieval. Preserve screenshots as internal QA or explicitly requested evidence, not as substitutes for telemetry logs.

## 11 M6 Fault suite and realism

Implement versioned scenarios for calm delivery, low energy before departure, battery fault in flight, gateway link loss, API outage, gateway restart, blocked pad before launch, blocked pad in flight, payload jam, and uniform wind. Add GNSS degradation only when effective injection is demonstrated by the selected build; otherwise mark it SKIPPED with the capability blocker.

Each scenario declares expected aircraft policy and expected software state independently. Verify the fault reaches its target. Dropped browser updates cannot count as a GNSS or aircraft link fault. Test with QGroundControl both absent and present where heartbeat behavior matters. Record actual recovery destination after rearming at B.

Use simulator time for fault schedules and flight metrics, monotonic time for live process deadlines, and UTC for audit. Record world pauses and resets. No silent reclassification of unsupported fault cases as passed.

**Exit tests:** T09 through T11, T14, T16, T17 and all supported fault outcomes. Every result includes injection evidence, event sequence, aircraft observations, model limitations, and whether recovery completed or the run was terminated.

## 12 M7 Release and reproducibility

Implement the complete command contract, setup and troubleshooting guide, example environment configuration, scenario catalog, artifact export, bounded retention, and acceptance report. Keep secrets, local journals, PX4 builds, and run artifacts outside git. Test a fresh project setup from pinned dependencies where the host permits it.

Run the proposed gates from full design section 22: 10 calm deliveries, landing tolerance, zero nominal collisions and fence breaches, obstacle clearance, no duplicate logical actions, supported fault behavior, resource observations, and artifact completeness. Attach failures and limitations. A successful ten run gate is a regression result, not a quantified safety or hardware reliability claim.

**Exit:** a tagged simulation release or reviewable implementation revision with exact commit, manifests, test output, run report links, remaining unverified items, and commands another developer can follow. Never mark a gate complete based on a design assumption.

## 13 Contract examples to implement

An order request uses the following shape. Example IDs are placeholders in this guide and must be replaced with IDs returned by the running application.

```json
{
  "route_revision_id": "<published-route-revision-id>",
  "payload": {
    "mass_kg": 0.25,
    "dimensions_m": {"length": 0.15, "width": 0.10, "height": 0.08}
  },
  "test_reference": "calm-baseline"
}
```

A command observation envelope includes `command_id`, `mission_id`, `status`, `receipt_at_utc`, `acknowledged_at_utc`, `observed_at_utc`, `outcome_code`, and `last_event_sequence`. Null observations are meaningful; do not fill them with receipt time. Use the same idempotency key on a network retry of the same logical operation.

The scenario contract minimally contains `schema_version`, `name`, `mode`, `seed`, `world_revision`, `vehicle_revision`, `payload`, `policy_revision`, `faults`, `expected_outcomes`, and `timeouts`. Resolve revisions to hashes in the run manifest. This is a schema specification; complete runnable YAML belongs to M1 and M6.

## 14 Required Codex completion report

After each meaningful implementation slice, state what now works, why the change was needed, tests actually run and their results, tests not run and their blocker, files or evidence to review, and the next unverified gate. Keep the report concise and factual. Continue within the authorized milestone when practical; do not ask routine architecture questions already answered here.

If a design conflict arises, make useful independent progress, record an ADR for material changes, and ask only when the missing decision prevents correct work. Do not weaken the no hardware boundary, duplicate action protections, or test evidence rules as a shortcut.

## 15 Starter prompt

Paste the following prompt into Codex after placing this package in the intended repository.

> Read AGENTS.md, docs/SOFTWARE_DESIGN.md, and docs/CODEX_IMPLEMENTATION.md. Implement the drone delivery simulation in milestone order, starting with repository inspection and M0 diagnostics. Preserve existing work. Create docs/IMPLEMENTATION_STATUS.md with evidence and blockers. Validate the toolchain before integrating SDK APIs, and build simulator independent domain code where useful. Use only simulated vehicle endpoints, never hardware. Complete each feasible milestone slice, run meaningful checks, and report precisely what was verified. If PX4 or Gazebo cannot run in this host, continue with the domain and fake adapters, provide the exact local verification steps, and leave SITL gates explicitly pending. Do not claim a successful flight without actual simulator evidence.

