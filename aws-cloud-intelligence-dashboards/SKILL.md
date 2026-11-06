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

- the **`cid-cmd`** CLI ([PyPI](https://pypi.org/project/cid-cmd/4.4.19/),
  `pip install cid-cmd==4.4.19`) for the dashboard lifecycle, and
- **CloudFormation** (the CID templates) for the underlying infrastructure: CUR 2.0 / FOCUS /
  Cost Optimization Hub data exports and the advanced Data Collection pipeline.

You act with the user's own AWS credentials. Product documentation:
[Cloud Intelligence Dashboards on AWS](https://docs.aws.amazon.com/guidance/latest/cloud-intelligence-dashboards/).

**Works best with** the [AWS MCP server](https://docs.aws.amazon.com/aws-mcp/), which is recommended for sandboxed execution and audit logging of the AWS CLI calls this skill makes (stack status, QuickSight and Glue lookups, CloudFormation deploys). All guidance also works with standard AWS CLI access. The `cid-cmd` steps need a local shell with Python 3.10+.

## Routing

| User need | Action |
|-----------|--------|
| Pick a dashboard or resolve a friendly name ("CUDOS", "trusted advisor") to an id | Read [dashboard-catalog.md](references/dashboard-catalog.md) |
| Deploy or update a dashboard unattended, or fix a `Please set parameter` abort | Read [deploy-parameters.md](references/deploy-parameters.md) |
| Decide whether a `cid-cmd` run passed or failed | Read [output-interpretation.md](references/output-interpretation.md) |
| Set up CUR 2.0 / FOCUS / COH data exports, Data Collection, or the CFN dashboard stack | Read [cloudformation-templates.md](references/cloudformation-templates.md) |
| Choose Data Collection modules for an advanced dashboard | Read [data-collection-modules.md](references/data-collection-modules.md) |
| Run `open`, `export`, `refresh`, `map`, `csv2view`, `init-qs`, `create-cur-table`, `create-cur-proxy`, `cleanup`, `teardown`, or the Quick Agent commands (`list-agents`, `create-agent`, `update-agent`, `delete-agent`) | Read [cid-cmd-subcommands.md](references/cid-cmd-subcommands.md) |
| Status, delete, migrate to CUR 2.0, or share | Follow the sections below |

## Files

| File | Content |
|------|---------|
| [dashboard-catalog.md](references/dashboard-catalog.md) | 32 canonical dashboard ids, 45 friendly-name aliases, resolution rule, which dashboards need Data Collection |
| [deploy-parameters.md](references/deploy-parameters.md) | `deploy` / `update` options, the unattended parameter chain and how to discover each value, deploy-time issues (stale CUR view, `--timezone`) |
| [output-interpretation.md](references/output-interpretation.md) | Failure-log rule, the `status` picker-traceback exception, real vs. recoverable `ERROR -` lines |
| [cloudformation-templates.md](references/cloudformation-templates.md) | Pinned template versions and URLs, per-stack parameters, multi-account Data Exports ordering, region constraints |
| [data-collection-modules.md](references/data-collection-modules.md) | 24 Data Collection modules mapped to `IncludeXModule` parameters, dashboards, and cross-account roles |
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

- **Python 3.10+** and `pip install cid-cmd==4.4.19` (preferably in a virtual environment).
  The references and limitations in this skill describe cid-cmd 4.4.19.
  If `cid-cmd` is not installed, install `cid-cmd==4.4.19` in a scratch virtual environment before continuing. This is a local tooling step that makes no change to the user's AWS account, so it is compatible with read-only requests such as "just check, don't change anything". Mention in your answer that you installed it.
  If a newer cid-cmd is already installed, use it rather than downgrading; this guidance was verified on 4.4.19.
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
  dashboard and Athena-view customizations**.
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
friendly-name aliases, and which dashboards need the Data Collection pipeline. Note that
`cudos` (friendly) resolves to `cudos-v5`, while the exact id `cudos` is the deprecated v4.

When you name, compare, or recommend dashboards, give each one's canonical id (for example
`cudos-v5`, `cost_intelligence_dashboard`, `kpi_dashboard`), and whenever CUDOS comes up, say
that the exact id `cudos` is the deprecated v4.

## Read-only operations

All read-only; no confirmation required.

- **Overall status / credential check:**
  `cid-cmd --profile "<p>" --region_name "<r>" -y status < /dev/null`
- **Check a dashboard for updates:**
  `cid-cmd --profile "<p>" --region_name "<r>" -y status --dashboard-id "<id>" < /dev/null`
  (the `< /dev/null` is REQUIRED; without it the command can hang on an interactive menu).
- **List deployed dashboards:** cid-cmd has no non-interactive list subcommand, so use
  QuickSight directly:
  `aws quicksight list-dashboards --aws-account-id "<acct>" [--profile "<p>" --region "<r>"]`
  (lists ALL QuickSight dashboards in the region, not just CID's).
- **CloudFormation stack status:** `aws cloudformation describe-stacks --stack-name "<name>"`
- **List CloudFormation stacks:** `aws cloudformation list-stacks` (filter by a `cid` name
  prefix to surface CID stacks).
- **Is a CID stack up to date:** compare its deployed template version with the pin in
  [cloudformation-templates.md](references/cloudformation-templates.md) and report it against
  that pin (for example "v3.14.2, behind the pinned v3.14.8"). Mention a newer upstream
  release only as newer than the pin and not verified.
- **Not read-only:** `aws cloudformation detect-stack-drift` and `detect-stack-set-drift`
  start a drift operation and leave a drift record. Do not run them for "just check"
  requests.
- **Diagnostics:** confirm `cid-cmd` is installed (`cid-cmd --help`; the first line shows the
  version, e.g. `... CLI 4.4.19`); confirm credentials resolve (`aws sts get-caller-identity`);
  note the pinned template versions in
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

All options, how to discover each value, and deploy-time issues (stale CUR view,
`--timezone`, `cora` and `focus-dashboard` export requirements) are in
[deploy-parameters.md](references/deploy-parameters.md). Advanced dashboards show no data
until the Data Collection pipeline has run (see Workflows).

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

Without `--recursive`, cid-cmd reuses the dashboard's existing datasets and views. The only
view it writes is its own parameter store, `cid_parameters`, in the `--athena-database` you
pass. Pass the database the user names; if they name none, use the one the dashboard's
datasets read. If the two differ, run the update anyway and report the mismatch, unless one of
the dashboard's datasets is missing: cid-cmd would then create it in that database, so stop
and ask. Options and the parameter chain are in
[deploy-parameters.md](references/deploy-parameters.md).

## Migrate to CUR 2.0

Mutating; confirm first. This is exactly:

```bash
cid-cmd --profile "<p>" --region_name "<r>" -y update --force --recursive
```

It repoints existing dashboards at the CUR 2.0 table. It **WILL override Athena view
customizations**. A CUR 2.0 data export must already exist and be delivering data; this
command does not create the export. Set up the export first via the Data Exports stack.
Recursive updates can prompt for `--timezone` and can take more than 15 minutes (see
[deploy-parameters.md](references/deploy-parameters.md)). As of cid-cmd 4.4.19, existing
Athena views are replaced only when stdin is a terminal; with no terminal (or stdin closed)
they are kept. Afterwards, check that the dashboard datasets and views now read CUR 2.0. If
the views were kept because there was no terminal, the user must run `update --recursive`
in a terminal to review and replace them.

## Delete a dashboard (`cid-cmd delete`)

Mutating and **destructive/irreversible**; confirm first.

```bash
cid-cmd --profile "<p>" --region_name "<r>" -y delete --dashboard-id "<id>" [--athena-database "<name>"]
```

Deletes the dashboard and its unshared dependencies. Dashboards deployed via CloudFormation
are not cid-cmd dashboards; delete those by deleting the stack instead.

## Sharing is not automatable as of cid-cmd 4.4.19

Automated sharing with specific users, groups, or folders does **not** work in cid-cmd
4.4.19 (the `share` code path is unchanged since 4.4.17). The `share` subcommand parses
`--share-method`, `--folder-method`, `--folder-id`, `--folder-name`, `--quicksight-user`, and
`--quicksight-group`, but its body ignores them (it calls the share routine with only the
dashboard id, unlike deploy/update/delete, which forward their options). Run
non-interactively, it then fails on a prompt it cannot satisfy
(`Exception: Please set parameter share-method. Unable to request user ...`).

- You **MUST NOT** claim a share succeeded via automation.
- Instead, instruct the user to share through the **QuickSight console**, or to run
  `cid-cmd share` **interactively** in a real terminal. Share only with the principals the
  user names (see Security Considerations).

For reference, the `share` arguments as of cid-cmd 4.4.19 are
`--dashboard-id <id> --share-method <account|user|folder>`; `user` requires
`--quicksight-user`; `folder` requires `--folder-method <new|existing>` then `--folder-name`
(new) or `--folder-id` (existing); `account` takes none of those.

Account-wide sharing is the one exception, as of cid-cmd 4.4.19 (added in 4.4.18):

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
- **Unattended defaults (as of cid-cmd 4.4.19):** with `-y` and no terminal, a free-text
  prompt that has a default now takes it instead of aborting. For
  `export --dashboard-export-method template` that default makes `--reader-account` `*`
  (every AWS account), so always pass `--reader-account` explicitly.
- **Quick Agents (new in cid-cmd 4.4.19):** `list-agents` is read-only; `create-agent` and
  `update-agent` are mutating (they build an Amazon Quick Space and Agent over dashboards
  that are already deployed); `delete-agent` is destructive. Confirm all but `list-agents`.
- **Destructive:**
  - `cleanup` deletes unused QuickSight datasets and Athena views. **Confirm.**
  - `teardown` deletes **ALL** CID assets. Upstream's own docstring says *"VERY DANGEROUS. DO
    NOT USE THIS COMMAND."* **Refuse by default.** Run it only on explicit, unambiguous user
    direction that names `teardown` specifically, with a second confirmation; otherwise steer
    to targeted `delete` or `cleanup`.

## Infrastructure via CloudFormation

Use `aws cloudformation create-stack` / `update-stack` with the template URL and
`--capabilities CAPABILITY_NAMED_IAM` (`aws cloudformation deploy` accepts only a local
template file). Exact URLs, pinned versions, per-stack parameters, default stack names, a
validated `jq`-built parameter example, and region constraints are in
[cloudformation-templates.md](references/cloudformation-templates.md). Mutating; confirm
first.

Stacks you can drive:

- **Data Exports** (CUR 2.0 / FOCUS / Cost Optimization Hub), in the destination (data
  collection) account. Params: `DestinationAccountId`, `ManageCUR2`, `ManageFOCUS`,
  `ManageCOH` (`yes|no`), optional `SourceAccountIds`. Only one per account; check for an
  existing `*DataExports*` stack first.
- **Legacy CUR 1.0 aggregation.** Prefer Data Exports for new setups. Params:
  `DestinationAccountId`, optional `SourceAccountIds`.
- **Data Collection Step 1, read-permissions stack**, in the Organizations **MANAGEMENT**
  account. Params: `DataCollectionAccountID`, `OrganizationalUnitIds`, and one
  `IncludeXModule=yes` per module that needs a cross-account role (passing a no-role module
  here is an error).
- **Data Collection Step 2, the pipeline**, in a dedicated **DATA COLLECTION** account.
  Params: `ManagementAccountID`, optional `RegionsInScope`, and one `IncludeXModule=yes` per
  enabled module. Must be in one of the **18 supported regions**. Pricing has no separate
  toggle. Module-to-parameter mapping:
  [data-collection-modules.md](references/data-collection-modules.md).
- **CFN dashboard deploy (`cid-cfn.yml`)**, an alternative to `cid-cmd deploy` limited to 5
  dashboards (CUDOS v5, Cost Intelligence, KPI, TAO, Compute Optimizer). Requires the
  QuickSight prerequisites.

Check and manage stacks:

- Status: `aws cloudformation describe-stacks --stack-name "<name>"` (pull recent `*_FAILED`
  stack events with `describe-stack-events` to diagnose a failure).
- List: `aws cloudformation list-stacks` (filter by a `cid` prefix).
- Delete: `aws cloudformation delete-stack --stack-name "<name>"`. **Destructive**; confirm
  first.

## Workflows

### Foundational dashboards (CUDOS v5, Cost Intelligence, KPI; CUR only)

1. Ensure a CUR 2.0 export exists: deploy the Data Exports stack, or reuse an existing one.
2. `cid-cmd ... deploy --dashboard-id <id>` with the parameter chain above.

### Advanced dashboards (TAO, Compute Optimizer, Graviton, Health Events, and others)

Order matters:

1. Deploy the **Data Collection read-permissions** stack in the Organizations **Management**
   account (fans a read role across OUs via service-managed StackSets; needs Organizations
   trusted access).
2. Deploy the **Data Collection** pipeline stack in a dedicated **Data Collection** account,
   with the modules the dashboard needs enabled (see
   [data-collection-modules.md](references/data-collection-modules.md)).
3. After the pipeline has collected data at least once, deploy the dashboard
   (`cid-cmd ... deploy --dashboard-id <id>`).

### Two ways to deploy a dashboard

- **`cid-cmd deploy`**: the full catalog, run unattended with explicit flags. This is the
  default.
- **CFN path (`cid-cfn.yml`)**: infrastructure as code in one tracked stack, but only 5
  dashboards.

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
