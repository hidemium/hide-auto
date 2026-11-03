---
name: "hideauto-hidemium"
description: "Workflows that use Hidemium profiles (Open & Connect mode otherApp / Hidemium), looking up Hidemium profiles, scheduling them, and converting a Hidemium script file (.txt) into HideAuto workflows (hidemium_import)."
---

# hideauto-hidemium

## Running a workflow on Hidemium profiles

`openConnectMode: 'otherApp'` = Hidemium: the profiles live in the Hidemium app, which must be running.

- Find the profile with `hidemium_profiles` (read-only, cannot open/close profiles):
  - `sources` → which of `local` / `cloud` the user's Hidemium plan allows;
  - `list{source, search?, folderIds?, tagIds?}` → profiles [{uuid, name}];
  - `folders` / `tags` give the ids for filtering (ids belong to one source).
- Test run: set `openConnectBrowserUuid` = the profile `uuid` and `openConnectBrowserName` = its `name`, then `run_test`.
- Schedule on that workflow: `schedule_manage` with `hidemiumProfiles` [{uuid, name}], never `profiles` (see skill `hideauto-schedule-runs`).
- A connection error ("Cannot connect to Hidemium at …") means Hidemium is not running or its address is wrong: open Settings with `ui_navigate` section `hidemium` and ask the user to check it — do not retry in a loop.

## Importing a Hidemium script file (.txt)

Tool `hidemium_import`:

1. `preview{path}` first — writes nothing. Tell the user in plain words what will be created: the folder, the workflows, and how many steps need review.
2. `import{path, folder?}` — saves every script of the file into a NEW folder (never overwrites). `role: child` = a script called by another script of the same file (Run Other Workflow).
3. Steps Hidemium has no equivalent for become placeholders flagged for review. For every workflow with `reviewNodeCount` > 0: `critic_review`, then fix those steps before any test run.

If a tool named here does not exist in your tool list, the app is older than this guide: tell the user to update the app.
