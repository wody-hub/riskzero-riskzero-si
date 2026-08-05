# Timestamped Task Management Design

## Context

riskzero-si currently stores every pipeline artifact under
`.si-planning/{feature-name}/`. The feature name is supplied independently to
the skills, so two runs for the same feature are difficult to distinguish and
later stages may select the wrong directory.

The new contract gives every run a unique task ID, requires explicit task
selection when resuming work, and adds safe commands for listing and deleting
tasks. The same contract must work in Codex and Claude Code.

## Goals

- Create new task IDs in `yyyyMMddHHmmss_task-name` format.
- Reuse one task ID throughout all eight pipeline stages.
- Require an exact task ID when running or resuming an individual downstream
  stage.
- List in-progress tasks by default and completed tasks on request.
- Mark a task complete only after Step 8 writes an explicit PASS result.
- Physically delete one explicitly selected task after user confirmation.
- Preserve existing, non-timestamped task directories as legacy tasks.

## Non-goals

- No central task database, registry, or index file.
- No automatic selection of the newest or similarly named task.
- No bulk deletion, wildcard deletion, or automatic cleanup policy.
- No migration or renaming of existing task directories.

## Terminology

- **Base name**: The human-supplied name used to start new work, such as
  `회원관리`.
- **Task ID**: The generated, reusable identifier, such as
  `20260805103000_회원관리`.
- **Task directory**: The configured artifact directory whose `{feature}` value
  is the full task ID.
- **Legacy task**: An existing artifact directory whose name does not begin
  with a 14-digit timestamp and underscore.

## Command Contract

### Starting or resuming work

| Invocation | Behavior |
|---|---|
| `/riskzero-si-pipeline 회원관리` | Create a new task ID and run from Step 1. |
| `/riskzero-si-plan 회원관리` | Create a new task ID and run Step 1. |
| `/riskzero-si-pipeline 20260805103000_회원관리 --from=3` | Resume the exact existing task at Step 3. |
| `/riskzero-si-plan 20260805103000_회원관리` | Re-run Step 1 for the exact existing task. |
| `/riskzero-si-pipeline --init` | Initialize configuration; no task is created. |

A non-timestamped argument is a base name and may create work only when the
execution includes Step 1. An argument matching the task ID format always means
"reuse this exact task" and must already exist.

Step 2 through Step 8 accept an exact task ID as the first task-related
argument:

```text
/riskzero-si-plan-review {task-id}
/riskzero-si-impl {task-id}
/riskzero-si-review {task-id} [path]
/riskzero-si-pr-review {task-id}
/riskzero-si-qa-checklist {task-id}
/riskzero-si-qa {task-id} [URL]
/riskzero-si-browse {task-id} [URL]
```

When a task ID is missing, the skill invokes the task-list behavior, displays
in-progress task IDs, and stops. It never chooses a task automatically.

### Managing tasks

| Invocation | Behavior |
|---|---|
| `/riskzero-si-task` | List in-progress tasks, newest first. |
| `/riskzero-si-task --completed` | List completed tasks, newest first. |
| `/riskzero-si-task clear {task-id}` | Inspect the exact target, request confirmation, then physically delete it. |

`clear` accepts both in-progress and completed tasks. A legacy task can be
cleared only by its exact displayed name and goes through the same safety
checks.

## Task ID Creation

Task creation uses the host machine's local time and the exact format
`date +%Y%m%d%H%M%S`, followed by `_` and the base name.

