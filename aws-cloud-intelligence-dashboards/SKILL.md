---
name: aws-cloud-intelligence-dashboards
description: >-
  Deploys and manages AWS Cloud Intelligence Dashboards (CUDOS, Cost Intelligence, KPI,
  Trusted Advisor/TAO, CORA, Graviton, Health Events, FOCUS, and the rest of the CID
  catalog) in Amazon QuickSight in the user's own AWS account, by driving the cid-cmd CLI
  and the CID CloudFormation templates with the user's own AWS credentials. Use when a user
  wants to deploy, update, delete, migrate to CUR 2.0, check the status of, or share
  CID/QuickSight cost dashboards, or set up the CUR 2.0, FOCUS, and Cost Optimization Hub
  data exports and the advanced Data Collection pipeline that feed them.
version: 1
---

# AWS Cloud Intelligence Dashboards

## Overview

AWS Cloud Intelligence Dashboards (CID) are Amazon QuickSight dashboards (CUDOS, Cost
Intelligence, KPI, Trusted Advisor (TAO), CORA, Graviton, Health Events, FOCUS, and more)
that turn Cost and Usage data into cost-optimization insight. This skill deploys and manages
them in the user's own AWS account with two tools:

- the **`cid-cmd`** CLI ([PyPI](https://pypi.org/project/cid-cmd/), latest release) for the
  dashboard lifecycle, and
- **CloudFormation** (the CID templates) for the underlying infrastructure: CUR 2.0 / FOCUS /
  Cost Optimization Hub data exports and the advanced Data Collection pipeline.

You act with the user's own AWS credentials. Product documentation:
[Cloud Intelligence Dashboards on AWS](https://docs.aws.amazon.com/guidance/latest/cloud-intelligence-dashboards/).

**Works best with** the [AWS MCP server](https://docs.aws.amazon.com/aws-mcp/), which is recommended for sandboxed execution and audit logging of the AWS CLI calls this skill makes (stack status, QuickSight and Glue lookups, CloudFormation deploys). All guidance also works with standard AWS CLI access. The `cid-cmd` steps need a local shell with Python 3.11+.

## Routing

| User need | Action |
|-----------|--------|
| What a dashboard shows: its sheets and visuals, whether it covers a service or metric, which dataset or view feeds a visual | Read [dashboard-content.md](references/dashboard-content.md) |
| Which dashboards are available to deploy that the user doesn't have yet | Read the last section of [dashboard-catalog.md](references/dashboard-catalog.md#which-dashboards-can-i-deploy-that-i-dont-have-yet) |
| Pick a dashboard or resolve a friendly name ("CUDOS", "trusted advisor") to an id | Read [dashboard-catalog.md](references/dashboard-catalog.md) |
| Deploy or update a dashboard unattended, or fix a `Please set parameter` abort | Read [deploy-parameters.md](references/deploy-parameters.md) |
| Update a dashboard, or check for updates: decide whether the new version needs `--recursive`, `--force`, a newer cid-cmd, or a newer Data Collection | Read [update-changelog-check.md](references/update-changelog-check.md) |
| Decide whether a `cid-cmd` run passed or failed | Read [output-interpretation.md](references/output-interpretation.md) |
| Set up CUR 2.0 / FOCUS / COH data exports, Data Collection, or the CFN dashboard stack | Read [cloudformation-templates.md](references/cloudformation-templates.md) |
| Choose Data Collection modules for an advanced dashboard | Read [data-collection-modules.md](references/data-collection-modules.md) |
| Run `open`, `export`, `refresh`, `map`, `csv2view`, `init-qs`, `create-cur-table`, `create-cur-proxy`, `cleanup`, `teardown`, or the Quick Agent commands (`list-agents`, `create-agent`, `update-agent`, `delete-agent`) | Read [cid-cmd-subcommands.md](references/cid-cmd-subcommands.md) |
| Status, delete, migrate to CUR 2.0, or share | Follow the sections below |

## Files

| File | Content |
|------|---------|
| [dashboard-catalog.md](references/dashboard-catalog.md) | Dashboard ids and friendly-name aliases (snapshot), resolution rule, which dashboards need Data Collection, live list of dashboards available but not deployed |
| [dashboard-content.md](references/dashboard-content.md) | Per-dashboard docs page and upstream definition YAML, read-only outline of sheets, visuals, and datasets |
| [deploy-parameters.md](references/deploy-parameters.md) | `deploy` / `update` options, the unattended parameter chain and how to discover each value, deploy-time issues (stale CUR view, `--timezone`) |
| [update-changelog-check.md](references/update-changelog-check.md) | Pre-update changelog check: how to find the changelog, signals for `--recursive` / `--force` / minimum cid-cmd and Data Collection versions, cid-cmd's major-version auto-recursive behavior |
| [output-interpretation.md](references/output-interpretation.md) | Failure-log rule, the `status` picker-traceback exception, real vs. recoverable `ERROR -` lines |
| [cloudformation-templates.md](references/cloudformation-templates.md) | Latest public template URLs and the pre-deploy template check, per-stack parameters, multi-account Data Exports ordering, region constraints |
| [data-collection-modules.md](references/data-collection-modules.md) | 25 Data Collection modules mapped to `IncludeXModule` parameters, dashboards, and cross-account roles |
| [cid-cmd-subcommands.md](references/cid-cmd-subcommands.md) | The other `cid-cmd` subcommands with real flags and safety framing |

## Guardrail — where this skill's own files live (MCP vs local install)

This skill can be loaded two ways, and they resolve the skill's own bundled
files from different places. Determine how the skill was loaded before reading
a reference:

- **Loaded through the AWS MCP `retrieve_skill` tool:** The skill is not
  installed on the local filesystem. You MUST fetch each reference via
  `retrieve_skill` with the `file` parameter (e.g.
  `file="references/dashboard-catalog.md"` or
  `file="references/deploy-parameters.md"`). Do NOT `file_read` these paths
  locally — they do not exist on disk.
- **Installed locally** (e.g. `.kiro/skills/aws-cloud-intelligence-dashboards/` or
  `~/.claude/skills/aws-cloud-intelligence-dashboards/`): Read files from the local
  skill directory using relative paths.

This distinction applies only to the skill's own packaged files. User data and
session artifacts are always read from and written to the user's working
directory. Never fetch or write customer data through `retrieve_skill`.

## Prerequisites

- **Python 3.11+** and the **latest `cid-cmd` from PyPI**. cid-cmd releases are reviewed and
  tested, and fixes the skill relies on (for example `--on-drift override` without a
  terminal, 4.4.20) ship only in new releases. So always use the latest:
  - If `cid-cmd` isn't installed, run `pip install --upgrade cid-cmd` in a scratch virtual
    environment.
  - If it is installed, compare `cid-cmd --help` (the first line shows the version) with the
    latest on PyPI
    (`curl -fsSL https://pypi.org/pypi/cid-cmd/json | jq -r .info.version`). If it is older,
    upgrade it in a scratch virtual environment, and don't change the user's system Python.
  - Installing or upgrading is a local tooling step. It makes no change to the user's AWS
    account, so it is compatible with read-only requests such as "just check, don't change
    anything". Mention it in your answer.

  The behavior notes in this skill were last checked on cid-cmd 4.4.21. If a newer release
  behaves differently, trust its `--help` and output over this text.
- **Locate `cid-cmd`; do NOT assume it is on PATH.** The console script may land in a
  directory that is not on `PATH`, most notably on macOS **python.org framework builds**,
  where it goes to `/Library/Frameworks/Python.framework/Versions/<X.Y>/bin/cid-cmd`. Before
  running anything, locate the executable in this order: (1) the active venv's `bin/`
  (`<venv>/bin/cid-cmd`); (2) `which cid-cmd`; (3) the interpreter's scripts directory,
  `python3 -c "import sysconfig; print(sysconfig.get_path('scripts'))"`, and the macOS
  framework bin path above. Use the full path you find.
- The user's **own AWS credentials** (a profile, SSO session, or ambient environment or
  instance role) with least-privilege permissions for QuickSight, Athena, Glue, S3, IAM, and
  CloudFormation (see Security Considerations).
- **QuickSight Enterprise Edition with SPICE capacity activated** in the target region
  **before** deploying any dashboard. `cid-cmd deploy` does not provision QuickSight, but
  `cid-cmd init-qs` can activate QuickSight Enterprise (mutating and billable; see
  [cid-cmd-subcommands.md](references/cid-cmd-subcommands.md)).
- **Outbound HTTPS to `raw.githubusercontent.com`.** cid-cmd fetches its extended dashboard
  catalog (CORA, TAO, Graviton, Health Events, and others) on every run. If that is blocked,
  catalog dashboards fail with "unknown dashboard-id" even though cid-cmd works.

## Authentication

Exactly one model: the user's own AWS credentials via boto3's standard chain (environment,
`~/.aws` profiles, SSO, instance or task role). Every operation takes an **optional profile
and region**:

- `cid-cmd`: pass `--profile <profile>` and `--region_name <region>`.
- `aws` CLI: pass `--profile <profile>` and `--region <region>`.

When a profile is given, unset `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`,
`AWS_SESSION_TOKEN`, and `AWS_SECURITY_TOKEN` in the environment first, so a stale exported
key cannot silently shadow the profile the user asked for. If no region is given, let the AWS
tools resolve it from the profile or environment. Do not invent one: guessing wrong targets
the wrong place for CloudFormation.

Verify credentials before anything else with `aws sts get-caller-identity`.

## Safety model (read this before any mutating action)

Confirmation is your responsibility as the agent; nothing in `cid-cmd` or this skill enforces
it for you.

- You **MUST** obtain explicit user confirmation before any mutating action: deploy, update,
  delete, migrate, or share a dashboard; create, update, or delete any CloudFormation stack.
- You **MUST** clearly state that the action is **real and billable** in the user's AWS
  account before you run it.
- You **MUST** prefer read-only operations (status, check for updates, list dashboards, stack
  status, list stacks, diagnostics) when the user is only inspecting. Those **MAY** run
  without confirmation.
- You **MUST** warn that `update --recursive` and `--on-drift override` can **overwrite
  dashboard and Athena-view customizations** (see [Recursive updates](#recursive-updates)).
- You **MUST** warn that `cid-cmd delete` and CloudFormation **stack deletion are destructive
  and irreversible**.
- You **MUST NOT** disable safety protections or delete production stacks without explicit
  user direction. When you cannot tell whether a resource is production, assume it is and ask.

## Judging success and failure of `cid-cmd`

Full rules, with the exact output patterns, are in
[output-interpretation.md](references/output-interpretation.md). Read it before reporting a
result. In short:

- **Exit code 0 does NOT mean success.** cid-cmd catches its own errors, logs them, and exits
  0. Treat a run as **FAILED** if any output line (log lines go to stderr, so capture
  both streams) begins with `ERROR -` or `CRITICAL -`, or
  the process exits non-zero.
- **One exception:** `status` prints a complete `Dashboard status:` block and then a
  traceback ending in `Please set parameter <dashboard-id>. Unable to request user ...`. That
  is SUCCESS. The same phrasing for any other parameter during `deploy` / `update` is a REAL
  failure. Do not filter the traceback out of what you read (for example with `grep -v`):
  check that its last line names the dashboard id before you call the run a success.
- **Always run `status` with `< /dev/null`** (and a timeout). Otherwise its interactive menu
  can hang in an agent terminal.
- `ERROR - ... Will continue without.` lines can be recoverable warnings. Confirm with the
  completion message and `aws quicksight list-dashboards`; the deployed state is the source of
  truth.
- Run cid-cmd from a scratch temporary directory; it writes `cid.log` to the working
  directory.

## Choosing a dashboard

Resolve the user's phrase to a canonical `--dashboard-id` using
[dashboard-catalog.md](references/dashboard-catalog.md). It has the full id list, the
friendly-name aliases, and which dashboards need the Data Collection pipeline.

**"CUDOS" always means `cudos-v5`.** Any mention of CUDOS, in any case and with or without
backticks (including the bare id `cudos`), resolves to `cudos-v5` for every operation: status,
deploy, update, delete, content questions, and changelogs. Never pass `--dashboard-id cudos`.
The id `cudos` is the deprecated CUDOS v4. Mention it only when a v4 dashboard with that id is
already deployed in the account. Then call it "CUDOS v4 (legacy)" and suggest moving to
`cudos-v5`.

When the user asks what a dashboard shows, answer from its docs page and its upstream
definition YAML, following [dashboard-content.md](references/dashboard-content.md). Don't
answer from memory, and cite the source.

When you name, compare, or recommend dashboards, give each one's canonical id (for example
`cudos-v5`, `cost_intelligence_dashboard`, `kpi_dashboard`).

## Read-only operations

All read-only; no confirmation required.

- **Overall status / credential check:**
  `cid-cmd --profile "<p>" --region_name "<r>" -y status < /dev/null`
- **Check a dashboard for updates:**
  `cid-cmd --profile "<p>" --region_name "<r>" -y status --dashboard-id "<id>" < /dev/null`
  (the `< /dev/null` is REQUIRED; without it the command can hang on an interactive menu).
  When an update is available, also report what the changelog says it needs (`--recursive`,
  `--force`, a minimum cid-cmd or Data Collection version). Fetching the changelog is
  read-only; see [update-changelog-check.md](references/update-changelog-check.md).
- **List deployed dashboards:** cid-cmd has no non-interactive list subcommand, so use
  QuickSight directly:
  `aws quicksight list-dashboards --aws-account-id "<acct>" [--profile "<p>" --region "<r>"]`
  (lists ALL QuickSight dashboards in the region, not just CID's).
- **CloudFormation stack status:** `aws cloudformation describe-stacks --stack-name "<name>"`
- **List CloudFormation stacks:** `aws cloudformation list-stacks` (filter by a `cid` name
  prefix to surface CID stacks).
- **Is a CID stack up to date:** compare the version in its deployed template's `Description`
  with the version in the latest public template's `Description`, and report it (for example
  "v3.14.8, latest is v3.14.11"). See "Is a deployed stack up to date?" in
  [cloudformation-templates.md](references/cloudformation-templates.md).
- **Not read-only:** `aws cloudformation detect-stack-drift` and `detect-stack-set-drift`
  start a drift operation and leave a drift record. Do not run them for "just check"
  requests.
- **Diagnostics:** confirm `cid-cmd` is installed (`cid-cmd --help`; the first line shows the
  version, e.g. `... CLI 4.4.21`); confirm credentials resolve (`aws sts get-caller-identity`);
  read the latest template versions as described in
  [cloudformation-templates.md](references/cloudformation-templates.md); and confirm outbound
  HTTPS to `raw.githubusercontent.com` is reachable.

## Deploy a dashboard (`cid-cmd deploy`)

Mutating; confirm first. Unattended runs must supply the whole Athena / QuickSight parameter
chain up front, or cid-cmd aborts with `Please set parameter <X>`. For a CUR-based dashboard
(CUDOS, Cost Intelligence, KPI):

```bash
cid-cmd --profile "<p>" --region_name "<r>" -y deploy --dashboard-id "<id>" \
  --athena-database "<db>" \
  --athena-workgroup "<workgroup-with-a-result-location>" \
  --quicksight-datasource-id "<quicksight-athena-datasource-id>" \
  --cur-table-name "<cur-table-or-view>"
```

For `cudos-v5`, `cost_intelligence_dashboard`, `kpi_dashboard`, and `focus-dashboard`, list
the available tags, cost categories, and (CUR 2.0) `line_item_iam_principal` first, as
interactive cid-cmd does. Ask the user to pick the strict minimum, and pass them with
`--resource-tags` (see "Tags, cost categories, and IAM principal" in
[deploy-parameters.md](references/deploy-parameters.md)). Unattended, cid-cmd otherwise
deploys with no tags.

All options, how to discover each value, and deploy-time issues (stale CUR view,
`--timezone`, `cora` and `focus-dashboard` export requirements) are in
[deploy-parameters.md](references/deploy-parameters.md). Advanced dashboards show no data
until the Data Collection pipeline has run (see [Infrastructure via CloudFormation](#infrastructure-via-cloudformation)).

## Update a dashboard (`cid-cmd update`)

Mutating; confirm first.

```bash
cid-cmd --profile "<p>" --region_name "<r>" -y update --dashboard-id "<id>" \
  --athena-database "<db>" --athena-workgroup "<workgroup-with-a-result-location>" [options]
```

Run `update` only for the dashboards `status` reports as having an update available. With an
explicit `--dashboard-id`, cid-cmd republishes the dashboard even when it is already up to
date, with or without `--force`, so skip up-to-date dashboards unless the user asks for a
rebuild or a CUR 2.0 migration. `--recursive` also updates datasets and Athena views and
**can overwrite customizations**.

**Check the changelog before every update.** You **MUST** read the dashboard's upstream
changelog for every version between the deployed and the latest one, following
[update-changelog-check.md](references/update-changelog-check.md), before you propose the
command. cid-cmd checks only the major version, so the changelog is the only place that says
whether a minor release needs `--recursive` or `--force`, a newer cid-cmd, or a newer Data
Collection. A major version bump makes cid-cmd go recursive by itself under `-y`. Treat it as a
recursive update. Cite the changelog version behind each flag. If the changelog can't be
fetched, say so and let the user choose.

### Recursive updates

This section is the single source for recursive updates. It covers changelog-driven ones, major
version bumps, CUR 2.0 migrations, and forced rebuilds.

An agent shell has no terminal. Without one, cid-cmd keeps the existing Athena views, even with
`--recursive`, unless you pass `--on-drift override`. A recursive update against old views fails
on missing columns. It can also leave the dashboard **partly broken**: migrating a dataset to
the new data preparation experience deletes the dataset, and then recreating it fails.

1. **Warn and confirm, in one message.** Say that the views and datasets of `<ids>` will be
   **replaced with the new versions**, that customizations to them are lost, and that views
   shared with other dashboards (for example `summary_view` for `cudos-v5`,
   `cost_intelligence_dashboard`, and `kpi_dashboard`) change for those dashboards too. Say it
   is real and billable, and that it can take more than 15 minutes. In the same message, ask
   for the timezone (see `--timezone` in [deploy-parameters.md](references/deploy-parameters.md))
   and, for CUDOS v5, Cost Intelligence, KPI, and FOCUS, confirm the **tag selection**. Show the
   current tags and ask whether to keep or change them (see "Tags, cost categories, and IAM
   principal" in [deploy-parameters.md](references/deploy-parameters.md)). Without an explicit
   `--resource-tags`, an unattended run can drop every tag column.
2. **After confirmation, run it unattended, in the background:**

   ```bash
   cid-cmd --profile "<p>" --region_name "<r>" -y update --recursive --on-drift override \
     --dashboard-id "<id>" --athena-database "<db>" --athena-workgroup "<workgroup>" \
     --timezone "<tz>" [--resource-tags "<names>"] [--force] [other chain params]
   ```

   Don't ask the user to run it in a terminal themselves. Releases before cid-cmd 4.4.20
   ignore `--on-drift override` without a terminal, so use the latest release (see
   Prerequisites). If you can't, stop.
3. **If the user declines,** don't run the recursive update. A plain update is possible only for
   a minor or patch bump. Say that the dashboard may break or show stale data until a recursive
   update is done. On a major bump, a plain update goes recursive anyway, so report that the
   dashboard stays on its current version.

Without `--recursive`, cid-cmd reuses the dashboard's existing datasets and views. The only
view it writes is its own parameter store, `cid_parameters`, in the `--athena-database` you
pass. Pass the database the user names; if they name none, use the one the dashboard's
datasets read. If the two differ, run the update anyway and report the mismatch, unless one of
the dashboard's datasets is missing: cid-cmd would then create it in that database, so stop
and ask. Options and the parameter chain are in
[deploy-parameters.md](references/deploy-parameters.md).

## Migrate to CUR 2.0

A recursive update of all dashboards. Follow [Recursive updates](#recursive-updates), with
this command:

```bash
cid-cmd --profile "<p>" --region_name "<r>" -y update --force --recursive --on-drift override --timezone "<tz>"
```

It repoints existing dashboards at the CUR 2.0 table. A CUR 2.0 data export must already exist
and be delivering data, because this command doesn't create it (see the Data Exports stack).
Afterwards, check that the dashboard datasets and views now read CUR 2.0.

## Delete a dashboard (`cid-cmd delete`)

Mutating and **destructive/irreversible**; confirm first.

```bash
cid-cmd --profile "<p>" --region_name "<r>" -y delete --dashboard-id "<id>" [--athena-database "<name>"]
```

Deletes the dashboard and its unshared dependencies. Dashboards deployed via CloudFormation
are not cid-cmd dashboards; delete those by deleting the stack instead.

## Sharing is not automatable

Automated sharing with specific users, groups, or folders does **not** work (checked in
cid-cmd 4.4.21; the `share` code path is unchanged since 4.4.17). The `share` subcommand parses
`--share-method`, `--folder-method`, `--folder-id`, `--folder-name`, `--quicksight-user`, and
`--quicksight-group`, but its body ignores them (it calls the share routine with only the
dashboard id, unlike deploy/update/delete, which forward their options). Run
non-interactively, it then fails on a prompt it cannot satisfy
(`Exception: Please set parameter share-method. Unable to request user ...`).

- You **MUST NOT** claim a share succeeded via automation.
- Instead, instruct the user to share through the **QuickSight console**, or to run
  `cid-cmd share` **interactively** in a real terminal. Share only with the principals the
  user names (see Security Considerations).

For reference, the `share` arguments are
`--dashboard-id <id> --share-method <account|user|folder>`; `user` requires
`--quicksight-user`; `folder` requires `--folder-method <new|existing>` then `--folder-name`
(new) or `--folder-id` (existing); `account` takes none of those.

Account-wide sharing is the one exception (added in cid-cmd 4.4.18):

- `update --dashboard-id "<id>" --share-with-account yes` shares the dashboard with
  **everyone in the account**, and `--share-with-account no` revokes that. Omitting the flag
  leaves sharing unchanged. It is mutating, and it is not per-user sharing, so use it only
  when the user explicitly asks to share with the whole account.
- This is a full dashboard `update`, not a sharing-only toggle: cid-cmd first brings the
  dashboard to the latest version and only then applies the sharing change. Confirm it with
  the user as an update, and pass the same unattended Athena parameter chain as any
  `update` (see [deploy-parameters.md](references/deploy-parameters.md)), or it aborts on a
  `Please set parameter ...` prompt before sharing changes.
- `deploy -y` shares a newly created dashboard with everyone in the account by default (the
  share prompt defaults to yes, and the `deploy` flag cannot be set to `no`). Tell the user
  before deploying. If they do not want account-wide sharing, revoke it afterwards with
  `update --dashboard-id "<id>" --share-with-account no`.

## Other `cid-cmd` subcommands

Full how-tos with real flags and safety framing are in
[cid-cmd-subcommands.md](references/cid-cmd-subcommands.md). Highlights:

- **Read-only:** `open` (headless it prints no usable URL; use `status --dashboard-id`
  instead); `export` (read-only in `definition` mode, but mutating when it creates a
  QuickSight `template`, so confirm that path).
- **Mutating; confirm first:** `refresh` (SPICE refresh, consumes billable ingestion);
  `init-qs` (**activates QuickSight Enterprise, billable**); `create-cur-table` /
  `create-cur-proxy` (stand up or translate a CUR Athena table or view without the Data
  Exports stack); `map` and `csv2view` (build the account map and CSV-backed views that
  `deploy --account-map-source csv --account-map-file <csv>` consumes).
- **Unattended defaults:** with `-y` and no terminal, a free-text
  prompt that has a default now takes it instead of aborting. For
  `export --dashboard-export-method template` that default makes `--reader-account` `*`
  (every AWS account), so always pass `--reader-account` explicitly.
- **Quick Agents:** `list-agents` is read-only; `create-agent` and
  `update-agent` are mutating (they build an Amazon Quick Space and Agent over dashboards
  that are already deployed); `delete-agent` is destructive. Confirm all but `list-agents`.
- **Destructive:**
  - `cleanup` deletes unused QuickSight datasets and Athena views. **Confirm.**
  - `teardown` deletes **ALL** CID assets. Upstream's own docstring says *"VERY DANGEROUS. DO
    NOT USE THIS COMMAND."* **Refuse by default.** Run it only on explicit, unambiguous user
    direction that names `teardown` specifically, with a second confirmation; otherwise steer
    to targeted `delete` or `cleanup`.

## Infrastructure via CloudFormation

Mutating; confirm first. Use `aws cloudformation create-stack` / `update-stack` with the
template URL and `--capabilities CAPABILITY_NAMED_IAM` (`aws cloudformation deploy` accepts
only a local template file). Before every create or update, run the read-only template check
in [cloudformation-templates.md](references/cloudformation-templates.md). It covers the latest
template URLs, the version and parameter check, IAM roles, regions, and the per-stack
parameters and gotchas. Take parameter names from the template itself.

| Stack | Account | Feeds |
|---|---|---|
| Data Exports (CUR 2.0 / FOCUS / Cost Optimization Hub) | Destination (data collection) account, plus each source account | Foundational dashboards, `focus-dashboard`, `cora`. One per account: check for an existing `*DataExports*` stack first. |
| Legacy CUR 1.0 aggregation | Destination account | Old CUR 1.0 setups. Prefer Data Exports. |
| Data Collection read permissions | Organizations **management** account | Cross-account read roles for Data Collection |
| Data Collection | Dedicated **data collection** account, in a supported region | Advanced dashboards. Modules: [data-collection-modules.md](references/data-collection-modules.md). |
| CFN dashboard deploy (`cid-cfn.yml`) | Dashboard account | Alternative to `cid-cmd deploy`, limited to CUDOS v5, Cost Intelligence, KPI, TAO, and Compute Optimizer |

Order:

- **Foundational dashboards** (CUDOS v5, Cost Intelligence, KPI): make sure a CUR 2.0 export
  exists (Data Exports stack, or reuse one), then `cid-cmd deploy`.
- **Advanced dashboards** (TAO, Compute Optimizer, Graviton, Health Events, and others):
  deploy the read-permissions stack, then the Data Collection stack with the modules the
  dashboard needs. Wait until the pipeline has collected data at least once, then
  `cid-cmd deploy`.

`cid-cmd deploy` is the default way to deploy dashboards. It covers the full catalog. Use
`cid-cfn.yml` only when the user wants the dashboards tracked in one stack.

Manage stacks with `describe-stacks` (status), `describe-stack-events` (look for `*_FAILED`),
and `list-stacks` (filter on `cid`). `delete-stack` is **destructive**: confirm first.

Teardown order (from the CID docs): delete the Data Collection stack only after emptying its
Destination S3 bucket, or the deletion fails. Historical CUR data stays in the
`<resource-prefix>-<payer-account-id>-shared` bucket, so empty it manually if the user wants it
gone. Also delete the Data Exports Source stack in the payer or source account.

## Gotchas and troubleshooting

- **"Unknown dashboard-id" on a catalog dashboard.** If the catalog fetch from
  `raw.githubusercontent.com` is blocked, cid-cmd silently falls back to its ~6 bundled
  dashboards and catalog-only dashboards (CORA, TAO, Graviton, Health Events, and others)
  fail with "unknown dashboard-id". Check reachability first.
- **The two most common macOS setup failures (python.org framework builds):**
  - **`cid-cmd: command not found` after a successful `pip install`.** The console script
    went to the framework bin directory, which is often not on `PATH`. Do not conclude it
    failed to install; locate it (see Prerequisites) and invoke it by full path.
  - **`CERTIFICATE_VERIFY_FAILED` on every HTTPS call.** python.org framework builds can ship
    without a CA bundle, so every outbound HTTPS call (the catalog fetch and AWS API calls)
    fails. It looks like a network problem but is a local Python issue. Fix: run the bundled
    `/Applications/Python <X.Y>/Install Certificates.command` once for that interpreter.
- **`cid.log` in the project folder.** Run cid-cmd from a scratch temporary directory.
- **Credential errors; tell them apart from the output text:**
  - profile not found: check `aws configure list-profiles`.
  - `InvalidClientTokenId` / expired security token: re-authenticate (e.g. `aws sso login`).
  - `UnrecognizedClientException`: the access key or secret is wrong.
  - `AccessDenied` / "not authorized": credentials are valid but the IAM policy lacks
    permission for the action.
- **`Please set parameter <X>` during deploy or update.** A required value is missing; see
  [deploy-parameters.md](references/deploy-parameters.md).
- **Data Collection region limit.** The Data Collection stacks deploy only in the 18
  supported regions (see [cloudformation-templates.md](references/cloudformation-templates.md)).

## Security Considerations

- **Least-privilege IAM.** Run as a principal scoped to the actions the task needs:
  QuickSight dashboard, dataset, data source, and template actions; Athena queries and
  workgroup reads; Glue Data Catalog reads and the crawler/table writes for CUR setup; S3
  access to the CUR and Athena result buckets only; CloudFormation stack actions; and the IAM
  role actions the CID templates need. Do not use `AdministratorAccess` or `*FullAccess`
  policies. Use read-only credentials for status and diagnostics.
- **`CAPABILITY_NAMED_IAM`.** Every CID stack creates named IAM roles (QuickSight data source
  roles, Data Collection Lambda and cross-account read roles, StackSet-deployed roles across
  OUs). Acknowledging this capability lets the template create those roles. Tell the user
  which roles a stack creates before deploying, review them with
  `aws cloudformation get-template-summary`, and use `RolePath` or permission boundaries where
  their organization requires it.
- **Protect cost data at rest and in transit.** CUR, FOCUS, and Athena query results contain
  account ids, resource ids, and spend. Keep those S3 buckets encrypted (SSE-S3 or SSE-KMS;
  pass KMS key ARNs via `DataBucketsKmsKeysArns` when buckets use KMS) with S3 Block Public
  Access enabled, and grant QuickSight access only to the buckets it needs
  (`--allow-buckets`). AWS API calls use TLS.
- **Restrict dashboard sharing.** Share dashboards, folders, and `export --reader-account`
  grants only with the QuickSight users, groups, or accounts the user explicitly names. Never
  use `--reader-account '*'` unless the user asks for it and confirms. Use row-level
  security (`--rls ENABLED`) when viewers should see only their own accounts.
- **Redact identifiers in shared output.** When you summarize results for tickets, chats, or
  documents, redact 12-digit account ids, ARNs, bucket names, and QuickSight URLs unless the
  user asks to keep them. Never print or log credentials.
- **Audit trail.** Ensure CloudTrail is enabled in each account touched, so QuickSight,
  CloudFormation, IAM, and Athena actions taken during deployment are logged.
- **Validate and quote user-supplied values** before they go into a shell command or JSON
  body. Check dashboard ids against `^[A-Za-z0-9_-]+$`, account ids against `^[0-9]{12}$`,
  stack names against `^[A-Za-z][A-Za-z0-9-]{0,127}$`, and bucket names against S3 naming
  rules; reject anything else. Always double-quote variables in commands, and build JSON
  (for example CloudFormation `--parameters`) with `jq -n --arg`, never by string
  concatenation.
- **No production-like defaults.** Ask the user for stack names, prefixes, and target
  accounts rather than inventing names, and confirm the account and region before every
  mutating action.
- **References:** [IAM security best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html),
  [Amazon QuickSight security](https://docs.aws.amazon.com/quicksight/latest/user/security.html),
  [Amazon S3 security best practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html),
  [CloudFormation IAM capabilities](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/control-access-with-iam.html),
  [CloudTrail security best practices](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/best-practices-security.html)
