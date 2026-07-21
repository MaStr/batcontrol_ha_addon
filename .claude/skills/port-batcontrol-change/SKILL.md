---
name: port-batcontrol-change
description: Port a change from the upstream MaStr/batcontrol repo into this Home Assistant add-on repository. Use when an upstream PR, commit, or release adds/changes/removes configuration parameters, when syncing the development add-on with upstream main, or when promoting a new upstream release into the stable add-on.
---

# Port a batcontrol change into the HA add-on

Upstream repo: `MaStr/batcontrol` (Python application).
This repo only packages it — porting a change means updating the add-on manifests and docs,
never application code.

## Step 1 — Identify what changed upstream

Gather the upstream change set (PR number, commit range, or release tag). The authoritative
source for configuration is the upstream reference config:

```
MaStr/batcontrol: config/batcontrol_config_dummy.yaml
```

Diff it against the add-on's `options:` block. If a local clone of batcontrol exists (e.g.
`../batcontrol`), diff directly; otherwise fetch the file from GitHub (raw URL or MCP
`get_file_contents`). Classify each difference:

- **New/changed/removed config parameter** -> Steps 2-4 (config.yaml), 5 (DOCS), 6 (CHANGELOG), 7 (version)
- **Behavior change, no config change** -> Steps 5-7 only (DOCS if user-facing, CHANGELOG, version)
- **Build-relevant change** (entrypoint_ha.sh, pyproject, file moves in upstream `config/`) -> also check both Dockerfiles

## Step 2 — Pick the target add-on

- `batcontrol-development/` — **default target.** Tracks upstream `main`; every merged upstream
  change is portable immediately.
- `batcontrol/` (stable) — **only** when upstream has tagged a release. Its `version` must be
  set to exactly that release tag (the Dockerfile downloads
  `releases/download/<version>/batcontrol-<version>-py3-none-any.whl`). Never point stable at
  an untagged state.

When promoting a release to stable: replay all accumulated development changes (options,
schema, DOCS.md) and move the "in Development" changelog content into a release section of
the stable CHANGELOG.md.

## Step 3 — Update `options:` in config.yaml

- Insert the new key at the **same position and nesting** as in upstream
  `batcontrol_config_dummy.yaml`, with a working example value and the upstream comment.
- Parameters that most users leave at default: add as a **commented-out line** with default
  noted (see `min_grid_charge_soc` or `market_price_refresh_time` for the pattern).
- Do NOT expose keys the add-on manages itself: `logfile_path`, `logfile_enabled` (log path is
  fixed to `/data/batcontrol.log` by `entrypoint_ha.sh`).
- Removed/renamed upstream keys: remove/rename here too and record it as a breaking change in
  the CHANGELOG.

## Step 4 — Update `schema:` in config.yaml

Every `options:` key needs a `schema:` entry at the same path. Mapping from upstream value
types to HA schema syntax:

| Upstream value | Schema entry |
|---|---|
| free string | `str` |
| secret/token/password | `password` |
| true/false | `bool` |
| integer (bounded) | `int` / `int(0,23)` |
| decimal (bounded) | `float` / `float(0,1)` |
| fixed choice set | `list(default\|next)` |
| optional anything | append `?`, e.g. `float(0,1)?` |
| list of values | YAML list with one schema item, e.g. `- str` |

Rules of thumb:
- New parameters are almost always optional (`?`) so existing user configs stay valid —
  a non-optional new key breaks every installed instance on update.
- Carry the upstream comment onto the schema line (this repo documents both blocks).
- SoC-style fractions -> `float(0,1)`, hours -> `int(0,23)`, watts/Wh -> `float(0,)`/`int(0,)`.

## Step 5 — Update DOCS.md (if user-facing)

`DOCS.md` is rendered in the HA add-on UI. Add/adjust the parameter under its section
(`### \`battery_control:\``-style headings, backtick-quoted keys, short prose). For deep
detail, link to https://mastr.github.io/batcontrol/ instead of duplicating it.

## Step 6 — Update CHANGELOG.md

Development add-on: keep/extend the top section `# Release X.Y.Z - in Development`, grouped
by category (`### New Features`, `### Documentation`, ...), one bullet per change with the
upstream PR number, e.g.:

```markdown
- **Dynamic Network Fees - Section 14a EnWG** (#364): short user-facing summary.
```

Stable add-on: changes only arrive with a release; add a `# Release X.Y.Z - Released on DD.MM.YYYY`
section summarizing everything since the previous release.

## Step 7 — Bump the version

In the target `config.yaml`:
- Development: increment the dev suffix (`0.8.1dev3` -> `0.8.1dev4`).
- Stable: set to the upstream release tag (`0.8.0` -> `0.8.1`).

Without a version bump HA never offers the update. Commit message convention:
`chore(dev): bump version to X.Y.ZdevN, update changelog`.

## Step 8 — Translations (rarely)

`translations/en.yaml` only names/describes **top-level** option groups. Touch it only when a
new top-level block was added in Step 3.

## Step 9 — Verify

1. YAML parses:
   `python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" <addon>/config.yaml`
2. Every `options:` key has a `schema:` counterpart at the same path (and vice versa).
3. New keys are optional (`?`) unless a break is intended and documented.
4. Version bumped, CHANGELOG entry present.
5. Optional: CI (`lint.yaml`) runs `frenck/action-addon-linter` on push — it catches
   schema/options mismatches.

## Checklist

```
[ ] Upstream change identified (PR # / tag)
[ ] Target chosen (development vs. stable)
[ ] options: updated (position + comments match upstream dummy config)
[ ] schema: updated (optional `?` for new keys)
[ ] DOCS.md updated (if user-facing)
[ ] CHANGELOG.md entry with upstream PR #
[ ] version bumped in config.yaml
[ ] translations/en.yaml (only for new top-level groups)
[ ] YAML valid, options/schema consistent
```
