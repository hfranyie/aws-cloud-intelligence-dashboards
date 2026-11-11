# What is on a dashboard

Use this file when the user asks what a dashboard shows, which sheets or visuals it has,
whether it covers a service or metric, or which dataset or Athena view feeds a visual. Answer
from the two sources below, not from memory. Dashboards change between releases, and memory
is often out of date.

## Contents

- [Sources and when to use each](#sources-and-when-to-use-each)
- [Find the sources for a dashboard](#find-the-sources-for-a-dashboard)
- [Read the docs page](#read-the-docs-page)
- [Outline the definition YAML](#outline-the-definition-yaml)
- [Answering](#answering)

## Sources and when to use each

All of these are read-only and need no AWS credentials.

1. **The dashboard's docs page** (AWS documentation). Start here for "what is this dashboard
   for", "what does it show", the sheets and their purpose, prerequisites, and the data it
   needs. Written for users, and reviewed.
2. **The definition YAML in the upstream repo**
   (`aws-solutions-library-samples/cloud-intelligence-dashboards-framework`, `dashboards/`).
   This is the source of truth for exact sheet names and visual titles, the datasets the
   dashboard uses, and their Athena views. Use it for "is there a visual for X", "what's on
   sheet Y", and "which dataset or view feeds Z". It reflects the **latest** release, which
   may be newer than what the user has deployed (compare with `status`).

If the two disagree, say so. The YAML is what cid-cmd deploys, and the docs page can lag
behind it.

## Find the sources for a dashboard

Discover both sources at run time instead of using a fixed list, because upstream adds and
renames dashboards. Work in a scratch directory. Every step is read-only.

### Resource file (upstream repo)

Run the dashboard listing in
[dashboard-catalog.md](dashboard-catalog.md#which-dashboards-can-i-deploy-that-i-dont-have-yet)
(step 1), and take the resource file on the line for the `dashboardId`. If the id isn't listed,
the dashboard isn't in the default catalog (for example `shard` and `unit-cost-dashboard`,
which are deployed with `--resources`). List the repo's `dashboards/` folders
(`curl -fsSL "https://api.github.com/repos/aws-solutions-library-samples/cloud-intelligence-dashboards-framework/contents/dashboards" | jq -r '.[].name'`)
and look in the one that matches the name.

In the resource file, the `dashboards:` entry with that `dashboardId` says where the sheets
and visuals are defined:
- `file:` names a separate `*-definition.yaml` in the same folder;
- `data:` holds the definition inline;
- `templateId:` means it is built from a QuickSight template and the repo has no definition.
  Use the docs page. The resource file still lists the datasets and views.

### Docs page (AWS documentation)

The dashboards index page links to every dashboard's page:

```bash
curl -fsSL "https://docs.aws.amazon.com/guidance/latest/cloud-intelligence-dashboards/dashboards.html" \
  | grep -oE '<a [^>]*href="\./[a-z0-9-]+\.html"[^>]*>[^<]+' \
  | sed -E 's/.*href="\.\/([^"]+)"[^>]*>/\1 | /' | grep -v -E '\| (Deploy|Details)$' | sort -u
```

Pick the page whose link text matches the dashboard's `name:` in the resource file (for
example "Health Events Dashboard" → `health-events-dashboard.html`). Page URLs are relative to
`https://docs.aws.amazon.com/guidance/latest/cloud-intelligence-dashboards/`. Some dashboards
share a page: CUDOS, Cost Intelligence, and KPI are on `cudos-cid-kpi.html`, and CUDOS visuals
are listed on `cudos-visual-reference.html`. Row Level Security (`cid-rls`) is under
Customizations (`row-level-security.html`). If no page matches, say the dashboard has no docs
page and answer from the definition YAML.

## Read the docs page

Fetch the page with your web or documentation tool, for example `read_documentation` on the
AWS MCP server, or `curl -fsSL "<url>"`. Pages are HTML, so read the main content and skip
the navigation.

## Outline the definition YAML

Definition files are large (CUDOS v5 is about 3.4 MB). Don't read one into context. Download
it to a scratch directory with `curl`, then extract a short outline: the datasets, then each
sheet with its visual titles and types. Python parses only the local files, so the macOS
python.org `CERTIFICATE_VERIFY_FAILED` issue doesn't apply. PyYAML is already installed with
cid-cmd, so run this with the `python3` from the same environment as cid-cmd:

```bash
cd "$(mktemp -d)"
R="https://raw.githubusercontent.com/aws-solutions-library-samples/cloud-intelligence-dashboards-framework/main"
RES="<resource file>"   # found above, e.g. dashboards/health-events/health-events.yaml
ID="<dashboardId>"      # e.g. health-events-dashboard
curl -fsSL "$R/$RES" -o resource.yaml
# "file" definitions: also download the *-definition.yaml the resource file names
DEF="$(python3 -I -c 'import sys,yaml; r=yaml.safe_load(open("resource.yaml")); print(next((d.get("file") or "") for d in r["dashboards"].values() if d.get("dashboardId")==sys.argv[1]))' "$ID")"
[ -n "$DEF" ] && curl -fsSL "$R/${RES%/*}/${DEF#./}" -o "${DEF##*/}"
python3 -I - "$ID" <<'PY'
import sys, os, re, html, yaml
dash_id = sys.argv[1]
res = yaml.safe_load(open('resource.yaml'))
dash = next(d for d in res['dashboards'].values() if d.get('dashboardId') == dash_id)
if dash.get('file'):
    definition = yaml.safe_load(open(os.path.basename(dash['file'])))
elif dash.get('data'):
    definition = dash['data'] if isinstance(dash['data'], dict) else yaml.safe_load(dash['data'])
else:
    sys.exit(f"{dash_id}: built from QuickSight template {dash.get('templateId')}; no definition in the repo")
def title(v):
    t = (v.get('Title') or {}).get('FormatText') or {}
    return ' '.join(html.unescape(re.sub(r'<[^>]+>', ' ', t.get('PlainText') or t.get('RichText') or '')).split())
print('Datasets:', ', '.join(d['Identifier'] for d in definition.get('DataSetIdentifierDeclarations', [])))
for s in definition.get('Sheets', []):
    print(f"\n## {s.get('Name')}")
    for v in s.get('Visuals', []):
        kind, body = next(iter(v.items()))
        if title(body): print(f"- {title(body)} ({kind})")
PY
```

To find the **Athena view behind a dataset**, read that dataset's entry in `resource.yaml`
(`datasets:` section) for its `dependsOn.views`, and the view's SQL in the `views:` section.
Search for the names with `grep`; don't read the whole file.

For a question about one visual's fields or calculation, `grep -n` the visual's title in the
definition file and read only the lines around it.

## Answering

- Name the dashboard by its canonical id, and cite the source with its URL: the docs page,
  the definition YAML, or both.
- Summarize. Group visuals by sheet, and quote exact sheet and visual names when the user
  asks whether something exists.
- Say which version the YAML describes: the latest upstream release. If the dashboard is
  deployed and `status` shows an older version, tell the user their copy may not have it yet
  (see [update-changelog-check.md](update-changelog-check.md) for what changed).
- If the dashboard needs Data Collection or a specific data export to show data, say so (see
  [dashboard-catalog.md](dashboard-catalog.md)).
- For template-based dashboards, say that the outline comes from the docs page only.
