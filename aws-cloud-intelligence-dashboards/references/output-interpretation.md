# Interpreting `cid-cmd` output

How to decide whether a `cid-cmd` run succeeded or failed (last checked on cid-cmd 4.4.21). Apply these
rules to every subcommand (`status`, `deploy`, `update`, `delete`, and the others in
`references/cid-cmd-subcommands.md`).

## Contents

- [Exit code 0 does not mean success](#exit-code-0-does-not-mean-success)
- [The failure-log rule](#the-failure-log-rule)
- [The status picker exception (treat as success)](#the-status-picker-exception-treat-as-success)
- ["Please set parameter" during deploy or update is a real failure](#please-set-parameter-during-deploy-or-update-is-a-real-failure)
- [Recoverable "Will continue without" errors](#recoverable-will-continue-without-errors)
- [Working directory](#working-directory)

## Exit code 0 does not mean success

`cid-cmd`'s CLI wrapper (the `cid_command` decorator in upstream `cid/cli.py`) catches its own
`CidError` / `CidCritical` exceptions, logs them, and returns normally. The process exits
**0 even when the command failed**. Do not trust the exit code alone.

## The failure-log rule

The reliable failure signal is a **log line**. The console log format is fixed as
`%(levelname)s - %(message)s`. Those log lines go to **stderr** (a default Python
`StreamHandler`), while status blocks and completion messages are printed to stdout, so
capture both streams (for example `2>&1`). Scan that output and treat the run as **FAILED**
if **either**:

- any line matches the regular expression `^(ERROR|CRITICAL) -\s` (a line that begins with
  `ERROR -` or `CRITICAL -` followed by a space), **or**
- the process exits **non-zero** (genuine crashes that cid-cmd does not catch, such as
  a missing AWS profile, do exit 1).

Otherwise treat it as **success**.

`WARNING -` lines are not failures. Since cid-cmd 4.4.18, an unattended
`-y` run without a terminal logs `WARNING - Parameter <X> is not set. Using default: <value>`
when it fills a free-text prompt from its default instead of aborting. Report which defaults
were used, because the user did not choose them.

There is exactly **one** exception to "non-zero exit = failure": the `status` picker traceback
below, which exits non-zero but is NOT a real failure and has no `ERROR -` / `CRITICAL -`
line. Reconcile the two rules in that order: the status-picker recognition wins for `status`;
everywhere else, apply the failure-log rule.

## The status picker exception (treat as success)

**Always run `status` with stdin closed and a timeout.** After printing the status,
`cid-cmd status` opens an interactive action menu (Back / Open / Refresh / Update / Exit).
With **no TTY** it raises immediately and the process exits (the traceback case below). In a
shell that provides a **pseudo-TTY** (common for agent and IDE terminals) the menu renders and
blocks, waiting for a keypress, and the command hangs until it is killed. Append `< /dev/null`
and bound the run with a timeout so the behavior is deterministic:

```bash
# read-only status, hardened against the interactive-picker hang
cid-cmd --region_name "<region>" -y status --dashboard-id "<id>" < /dev/null
# optionally wrap with a timeout (for example 180 seconds); status still makes real AWS calls
```

With stdin closed, the picker cannot block. It degrades to the traceback case.

`cid-cmd status` and `cid-cmd status --dashboard-id <id>` print the **real** status and then
may emit a Python traceback. This is a known upstream defect, not a failure: `status()` in
upstream `cid/common.py` calls `display_status()` and `display_url()` and THEN enters the
interactive menu loop, which raises when there is no TTY. The status lookup has already
**completed successfully** before the traceback appears.

**Recognition rule (you MUST apply it).** Treat the run as **SUCCESS** when the output contains a
complete `Dashboard status:` block followed by that specific trailing exception:

1. a line `Dashboard status:`, then the fields `Id: <id>`, `Health: ...`, `Status: ...`,
   `Version: ...` (a healthy block also shows `Datasets:` and the dashboard URL
   `https://.../sn/dashboards/<id>`), **followed by**
2. a Python traceback ending in exactly:
   `Exception: Please set parameter <id>. Unable to request user in environment={...}`
   (where `<id>` is the dashboard id you asked about).

Report the status block above the traceback as the valid result and do **NOT** report failure.

Do NOT apply this leniency to any other output. If the `Dashboard status:` block is
incomplete, the trailer is a different exception, or an `ERROR -` / `CRITICAL -` line appears
in the status text itself, it is a real failure. Report it.

## "Please set parameter" during deploy or update is a real failure

`Please set parameter <X>` is benign **only** when `<X>` is the dashboard id after a complete
status block. `deploy` and `update` can fail with the same phrasing for a different
parameter, for example `athena-database`, `athena-workgroup`, `athena-result-bucket`,
`quicksight-datasource-id`, `cur-table-name-and-db`, or `timezone`, when run unattended
without that parameter. Those are **real failures**: cid-cmd needed a value, had no TTY to
prompt, and aborted before completing the operation (there is no completed result block above
them). Verified live. They are all selection prompts, which still abort (checked on cid-cmd 4.4.21);
the 4.4.18 change that fills free-text prompts from their defaults does not cover them. Supply the missing parameter (see `references/deploy-parameters.md`);
do not treat it as benign.

## Recoverable "Will continue without" errors

`deploy` can emit `ERROR -` lines that cid-cmd recovers from, each ending
`... Will continue without.` (for example, a stale CUR view blocking optional tag-scan
queries). These are warnings, not the operation failing, so the raw failure-log scan can
produce a **false FAILED**. Resolve it with a positive completion signal:

- a successful deploy or update ends with `Update completed`, or
  `... is available at: https://.../sn/dashboards/<id>`; a successful delete prints
  `Dashboard <id> deleted`, **and**
- the dashboard then appears in (or, for delete, is gone from)
  `aws quicksight list-dashboards`.

When every `ERROR -` line is immediately followed by `Will continue without`, the completion
signal is present, and the dashboard is listed, treat the run as **success with warnings**.
Otherwise treat `ERROR -` / `CRITICAL -` as failure. When in doubt, confirm with a follow-up
`status --dashboard-id <id>` or `list-dashboards`. The deployed state is the source of truth.

## Working directory

Run cid-cmd from a scratch temporary directory. It writes a `cid.log` into the current working
directory, and that file does not belong in the user's project.
