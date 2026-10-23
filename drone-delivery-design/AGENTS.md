# Drone delivery simulation repository instructions

## Project objective

Build a physics based software simulation before hardware. The owner is Makomane Tau. Confirmed environment is Windows with 16 GB RAM, a generic world, one vehicle, and local development through WSL2. Proposed baseline is two pads 100 m apart and a 0.25 kg package.

## Read before implementation

1. `docs/SOFTWARE_DESIGN.md` is the architecture and behavior specification.
2. `docs/CODEX_IMPLEMENTATION.md` is the milestone plan and verification contract.
3. Read existing project instructions and status records before changing source.

## Required boundaries

- Work only with simulated vehicles. Do not scan serial devices, connect hardware, or expose aircraft control to the public internet.
- Keep flight stabilization in PX4. The gateway owns mission execution; the API owns orders, planning, and approval.
- Persist command intent and separate receipt, acknowledgement, and observed outcome.
- Reconcile ambiguous actions after a crash before sending more commands. Do not blindly resend arm, mission start, or payload detach.
- Require operator approval for outbound and return departures. Confirm landing and disarming before simulated package removal.
- Ground truth is for evaluation and simulated mechanism status; navigation uses estimated sensor and autopilot outputs.
- Use explicit coordinate frames, altitude datums, units, and clock sources.
- Keep the first scene small and one run active. Do not add cloud services, fleets, ROS 2, or perception without a documented requirement.
- Pin versions after a real compatibility spike. Do not mix MAVSDK API generations.
- Preserve secrets outside git, retain third party licenses, and ignore large run and build outputs.

## Work and verification

Follow M0 through M7. Create `docs/IMPLEMENTATION_STATUS.md` when implementation starts. Record tested versions, actual commands, evidence paths, and unverified gates. Use meaningful behavior tests for state guards, idempotency, authorization, geometry, and restart recovery.

MOCK, SITL, and REPLAY are separate modes. Fake adapter tests cannot satisfy a SITL gate. Report unavailable integration checks as NOT RUN or BLOCKED with the reason. A document, screenshot, or generated config alone does not prove a simulated flight completed.

Use normal implementation judgment within the design. Record material changes in an ADR and update affected contracts and tests. Never silently relax acceptance criteria to turn a failure into a pass. Report what works, what was checked, what remains uncertain, and the next gate.

