# Changelog check before `update`

Read the dashboard's upstream changelog before every `cid-cmd update`. It tells you whether the
new version needs `--recursive`, `--force`, a newer cid-cmd, or a newer Data Collection stack.
cid-cmd does not read it for you.

## Contents

- [Why cid-cmd's own check is not enough](#why-cid-cmds-own-check-is-not-enough)
- [Procedure](#procedure)
- [Find the changelog file](#find-the-changelog-file)
- [What to look for](#what-to-look-for)
- [Reporting and confirming](#reporting-and-confirming)

## Why cid-cmd's own check is not enough

`update` without `--recursive` compares only the **major** version of
the deployed dashboard with the latest available one:

- **Major bump** (for example `4.x` -> `5.x`): cid-cmd asks `confirm-recursive`, *"This is a
  major update and require recursive action..."*. The default is **yes**, so with `-y` it
  **switches to recursive by itself** and can overwrite dataset and view customizations. A
  plain `update` is therefore not "safe" across a major version. Treat it as recursive and
  warn the user before you run it.
- **Minor or patch bump**: cid-cmd stays non-recursive even when the release changed Athena
  views or datasets. The dashboard is republished against the old views and datasets, and
  visuals can break or show stale logic. Only the changelog says so. Example: CUDOS `5.8.0`
  (a minor release on the 5.x line) says to run `cid-cmd update --dashboard-id cudos-v5
  --recursive` and requires cid-cmd 4.4.12 or later.

## Procedure

1. Run `status --dashboard-id "<id>" < /dev/null` (read-only). Note the deployed and latest
   versions from its `Version <deployed> -> <latest>` line. Skip dashboards that are up to
   date.
2. Fetch the changelog (read-only, no AWS call):

   ```bash
   curl -fsSL "https://raw.githubusercontent.com/aws-solutions-library-samples/cloud-intelligence-dashboards-framework/main/changes/CHANGELOG-<file>.md"
   ```

   Find `<file>` as described in [Find the changelog file](#find-the-changelog-file).

3. Find every version section **newer than the deployed version, up to and including the
   latest**. Version headings vary by file (`# CUDOS - 5.8.0`, `## TAO Dashboard v4.0.1:`,
   `## v3.1.0 (May 2026)`, `## KPI - 2.1.0`). Compare by version number, not by position:
   some files are not in descending order (for example EUC and Sustainability).
4. Scan those sections for the signals in [What to look for](#what-to-look-for). A signal in
   **any** skipped version applies to this update. A user going from 5.7 to 5.10 still needs
   what 5.8.0 required.
5. Report the findings and build the command (see
   [Reporting and confirming](#reporting-and-confirming)).

If the changelog can't be fetched (network blocked, 404) or has no section for the target
version, say so. Don't guess "no recursive needed". Tell the user the requirement is unknown
and let them choose between a plain and a recursive update.

## Find the changelog file

Changelogs are in the upstream repo's `changes/` folder, named `CHANGELOG-<short name>.md`.
The short name often differs from the dashboard id (`hed`, `cod`, `tao`, `cid`, `pca`,
`scad-cca`, and others), so discover it. List each file with its first heading, which names
the dashboard:

```bash
R="https://raw.githubusercontent.com/aws-solutions-library-samples/cloud-intelligence-dashboards-framework/main"
for f in $(curl -fsSL "https://api.github.com/repos/aws-solutions-library-samples/cloud-intelligence-dashboards-framework/contents/changes" | jq -r '.[].name | select(startswith("CHANGELOG-"))'); do
  echo "$f | $(curl -fsSL "$R/changes/$f" 2>/dev/null | grep -m1 -E '^#')"
done
```

Pick the file whose heading matches the dashboard's display name (for example "What's new in
Health Events Dashboard (HED)" → `CHANGELOG-hed.md`). `CHANGELOG-cid.md` is the Cost
Intelligence Dashboard, not the whole framework. If no file matches, the dashboard has no
changelog. Fall back to cid-cmd's major-version check, and tell the user there is no changelog
to check against.

## What to look for

Upstream doesn't use a fixed format. Requirements usually sit in an `**Important:**` or
`NOTE:` line, or in a suggested `cid-cmd update ...` command block. Match case-insensitively:

| Signal | Typical wording | Effect on the update |
|---|---|---|
| Recursive needed | `--recursive`, "recursive update", "requires recursive", "An Athena view changed", "dataset changes", "updates the definition of the ... view" | Add `--recursive`. |
| Force needed | `--force`, "forced update", "forced and recursive" | Add `--force`. |
| Drift override | `--on-drift override` | Every recursive update gets it anyway, after the warning (see "Recursive updates" in SKILL.md). |
| Minimum cid-cmd | "requires cid-cmd v4.4.12", "cid-cmd v0.2.27+" | Compare with the installed `cid-cmd --help` version. If it is older, upgrade cid-cmd first (local tooling change, no AWS change). |
| Minimum Data Collection | "requires the data collection version 3.2.0+", "Data Collection Framework beyond version 3.14.4" | Check the deployed Data Collection stack version (see [cloudformation-templates.md](cloudformation-templates.md)). If it is older, the stack must be updated first. Stop and tell the user. |
| Data-prep migration | "migrated to the new data preparation experience" | Warn that custom dataset changes are lost. Advise a copy first. |
| Post-update step | "SPICE refresh", "provide a path to s3 folder" | Tell the user. A refresh is a separate mutating step (`cid-cmd refresh`). An extra parameter may cause a `Please set parameter` abort, see [deploy-parameters.md](deploy-parameters.md). |
| Major version | The deployed and latest versions differ in the first number | cid-cmd goes recursive on its own under `-y`. Say so even if the changelog is silent. |

Use the suggested command in the changelog as a hint, not as something to copy. Some upstream
commands have typos (for example `--dashboards-id` in the COD changelog, or a missing `update`
in Media Services). Always build the command from the skill's own template with the full
unattended parameter chain.

## Reporting and confirming

Before the update, show the user one line per dashboard:

```
<id>: <deployed> -> <latest>; changelog says: <recursive|force|cid-cmd >= X|DC >= Y|nothing special>; plan: update [--force] [--recursive]
```

- If any skipped version needs `--recursive`, or the update is a major bump, follow
  "Recursive updates" in SKILL.md for the warning, the confirmation, `--on-drift override`, the
  timezone, and what to do if the user declines.
- Cite the changelog version(s) that triggered each flag, so the user can check the source.
