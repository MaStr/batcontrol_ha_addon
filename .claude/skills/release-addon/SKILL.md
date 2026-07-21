---
name: release-addon
description: Promote a new upstream batcontrol release into the stable Home Assistant add-on and start the next development cycle. Use when upstream MaStr/batcontrol has tagged/published a release, when asked to release or update the stable add-on, or to bump the development add-on to the next dev cycle.
---

# Release the Home Assistant add-on

Two-part process: (A) promote the upstream release into the **stable** add-on
(`batcontrol/`), (B) open the next cycle in the **development** add-on
(`batcontrol-development/`).

Execution policy: perform edits and commits directly on a feature branch, but STOP and ask
for confirmation before pushing/opening the PR, and never merge to `main` yourself.

## Part A — Promote upstream release X.Y.Z into `batcontrol/`

### Step 1 — Verify the upstream release is usable

The stable Dockerfile downloads
`https://github.com/MaStr/batcontrol/releases/download/X.Y.Z/batcontrol-X.Y.Z-py3-none-any.whl`.
Confirm before touching anything:

1. Tag `X.Y.Z` exists in MaStr/batcontrol.
2. The release is **published** (not draft) and has the wheel asset with exactly that name.
   Check via GitHub MCP release tools or
   `curl -sI -L <wheel-url> | head -1` (must be HTTP 200).

If the wheel is missing, stop and report — the add-on build would fail.

### Step 2 — Sync configuration from development

`batcontrol-development/` has been tracking upstream `main`, so it already contains every
change that is now in the release. Diff and sync:

```bash
diff batcontrol/config.yaml batcontrol-development/config.yaml
diff batcontrol/DOCS.md batcontrol-development/DOCS.md
diff batcontrol/translations/en.yaml batcontrol-development/translations/en.yaml
```

- Copy over all `options:` and `schema:` differences that belong to the release.
  Watch out: if development carries changes from upstream commits made AFTER the release
  tag, exclude those (compare against upstream `config/batcontrol_config_dummy.yaml` at the
  tag if unsure).
- Keep the stable-only header intact: `name: "Batcontrol-next"`, `slug: "batcontrol"`,
  stable description. Only sync options/schema/comments, not the manifest identity.
- Sync DOCS.md and translations/en.yaml the same way.
- Do NOT sync the Dockerfile blindly — stable builds from the release wheel, development
  builds from `main` with uv. Only port Dockerfile changes that are explicitly
  release-relevant (base image, runtime deps).

### Step 3 — Set the stable version

In `batcontrol/config.yaml`: `version: "X.Y.Z"` — must equal the upstream tag exactly
(this value is the Dockerfile's `BUILD_VERSION`). This bump is what makes HA offer the
update to users.

### Step 4 — Stable CHANGELOG.md

Prepend a new section matching the existing style (see the `0.8.0` entry — emoji headers):

```markdown
# 🚀 Release X.Y.Z - Released on DD.MM.YYYY

## What's Changed
### 🌟 Major New Features
...
```

Source material: the `# Release X.Y.Z - in Development` section from
`batcontrol-development/CHANGELOG.md` plus the upstream GitHub release notes. Keep upstream
PR references (`#NNN`).

### Step 5 — Validate

1. `python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" batcontrol/config.yaml`
2. Every `options:` key has a `schema:` counterpart at the same path (and vice versa).
3. `version` matches the upstream tag character for character.
4. CI (`lint.yaml`) will run the HA add-on linter; optionally trigger `manual-build.yaml`
   for add-on `batcontrol` to test the image build against the real release wheel.

## Part B — Open the next development cycle in `batcontrol-development/`

Do this after Part A (or standalone right after upstream bumped `main` to the next dev
version):

1. `batcontrol-development/config.yaml`: set `version` to the next cycle,
   `X.Y.(Z+1)dev1` (e.g. after releasing `0.8.1` -> `0.8.2dev1`).
2. `batcontrol-development/CHANGELOG.md`: the released section was consumed by Part A.
   Start a fresh top section:

   ```markdown
   # Release X.Y.(Z+1) - in Development

   ## What's Changed
   ```

   Keep the released section below it for history only if that matches the existing file's
   convention; otherwise remove the consumed content.

## Finish  [CONFIRM before push]

Commit both parts on a feature branch with a message like
`chore: release add-on X.Y.Z, open X.Y.(Z+1)dev cycle`, then ask before pushing / opening
the PR. Do not create a PR unless asked.

## Checklist

```
[ ] Upstream release X.Y.Z published, wheel asset name verified
[ ] batcontrol/config.yaml: options/schema/DOCS/translations synced from development
[ ] batcontrol/config.yaml: version = X.Y.Z
[ ] batcontrol/CHANGELOG.md: release section added (existing emoji style)
[ ] YAML valid, options/schema consistent
[ ] batcontrol-development/config.yaml: version = X.Y.(Z+1)dev1
[ ] batcontrol-development/CHANGELOG.md: fresh "in Development" section
[ ] Committed on feature branch; push/PR confirmed by user
```
