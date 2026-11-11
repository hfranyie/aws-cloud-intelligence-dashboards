# `cid-cmd deploy` and `update` parameters

Options for `cid-cmd deploy` and `cid-cmd update`, the parameter chain an unattended run
needs, and deploy-time issues to pass on to the user. Flags match upstream `cid/cli.py`
(last checked on cid-cmd 4.4.21). Every `deploy` and `update` here is mutating: get explicit confirmation first
(see "Safety model" in SKILL.md). The `aws ... list` / `get` discovery commands are
read-only. Validate and quote every user-supplied value (see "Security Considerations" in
SKILL.md).

## Contents

- [Deploy options](#deploy-options)
- [Required parameters for unattended deploy and update](#required-parameters-for-unattended-deploy-and-update)
- [Tags, cost categories, and IAM principal (`--resource-tags`)](#tags-cost-categories-and-iam-principal---resource-tags)
- [Update options](#update-options)
- [Deploy-time guidance for the user](#deploy-time-guidance-for-the-user)
- [Stale CUR view (`VIEW_IS_STALE`)](#stale-cur-view-view_is_stale)

## Deploy options

Pattern:

```bash
cid-cmd --profile "<profile>" --region_name "<region>" -y deploy --dashboard-id "<id>" [options]
```

Options and their allowed values or constraints:

- `--athena-database <name>`
- `--athena-workgroup <name>`
- `--glue-data-catalog <name>`
- `--cur-table-name <name>` (the actual CUR Athena table name; not a CUR version)
- `--quicksight-datasource-id <id-or-arn>`
- `--quicksight-datasource-role-arn <arn>`
- `--allow-buckets <bucket1,bucket2,...>` (comma-separated S3 buckets granted to the default
  CID QuickSight role; grant only the buckets the dashboard needs)
- `--quicksight-user <user>`
- `--quicksight-group <group>`
- `--catalog <files-or-urls>` (comma-separated cid-cmd catalog files or URLs)
- `--account-map-source <csv|dummy|organization>` (lowercased). With `csv`,
  `--account-map-file <existing.csv>` is **required**; `--account-map-file` is valid only when
  the source is `csv`.
- `--account-map-file <existing csv path>` (must exist locally)
- `--resources <local-yaml | https:// URL>` (a local path must exist; otherwise an `https://`
  URL). Used to deploy custom dashboards such as `unit-cost-dashboard` and `shard`.
- `--on-drift <show|override>` (lowercased; `override` **replaces** customized views and
  datasets). Without a terminal, `show` keeps existing Athena views, and `override` replaces
  them (cid-cmd 4.4.20 and later). Every recursive update passes `override` after a warning.
  See "Recursive updates" in SKILL.md.
- `--theme <CLASSIC|MIDNIGHT|SEASIDE|RAINIER>`
- `--currency <USD|GBP|EUR|JPY|KRW|DKK|TWD|INR>`
- `--update <yes|no>` (whether cid-cmd updates existing dashboard elements)
- Dataset overrides: one `--dataset-<name>-id <value>` per override (names limited to
  letters, digits, underscore, hyphen).
- View parameter overrides: one `--view-<view>-<param> <value>` per override (view and param
  names limited to letters, digits, underscore, hyphen).
- `--rls <CLEAR|ENABLED|DISABLED>` with `--rls-dataset-id <id>` **required if and only if**
  `ENABLED` (and only valid when `ENABLED`).
- `--share-with-account` (boolean flag on `deploy`; shares the new dashboard with everyone in
  the account). Unattended `deploy -y` shares with the whole account by default even without
  this flag — warn the user first. On `update` the equivalent is `--share-with-account yes|no`
  (see "Update options").
- `--quicksight-delete-failed-datasource` (boolean flag)
- `--timezone <IANA tz>` (see the unattended chain below)

## Required parameters for unattended deploy and update

The upstream CLI lists the options above as optional, but a headless (no TTY) `deploy` or
`update` **aborts asking for them one at a time** when they are missing, each as
`Please set parameter <X>. Unable to request user ...`. That is a real failure (see
`references/output-interpretation.md`). This still holds for every
parameter in the chain below: they are selection prompts, and the 4.4.18 change that lets
`-y` fill free-text prompts from their defaults (logged as `WARNING - Parameter <X> is not
set. Using default: ...`) does not apply to them. Supply the whole chain up front. For a CUR-based
dashboard (CUDOS, Cost Intelligence, KPI) the working set is (verified live):

```bash
cid-cmd --region_name "<region>" -y deploy --dashboard-id "<id>" \
  --athena-database "<db>" \
  --athena-workgroup "<workgroup-with-a-result-location>" \
  --quicksight-datasource-id "<quicksight-athena-datasource-id>" \
  --cur-table-name "<cur-table-or-view>"
```

The parameters cid-cmd prompts for, in the order they surface, and how to discover each:

- `--athena-database`: list with `aws glue get-databases --query "DatabaseList[].Name"`. For
  CUR dashboards it is the CUR database (commonly `cid_cur`). For an `update` where the user
  names no database, read it from the dashboard's datasets:
  `aws quicksight describe-data-set --aws-account-id "<account-id>" --data-set-id "<id>" --query "DataSet.PhysicalTableMap.*.RelationalTable.Schema"`
  (the dataset ids are in `describe-dashboard`'s `Version.DataSetArns`). A non-recursive
  `update` reuses existing datasets, so a different database does not repoint them; cid-cmd
  only writes its `cid_parameters` view there. If a dataset the dashboard needs is missing,
  cid-cmd creates it in that database, so stop and ask in that case.
- `--athena-workgroup`: list with `aws athena list-work-groups`. **Pick one that already has a
  result OutputLocation configured**; check with
  `aws athena get-work-group --work-group "<wg>" --query "WorkGroup.Configuration.ResultConfiguration.OutputLocation"`.
  A workgroup with no result location (often `primary`) makes cid-cmd try to create one and
  then prompt for `athena-result-bucket`. CID deployments usually have a purpose-built `CID`
  workgroup. Use it.
- `--quicksight-datasource-id`: needed on a **fresh deploy** that must create datasets (an
  `update` of an existing dashboard whose datasets exist does not re-prompt for it). List with
  `aws quicksight list-data-sources --aws-account-id "<account-id>" --query "DataSources[?Type=='ATHENA'].[DataSourceId,Name]"`
  (commonly `CID-Athena`).
- `--cur-table-name`: the CUR Athena table or view the dashboard queries. List with
  `aws glue get-tables --database-name "<db>" --query "TableList[].Name"`. For CUR 2.0 it is
  typically the `cur2_view` view, not the raw `cur2` table.
- `--timezone <IANA tz>`: needed only when the operation migrates a dataset to the new
  QuickSight Data Preparation Experience (recursive updates, and some deploys). It sets the
  timezone of the datasets' **scheduled SPICE refresh**. Without it, an unattended run aborts
  with `Please set parameter timezone`. Do not invent a value. Resolve it in this order:
  1. **Reuse the existing schedule.** Get the dashboard's dataset ids from `describe-dashboard`
     (`Dashboard.Version.DataSetArns`, the last path segment of each ARN), then read their
     current schedule timezone (read-only):
     `aws quicksight list-refresh-schedules --aws-account-id "<acct>" --data-set-id "<dataset-id>" --query "RefreshSchedules[].ScheduleFrequency.Timezone"`.
     If the dashboard has no scheduled datasets (for a new deploy, for example), check the
     datasets of another CID dashboard in the same account and region, such as `cudos-v5` or
     `cost_intelligence_dashboard`. If they all agree, use that value and tell the user where
     it came from.
  2. **Otherwise ask the user.** Ask this together with the confirmation for the recursive
     update, not as a separate round trip. Offer the value from step 1 if one exists, or the
     shell's local timezone as a suggestion. Interactive cid-cmd defaults to the local
     timezone too (or, in Lambda/CloudShell, to a timezone mapped from the region).
  If the schedules disagree, list the values and ask. Don't pick one yourself.

Verified live end-to-end: with the full chain above (plus a healthy CUR view), a from-scratch
`deploy --dashboard-id cudos-v5` creates the datasets and lands a healthy dashboard
(`Health: healthy, Status: up to date`).

An **`update` of an already-deployed dashboard** typically needs only `--athena-database` and
`--athena-workgroup` (its datasource, datasets, and CUR table are already recorded). Supply
the full chain if an update prompts for more. **`delete` needs none of these**:
`delete --dashboard-id <id>` works unattended on its own.

## Tags, cost categories, and IAM principal (`--resource-tags`)

The CUR-based views of `cudos-v5`, `cost_intelligence_dashboard`, and `kpi_dashboard`
(`summary_view`, `resource_view`, `hourly_view`, and the KPI and CID views) carry a `tags_json`
column built from the tags the user selects. Interactive cid-cmd asks *"Select Cost Allocation
Tags and Cost Categories to be added to datasets"* on every deploy and recursive update. Do the
same, because the unattended defaults are silent and lossy:

- **CUR views:** with no terminal and no `--resource-tags`, cid-cmd takes the selection saved
  in the `cid_parameters` view of the `--athena-database` you pass. If nothing is saved there
  (a new deploy, older deployments, or a different database), it uses **no tags**. A recursive
  update then rebuilds the views without the tag columns, which breaks visuals and
  customizations that used them.
- **FOCUS views** (`focus_resource_view`, `focus_summary_view`): with no terminal and no
  explicit value, cid-cmd always uses **no tags**, and ignores the saved selection.
- The CUR views are **shared**. A tag change for one foundational dashboard applies to all
  three.

So for every **deploy** of these dashboards, and every **recursive update** of them, ask the
user which tags to use and pass the result explicitly.

### 1. Find the current selection (updates; read-only)

- Saved parameters (Athena query in the `--athena-database` and workgroup you'll pass):
  `SELECT context, parameter, value, date FROM "<db>"."cid_parameters" WHERE parameter LIKE '%resource-tags%' ORDER BY date DESC`.
  `context` is the dashboard id. cid-cmd uses the latest value for the dashboard, and falls
  back to the latest value for any dashboard.
- If nothing is saved, read the deployed view's SQL
  (`aws glue get-table --database-name "<db>" --name summary_view --query Table.ViewOriginalText`,
  which is base64-encoded Presto text after the `/* Presto View: ` prefix). The names inside
  `MAP_FROM_ENTRIES(ARRAY[('<name>', <column>), ...])` are the current selection. `'{}'`
  means no tags.

### 2. List what is available (new deploys, or when the user wants to change; read-only)

Do what cid-cmd does: scan the CUR table for the last 60 days (about 500,000 rows per scan, so
there is a small Athena cost) and show each key with its number of distinct values,
highest-cardinality first. Warn that every selected tag adds work to every refresh, so the user
should pick the strict minimum.

- **CUR 2.0** (map columns): run this once for each of `resource_tags`, `cost_category`, and,
  if the column exists, `tags`:

  ```sql
  SELECT key, COUNT(DISTINCT value) AS unique_values
  FROM (SELECT <column> FROM "<cur_db>"."<cur_table>"
        WHERE billing_period >= DATE_FORMAT(DATE_ADD('day', -60, CURRENT_DATE), '%Y-%m')
          AND line_item_usage_start_date > DATE_ADD('day', -60, CURRENT_DATE)
          AND cardinality(<column>) > 0
        LIMIT 500000) t
  CROSS JOIN UNNEST(<column>) AS t(key, value)
  GROUP BY key ORDER BY unique_values DESC
  ```

  The `tags` column holds three kinds of key, so show them as separate groups: `accountTag/…`
  (Account Tags), `userAttribute/…` (User Attributes), and `iamPrincipal/…` (IAM Principal
  Tags). If the column is missing, say that these need the "Tag" column in the CUR 2.0 data
  export.
- **IAM principal** (CUR 2.0, for Amazon Bedrock and other per-caller attribution): if the
  table has a `line_item_iam_principal` column with at least one non-empty value in the same
  60-day window, offer it as a choice too. The Data Exports stack adds it when
  `EnableIAMPrincipalData=yes`.
- **CUR 1.0** (flat columns): the choices are the table's `resource_tags_*` and
  `cost_category_*` columns (`aws glue get-table ... --query "Table.StorageDescriptor.Columns[].Name"`).
  No scan is needed.

### 3. Pass the selection with cid-cmd's names

Pass a comma-separated list of names (not column expressions) with `--resource-tags`. Use
`--resource-tags ""` for no tags. cid-cmd names each choice like this (unknown names are
skipped with a `WARNING`):

| Source | Example | Name to pass |
|---|---|---|
| CUR 2.0 user tag | `resource_tags['user_owner']` | `tag_owner` |
| CUR 2.0 AWS tag | `resource_tags['aws_createdBy']` | `tag_aws_createdBy` |
| Cost category | `cost_category['team']` | `cost_category_team` |
| Account tag | `tags['accountTag/env']` | `account_tag_env` |
| User attribute | `tags['userAttribute/dept']` | `user_attribute_tag_dept` |
| IAM principal tag | `tags['iamPrincipal/project']` | `iam_principal_tag_project` |
| IAM principal | `line_item_iam_principal` | `iam_principal` |
| CUR 1.0 column | `resource_tags_user_owner` | `tag_user_owner` |
| CUR 1.0 cost category | `cost_category_team` | `cost_category_team` |

Characters other than letters, digits, and `_` become `_` (for example `tag_cost-center` →
`tag_cost_center`).

For `focus-dashboard`, pass the FOCUS views' own parameters:
`--view-focus-resource-view-resource-tags "<names>"` and
`--view-focus-summary-view-resource-tags "<names>"`. FOCUS tag keys come from the `tags` map
of `focus_consolidation_view`, and are named `tag_<key>`. List them with the query in the
`resource_tags` parameter of those views in the upstream `dashboards/focus/focus.yaml`. View
parameters follow the pattern `--view-<view name with dashes>-<parameter with dashes>`
(checked in cid-cmd 4.4.21).

## Update options

Pattern:

```bash
cid-cmd --profile "<profile>" --region_name "<region>" -y update [options]
```

- `--force`: lets the dashboard picker offer up-to-date dashboards when `--dashboard-id` is
  omitted. With `--dashboard-id`, `update` republishes the dashboard even when it is up to
  date, with or without `--force`. Do not update a dashboard that `status` reports as up to
  date unless the user asks for a rebuild or a CUR 2.0 migration.
- `--recursive`: update datasets and Athena views. **Can overwrite customizations**; always
  follow "Recursive updates" in SKILL.md. Every
  `update`, recursive or not, also rewrites cid-cmd's own `cid_parameters` view (its parameter
  store) in the `--athena-database`.
- To force a full rebuild, pass `--force --recursive --on-drift override` (see "Recursive updates" in SKILL.md).
- `--dashboard-id <id>`: optional; omit only when unattended selection is acceptable.
- `--on-drift <show|override>`, `--theme <...>`, `--currency <...>`, and `--rls` /
  `--rls-dataset-id` work exactly as in deploy.
- `--share-with-account <yes|no>` (added in cid-cmd 4.4.18): `yes` shares the
  dashboard with everyone in the account, `no` revokes that; omitted leaves sharing
  unchanged. Mutating sharing change; confirm first, and use it only when the user asks for
  account-wide sharing. It is a full `update`, not a sharing-only toggle: the dashboard is
  first brought to the latest version and the sharing change is applied last. Confirm it as
  an update, and pass the parameter chain below.
- Unattended `update` also needs the Athena parameter chain above (`--athena-database`,
  `--athena-workgroup` with a result location, plus the datasource and CUR parameters if it
  prompts). The same prompt-then-abort behavior applies.

## Deploy-time guidance for the user

- Advanced dashboards (the "Needs Data Collection" column in
  `references/dashboard-catalog.md`) show no data until the Data Collection pipeline has run.
  Set that up first.
- `cora` needs AWS Cost Optimization Hub enabled **and** a Cost Optimization Hub data export.
- `focus-dashboard` needs a FOCUS data export.
- A recursive `update`, and any deploy or update that migrates a dataset to the **new
  QuickSight Data Preparation Experience**, prompts for **`--timezone <IANA tz>`** for the
  dataset refresh schedule and aborts unattended without it. Include `--timezone` for those
  paths. The migration message "will delete and recreate this dataset, temporarily breaking
  the dashboards listed above" means recursive updates briefly break dashboards that share
  those datasets until each is re-updated. Verified live.
- Recursive rebuilds are slow (they can exceed 15 minutes). Run them in the background.

## Stale CUR view (`VIEW_IS_STALE`)

`VIEW_IS_STALE ... stored view column count (N) does not match` on the CUR view means schema
drift between the view and its underlying table: the CUR export gained columns but the view's
stored schema was not regenerated (verified live: a view frozen at 73 columns over a
130-column table). During a fresh `deploy` this is a **hard blocker**, because creating the
required datasets reads the view, even though the earlier optional tag-scan queries only
logged `ERROR - ... Will continue without.`

**Fix: recreate the view so its stored schema re-derives from the current table.** Confirm
with the user first, because this replaces an Athena view.

1. Read the view's exact definition:
   `aws glue get-table --database-name "<db>" --name "<view>" --query Table.ViewOriginalText`.
   Presto views base64-encode the SQL after the `Presto View:` marker. A CID CUR view is
   simply `SELECT * FROM "<cur-export-db>"."<cur-table>"` (for example
   `SELECT * FROM "cid_data_export"."cur2"`).
2. Run `CREATE OR REPLACE VIEW <view> AS <that same select>` in Athena against a workgroup
   with a result location.
3. Re-probe with `SELECT COUNT(*) FROM <db>.<view>` and confirm the Glue stored column count
   now matches the table.
4. Redeploy.

`update --recursive` also rebuilds datasets and Athena views, but `--recursive` is a flag of
**`update`, not `deploy`**. Passing it to `deploy` is rejected with `Unknown extra argument`.
