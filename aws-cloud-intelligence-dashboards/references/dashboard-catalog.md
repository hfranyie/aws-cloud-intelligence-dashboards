# Dashboard catalog

Canonical dashboard IDs, friendly-name aliases, and resolution rules, as of cid-cmd 4.4.19.

cid-cmd bundles only 6 of these dashboards (`cudos-v5`, `cudos`, `cost_intelligence_dashboard`,
`kpi_dashboard`, `trends-dashboard`, `compute-optimizer-dashboard`). It fetches the rest from
the upstream catalog on GitHub at run time, so that list can change between cid-cmd
releases. The table below was checked against that catalog when this skill was moved to 4.4.19.

## Contents

- [Resolution rule](#resolution-rule)
- [Canonical dashboards (32)](#canonical-dashboards-32)
- [Friendly-name aliases (45)](#friendly-name-aliases-45)

Use this file to:

- turn a user's phrase ("CUDOS", "trusted advisor", "cost anomalies") into the canonical
  `--dashboard-id` value `cid-cmd` expects, and
- decide whether a dashboard needs the advanced Data Collection pipeline before it will show
  data.

## Resolution rule

`cid-cmd deploy/update/delete/status` take `--dashboard-id <canonical-id>`. To resolve a
user phrase to a canonical id:

1. Normalize: trim, lowercase, and collapse internal whitespace to single spaces.
2. If the normalized string matches an alias key in the alias table, use its canonical id.
3. Otherwise, if it matches a canonical id, use that.
4. Otherwise, pass the user's value through unchanged and let `cid-cmd` return the
   authoritative "unknown dashboard-id" error. This catalog is a convenience layer, not a
   gate.

Note the one deliberate collision: the friendly name `cudos` resolves to `cudos-v5` (the
current version), while the exact canonical id `cudos` is the deprecated v4. A user who
truly wants v4 must ask for the exact id `cudos`.

## Canonical dashboards (32)

| Canonical dashboardId | Display name | Category | Needs Data Collection | Notes |
|---|---|---|:---:|---|
| `cudos-v5` | CUDOS Dashboard v5 | Foundational | no | |
| `cost_intelligence_dashboard` | Cost Intelligence Dashboard | Foundational | no | |
| `kpi_dashboard` | KPI Dashboard | Foundational | no | |
| `cudos` | CUDOS Dashboard (v4) | Deprecated | no | Deprecated. Deploy `cudos-v5` instead. |
| `ta-organizational-view` | Trusted Advisor Organizational View (TAO) | Advanced | yes | |
| `compute-optimizer-dashboard` | Compute Optimizer Dashboard | Advanced | yes | |
| `cora` | CORA - Cost Optimization Recommended Actions | Additional | no | Requires AWS Cost Optimization Hub enabled + a Cost Optimization Hub data export. |
| `graviton-savings` | Graviton Savings Dashboard | Advanced | yes | |
| `graviton-opportunities` | Graviton Opportunities Dashboard | Advanced | yes | Deprecated. Renamed to `graviton-savings`. |
| `health-events-dashboard` | Health Events Dashboard | Advanced | yes | |
| `extended-support-cost-projection` | Extended Support Cost Projection | Advanced | yes | |
| `aws-cost-anomalies` | AWS Cost Anomalies Dashboard | Advanced | yes | |
| `aws-budgets` | AWS Budgets Dashboard | Advanced | yes | |
| `aws-feeds` | AWS News Feeds | Advanced | yes | |
| `support-cases-radar` | AWS Support Cases Radar | Advanced | yes | Requires Business, Enterprise On-Ramp, or Enterprise support plan. |
| `dc-monitor` | Data Collection Monitor Dashboard | Advanced | yes | |
| `euc-dashboard` | Amazon End User Computing (EUC) Dashboard | Advanced | yes | |
| `resiliencevue` | ResilienceVue Dashboard | Additional | yes | |
| `cid-rls` | Row Level Security (RLS) management dashboard | Advanced | no | Manages RLS rules, not a data dashboard itself. |
| `trends-dashboard` | Trends Dashboard | Additional | no | |
| `datatransfer-cost-analysis-dashboard` | Data Transfer Cost Analysis Dashboard | Additional | no | |
| `aws-marketplace` | AWS Marketplace Single Pane of Glass (SPG) | Additional | no | |
| `amazon-connect-cost-insight-dashboard` | Amazon Connect Cost Insight Dashboard | Additional | no | |
| `scad-containers-cost-allocation` | SCAD Containers Cost Allocation Dashboard | Additional | no | |
| `focus-dashboard` | FOCUS Dashboard | Additional | no | Requires a FOCUS data export. |
| `media-services-insights` | Media Services Insights Hub | Advanced | no | |
| `sustainability-proxy-metrics` | Sustainability Proxy Metrics & Carbon Emissions Dashboard | Additional | no | |
| `pricing-change-analysis` | Pricing Change Analysis Dashboard | Advanced | no | |
| `kiro-user-activity` | Kiro User Activity Dashboard | Advanced | yes | |
| `droid` | DROID - Database Resilience, Operations & Insights | Advanced | yes | Added to the upstream catalog after cid-cmd 4.4.19 shipped. Needs the Data Collection RDS module (`IncludeRDSModule`, which requires the Inventory and Reference modules). That module first appears in Data Collection `v3.14.11`, newer than the pinned `v3.14.8`. Also reads CUR 2.0. |
| `unit-cost-dashboard` | Unit Cost Dashboard | Custom | no | Not in the default catalog; deploy with `--resources` pointing at its yaml file. |
| `shard` | SHARD - Security Hub Analytics and Reporting Dashboard | Security | no | Not in the default catalog; deploy with `--resources` pointing at its yaml file. Requires AWS Security Hub data collection. |

Category legend: Foundational | Advanced | Additional | Deprecated | Custom | Security.

"Needs Data Collection = yes" means the dashboard's data comes from the advanced Data
Collection pipeline (`deploy-data-collection.yaml`) rather than CUR alone. Deploy the Data
Collection stack and let it run at least once before expecting data. See
`references/data-collection-modules.md` for the modules each dashboard needs.

## Friendly-name aliases (45)

Each alias key is matched case-insensitively, whitespace-normalized (see resolution rule).

| Alias | Canonical dashboardId |
|---|---|
| `cudos` | `cudos-v5` |
| `cudos v5` | `cudos-v5` |
| `cudos5` | `cudos-v5` |
| `cid` | `cost_intelligence_dashboard` |
| `cost intelligence` | `cost_intelligence_dashboard` |
| `cost intelligence dashboard` | `cost_intelligence_dashboard` |
| `kpi` | `kpi_dashboard` |
| `trusted advisor` | `ta-organizational-view` |
| `tao` | `ta-organizational-view` |
| `trusted advisor organizational` | `ta-organizational-view` |
| `compute optimizer` | `compute-optimizer-dashboard` |
| `graviton` | `graviton-savings` |
| `graviton savings` | `graviton-savings` |
| `health events` | `health-events-dashboard` |
| `health` | `health-events-dashboard` |
| `extended support` | `extended-support-cost-projection` |
| `cost anomalies` | `aws-cost-anomalies` |
| `cost anomaly` | `aws-cost-anomalies` |
| `budgets` | `aws-budgets` |
| `feeds` | `aws-feeds` |
| `news feeds` | `aws-feeds` |
| `support cases` | `support-cases-radar` |
| `support cases radar` | `support-cases-radar` |
| `data collection monitor` | `dc-monitor` |
| `euc` | `euc-dashboard` |
| `workspaces` | `euc-dashboard` |
| `resilience` | `resiliencevue` |
| `resilience hub` | `resiliencevue` |
| `rls` | `cid-rls` |
| `trends` | `trends-dashboard` |
| `data transfer` | `datatransfer-cost-analysis-dashboard` |
| `marketplace` | `aws-marketplace` |
| `connect` | `amazon-connect-cost-insight-dashboard` |
| `amazon connect` | `amazon-connect-cost-insight-dashboard` |
| `containers` | `scad-containers-cost-allocation` |
| `scad` | `scad-containers-cost-allocation` |
| `focus` | `focus-dashboard` |
| `media services` | `media-services-insights` |
| `media services insights` | `media-services-insights` |
| `sustainability` | `sustainability-proxy-metrics` |
| `carbon` | `sustainability-proxy-metrics` |
| `pricing change` | `pricing-change-analysis` |
| `pricing change analysis` | `pricing-change-analysis` |
| `kiro` | `kiro-user-activity` |
| `kiro user activity` | `kiro-user-activity` |
