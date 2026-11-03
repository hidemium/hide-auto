---
name: "hideauto-schedule-runs"
description: "Creating/editing schedules (schedule_manage) for real bulk runs: which profile type to send, Hidemium schedules, window placement of scheduled browsers (window_arrangement_manage), and reading past runs (run_history)."
---

# hideauto-schedule-runs

Real bulk runs on real profiles always go through `schedule_manage` and the user approval card. Never schedule anything the user did not ask for.

## Before creating a schedule

- The workflow must test clean first (`run_test` / `critic_review`).
- You need: the workflow, the profiles, and the timing. If any is missing → `ask_user` (one call, several questions). A wrong guess here is expensive.
- Profiles follow the workflow's Open & Connect mode:
  - built-in modes → `profiles` = id/path from `profile_manage list`;
  - Hidemium mode (`openConnectMode: 'otherApp'`) → `hidemiumProfiles` [{uuid, name}] from `hidemium_profiles list`, never `profiles` (see skill `hideauto-hidemium`).
- `run_now` with `started:false` did NOT start — report its `reason` truthfully, never claim "it is running".
- `run_stop` can stop a scheduled run, but only with user approval — offer it only when the user wants it.

## Window placement of scheduled browsers

Only browsers opened by SCHEDULE runs are arranged (test runs and manual opens never are). Settings section `displays`.

- Machine-wide setting: `window_arrangement_manage`. Call `get` first to see the screens (`key`), `maxGrid` and `capacity`; `set` changes only what you pass; cols/rows are capped to `maxGrid`.
- `capacity` = cols × rows × screens used, shared by ALL schedules running at the same time. If the schedules running together open more windows than that, tell the user.
- One schedule can differ: `schedule_manage` `arrangement` = `inherit` (default) · `off` · `custom{layout fields to change}`.

## Past runs

When the user asks "what failed / how did the last run go / is this stable lately":

1. `run_history` (`list` / `get` / `stats`, filter by `scheduleId` for one schedule);
2. `run_log_read` with the `runId` and kind: test run → `test`, schedule run → `campaign`.

Do not guess from the graph.

If a tool named here does not exist in your tool list, the app is older than this guide: tell the user to update the app.
