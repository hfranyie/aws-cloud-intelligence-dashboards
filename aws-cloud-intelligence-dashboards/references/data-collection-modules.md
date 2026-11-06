# Data Collection modules

Module key -> exact CloudFormation parameter name (`IncludeXModule`), what each collects,
which dashboards it powers, and which cross-account read role it needs. Values match the
pinned Data Collection templates (`v3.14.8`).

**Use the 24-module list below** wherever an abbreviated module table disagrees. Some
abbreviated tables collapse `identity_center` and `kiro_user_activity` into a catch-all and
omit `rightsizing`.

## How these parameters are used

- On the **Data Collection** stack (`deploy-data-collection.yaml`), enabling a module means
  passing `<IncludeXModule>=yes`. One parameter per module key you want.
- On the **read-permissions** stack (`deploy-data-read-permissions.yaml`), you pass
  `<IncludeXModule>=yes` only for modules that actually need a cross-account read role
  (management and/or linked). See the "Permissions stack" rule below.
- The module key (left column) is the short name a user says; the CFN parameter (second
  column) is the literal CloudFormation `ParameterKey`.

## Module table (24)

| Module key | CFN parameter | Collects | Powers dashboards | Cross-account role | Extra params |
|---|---|---|---|---|---|
| `trusted_advisor` | `IncludeTAModule` | AWS Trusted Advisor recommendations | `ta-organizational-view` | linked | |
| `compute_optimizer` | `IncludeComputeOptimizerModule` | AWS Compute Optimizer recommendations | `compute-optimizer-dashboard` | management | |
| `cost_anomaly` | `IncludeCostAnomalyModule` | AWS Cost Explorer cost anomalies | `aws-cost-anomalies` | management | |
| `rightsizing` | `IncludeRightsizingModule` | AWS Cost Explorer rightsizing recommendations (deprecated upstream in favor of Cost Optimization Hub data exports) | — | management | |
| `support_cases` | `IncludeSupportCasesModule` | AWS Support case history | `support-cases-radar` | linked | |
| `ecs_chargeback` | `IncludeECSChargebackModule` | Amazon ECS task/cluster cost allocation data | — | linked | |
| `inventory` | `IncludeInventoryCollectorModule` | Resource inventory (AMIs, EBS volumes/snapshots, EC2, RDS, ElastiCache, OpenSearch, Lambda, EKS) — pulls in pricing data automatically | `graviton-savings`, `extended-support-cost-projection` | linked | |
| `rds_utilization` | `IncludeRDSUtilizationModule` | RDS CloudWatch utilization metrics for chargeback | — | linked | |
| `org_data` | `IncludeOrgDataModule` | AWS Organizations account/OU/tag metadata | — | none | |
| `budgets` | `IncludeBudgetsModule` | AWS Budgets | `aws-budgets` | linked | |
| `transit_gateway` | `IncludeTransitGatewayModule` | AWS Transit Gateway CloudWatch metrics for chargeback | — | linked | |
| `backup` | `IncludeBackupModule` | AWS Backup/Restore/Copy job history | — | management | |
| `health_events` | `IncludeHealthEventsModule` | AWS Health organizational event notifications | `health-events-dashboard` | management | |
| `license_manager` | `IncludeLicenseManagerModule` | AWS License Manager licenses and grants | — | management | |
| `quicksight` | `IncludeQuickSightModule` | Amazon QuickSight user/group metadata (Data Collection account only) | — | none | |
| `service_quotas` | `IncludeServiceQuotasModule` | AWS Service Quotas current usage/limits | — | management + linked | |
| `euc_utilization` | `IncludeEUCUtilizationModule` | Amazon WorkSpaces CloudWatch utilization metrics | `euc-dashboard` | linked | `EUCAccountIDs` |
| `resilience_hub` | `IncludeResilienceHubModule` | AWS Resilience Hub assessment results | `resiliencevue` | linked | |
| `identity_center` | `IncludeIdentityCenterModule` | AWS IAM Identity Center users/groups/assignments | — | management | |
| `marketplace` | `IncludeMarketplaceModule` | AWS Marketplace agreements and invoicing schedules | `aws-marketplace` | linked | |
| `kiro_user_activity` | `IncludeKiroUserActivityModule` | Per-user Kiro AI coding assistant usage (reads from customer-provided source buckets) | `kiro-user-activity` | none | `KiroSourceBuckets` |
| `reference` | `IncludeReferenceModule` | Reference data (e.g. RDS end-of-life dates) needed by other modules/dashboards | — | none | |
| `aws_feeds` | `IncludeAWSFeedsModule` | AWS What's New / blog / security bulletin feeds | `aws-feeds` | none | |
| `isv_feeds` | `IncludeISVFeedsModule` | Independent software vendor feeds | — | none | |

## Pricing is automatic (no toggle)

There is deliberately **no** standalone pricing module. Pricing collection has no
user-facing parameter — it is pulled in automatically whenever `inventory`,
`rds_utilization`, or `euc_utilization` is enabled. Upstream models it as a derived OR
condition, not a parameter, so do not look for an `IncludePricingModule`.

## Permissions-stack rule (cross-account roles)

Modules with cross-account role `none` — `org_data`, `quicksight`, `reference`, `aws_feeds`,
`isv_feeds`, and `kiro_user_activity` — run entirely within the Management or Data
Collection account and have **no parameter** on the read-permissions stack
(`deploy-data-read-permissions.yaml`). Passing one of those to the permissions step is an
**error**, not a silent no-op, because the template genuinely has no such parameter. Enable
those modules only on the Data Collection stack itself.

Modules marked `management`, `linked`, or `management + linked` do have a cross-account read
parameter and should be passed to the permissions stack so the Data Collection account can
read from the organization.
