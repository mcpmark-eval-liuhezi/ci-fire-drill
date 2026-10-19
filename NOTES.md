# Fire Drill: Stuck CI Run Killed Mid-Flight

A rehearsal of the incident runbook section **"a CI run is stuck and has to be killed mid-flight,"** run end-to-end so the on-call rotation can see how it's done. This note is the summary of the drill.

## Setup

- **Repository:** `mcpmark-eval-liuhezi/ci-fire-drill`
- **Tracking issue:** #1 — "Fire drill: stuck CI run"
- **Workflow:** `.github/workflows/fire-drill.yml`, on `main`
- **Trigger:** The workflow was written so that the very commit adding it (`6e9977a`) kicked off the run — it fires on `push` to `main`.

## The workflow

Two jobs:

| Job | ID | What it does |
| --- | --- | --- |
| `marker` ("Print readiness marker") | 118085330403 | Echoes `FIRE-DRILL-READY-7C4D` and finishes. |
| `stuck-run` ("Simulated stuck job (ten minutes)") | 118085330198 | `sleep 600` — stand-in for the stuck run. |

## What happened, in order

1. **Repository and issue created.** Fresh repo `ci-fire-drill` with README; issue #1 opened describing the rehearsal.
2. **Workflow committed** at `.github/workflows/fire-drill.yml` (commit `6e9977a`). The push triggered **run 39260138170** (run #1 of "Fire Drill", event `push`, branch `main`).
3. **Marker job finished** in ~1s. Its log was pulled and shows the marker line:

   ```
   2026-10-19T13:26:19.8949610Z FIRE-DRILL-READY-7C4D
   ```

4. **Cancel issued while the sleeping job was still running** — approximately one minute into its ten-minute sleep window (job started 13:26:20 UTC, cancel accepted ~13:27 UTC), well inside the window and *after* the log check. `POST .../runs/39260138170/cancel` returned `202 Accepted`.
5. **Run state re-checked** afterwards: the run's status shows `completed` with conclusion `cancelled`, not `success` — i.e. it ended up cancelled rather than finishing on its own. (The job-level view shows the marker job succeeded and the stuck job was cancelled, with the sleeping step terminated before completion.)

## Outcome

- ✅ Marker line present in the marker job's log.
- ✅ Run cancelled mid-flight, inside the ten-minute window, after the log check.
- ✅ Run confirmed cancelled (not run-to-completion).
- ✅ Tracking issue #1 closed as completed, with a final comment quoting the run ID and the marker line.
- ✅ This note and PR "Fire drill notes" left behind for the rotation.

## Notes for on-call

- Cancel returns `202 Accepted` immediately, but the run may still read `in_progress` for several seconds while the runner tears down — re-check before declaring it dead.
- Both jobs start together; the marker job is the one you trust for log evidence, and the sleeping job is the one you kill.
- Everything needed for the write-up (run ID, marker line, timestamps) is visible from the run view and job logs.