The base name is trimmed and must be non-empty. It must not contain `/`, `\`,
control characters, `.` or `..` path segments, or glob syntax that could be
interpreted as multiple paths. Spaces and Korean characters remain unchanged.

Directory creation is exclusive. If the computed task directory already
exists, creation fails without modifying it. The user can retry after the
timestamp changes or choose a different base name.

The pipeline creates the task once and passes the resulting full task ID to
every sub-skill. Sub-skills must not regenerate timestamps.

## Storage and State Model

The filesystem is the single source of truth. No status registry is written.

`outputs.root` remains configurable. The full task ID is substituted for the
existing `{feature}` token in `outputs.featureDirPattern`; the default remains:

```text
.si-planning/{task-id}/
```

If a configured pattern introduces nesting, the resolved task path must still
remain inside `outputs.root`. Listing and clearing use the same configured
pattern used by the pipeline.

Step 8 writes a machine-readable final section to `final-report.md`:

```markdown
## 최종 판정
- 최종 상태: PASS
```

The only valid values are `PASS` and `FAIL`.

- **Completed**: `final-report.md` contains the exact PASS marker.
- **In progress**: the report is missing, contains FAIL, or lacks a recognized
  final marker.

Reports created before this change remain in progress until Step 8 is re-run
and writes the normalized marker. This conservative rule prevents an
incomplete or ambiguous report from being presented as completed.

Legacy directories participate in the same state calculation and are shown
with a `[legacy]` label. Timestamped tasks sort by task ID descending; legacy
tasks follow them and sort by modification time descending.

## Components and Responsibilities

### `riskzero-si-task/SKILL.md`

- Defines the user-facing list, completed-list, and clear commands.
- Resolves the project configuration and artifact root.
- Requests explicit user confirmation before destructive execution.
- Provides the no-task-ID fallback used by other riskzero-si skills.

### `riskzero-si-task/scripts/task-manager.sh`

- Performs deterministic task ID creation and exclusive directory creation.
- Classifies task directories from the normalized final-status marker.
- Lists exact task IDs without fuzzy matching.
- Validates and deletes one exact directory only when an internal confirmed
  flag is present.

The script receives resolved configuration values from the calling skill. It
does not maintain hidden state. Its direct clear operation refuses to run
without the confirmation flag, so bypassing the conversational confirmation
does not accidentally delete data.

### Existing pipeline skills

- `riskzero-si-pipeline` and `riskzero-si-plan` distinguish new base names from
  existing task IDs and call the shared creation logic.
- All stage skills resolve artifacts from the supplied full task ID.
- Stage skills with optional path or URL arguments keep those arguments after
  the task ID.
- `riskzero-si-browse` writes the normalized PASS or FAIL marker.
- Missing task IDs route to the in-progress task list and stop.

### Installation and documentation

The existing setup glob automatically links a new `riskzero-si-task` skill,
but setup output and usage documentation must list it explicitly. README.md,
GUIDE.md, MANUAL.md, help routing, command examples, and artifact examples must
use the new task-ID contract consistently.

## Clear Safety Contract

Before deletion, the skill displays:

- exact task ID;
- resolved absolute task directory;
- current in-progress or completed status;
- a statement that physical deletion is not recoverable by riskzero-si.

Deletion proceeds only after the user confirms that exact task ID. The shared
script then independently verifies all of the following:

- the input is one exact name, not a glob or list;
- the resolved target exists;
- the resolved target is the configured task directory inside
  `outputs.root`;
- neither the target nor a path component used to escape the root is a
  symbolic link;
- the internal confirmation value matches the requested task ID.

Failure of any check leaves the filesystem unchanged and reports the reason.
The script never deletes `outputs.root`, the project root, a parent directory,
or multiple task directories.

## Error Handling

- Missing config: use the existing config discovery order; task listing may
  fall back to the default `.si-planning` root, while pipeline execution keeps
  its current config-required gate.
- Missing artifact root: return an empty list, not an error.
- Invalid base name: explain the rejected characters and do not create a
  directory.
- Duplicate task ID: report the collision and do not overwrite.
- Unknown task ID: show the in-progress list and require an exact retry.
- Ambiguous legacy name or unsafe configured pattern: stop without selection
  or deletion.
- Unrecognized final marker: classify as in progress and recommend re-running
  Step 8.

## Verification Strategy

The task manager is tested in an isolated temporary project and output root.
Tests cover:

1. Generated IDs match `^[0-9]{14}_.+$` and preserve the base name.
2. A duplicate computed ID cannot overwrite an existing directory.
3. A task without `final-report.md` is in progress.
4. PASS is completed, while FAIL and malformed reports are in progress.
5. Timestamped results sort newest first and legacy tasks are labeled.
6. Clear without internal confirmation changes nothing.
7. Traversal, glob, multiple-target, root-target, and symlink attempts fail.
8. Confirmed clear removes only the exact requested task and preserves sibling
   tasks.
9. Pipeline and each individual skill document require or pass the same task
   ID.
10. Setup registers the new skill for both Codex and Claude Code.
11. Skill validation and repository documentation consistency checks pass.

## Acceptance Criteria

- Starting `회원관리` creates one directory named like
  `.si-planning/20260805103000_회원관리/` and all stages use it.
- Omitting a task ID in Step 2 through Step 8 displays in-progress tasks and
  performs no stage work.
- `/riskzero-si-task --completed` lists only tasks whose normalized final
  status is PASS.
- `/riskzero-si-task clear {task-id}` cannot delete anything until the exact
  target is confirmed.
- Existing non-timestamped work remains visible as legacy work and is not
  renamed or removed automatically.
- Codex and Claude Code expose the same command behavior after setup.
