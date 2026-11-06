# AWS Cloud Intelligence Dashboards Agent Skill (preview)

Preview of the **`aws-cloud-intelligence-dashboards`** Agent Skill. It's being contributed to the
[AWS Agent Toolkit](https://github.com/aws/agent-toolkit-for-aws) and isn't published there yet.

The skill lets an AI agent deploy and manage the
[Cloud Intelligence Dashboards](https://docs.aws.amazon.com/guidance/latest/cloud-intelligence-dashboards/getting-started.html)
(CUDOS, Cost Intelligence, KPI, Trusted Advisor (TAO), CORA, Graviton, Health Events, FOCUS and more) in Amazon
QuickSight in your own AWS account. It uses the `cid-cmd` CLI (pinned to 4.4.19) and the CID CloudFormation
templates. It covers status, deploy, update, delete, migration to CUR 2.0, sharing, the CUR 2.0 / FOCUS / Cost
Optimization Hub data exports, and the Data Collection pipeline.

The skill is text only: [`SKILL.md`](aws-cloud-intelligence-dashboards/SKILL.md) plus 6 reference files. There's no
executable code.

## Try it

Copy the skill folder into your agent's skills directory. Keep the folder name `aws-cloud-intelligence-dashboards`.

```bash
git clone https://github.com/hfranyie/aws-cloud-intelligence-dashboards.git
mkdir -p ~/.kiro/skills            # or your agent's skills folder, e.g. ~/.claude/skills
cp -R aws-cloud-intelligence-dashboards/aws-cloud-intelligence-dashboards ~/.kiro/skills/
```

Then ask your agent things like:

- "Which Cloud Intelligence Dashboards are installed in my account, and are any out of date?"
- "Deploy the CUDOS dashboard in us-east-1."
- "What's the difference between CUDOS, Cost Intelligence, and KPI?"

The skill acts with **your own AWS credentials**, so use a test account. It asks for confirmation before any change
to your account, and warns before destructive actions such as deleting a dashboard.

## Requirements

- Python 3.10+. The skill installs `cid-cmd==4.4.19` in a scratch virtual environment if it isn't already installed.
- AWS credentials with permissions for QuickSight, Athena, Glue, S3 and CloudFormation.
- QuickSight Enterprise Edition with SPICE capacity in the target region.
- Outbound HTTPS to `raw.githubusercontent.com` (cid-cmd fetches its dashboard catalog from there).

## Status

Preview for review. Evaluated on Claude Sonnet 5.5 (100%), Claude Opus 5.5 (100%) and GPT-6.1 (93.3%), compared with
24–33% without the skill.
