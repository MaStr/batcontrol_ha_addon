# CLAUDE.md — batcontrol_ha_addon

Home Assistant add-on repository packaging [MaStr/batcontrol](https://github.com/MaStr/batcontrol).
There is **no application code here** — only HA add-on packaging, configuration manifests, and
user documentation. The Python source lives in the upstream `batcontrol` repo.

## Layout

```
repository.yaml               # HA add-on repository manifest (name, url, maintainer)
batcontrol/                   # STABLE add-on ("Batcontrol-next")
batcontrol-development/       # DEVELOPMENT add-on ("Batcontrol-development")
.github/workflows/lint.yaml   # frenck/action-addon-linter over every add-on dir

# Both add-on directories contain the same set of files:
<addon>/config.yaml           # Add-on manifest: name/version/slug + options: + schema:
<addon>/Dockerfile            # Image build; BUILD_VERSION is taken from config.yaml version
<addon>/DOCS.md               # User-facing parameter docs shown in the HA UI
<addon>/CHANGELOG.md          # Shown in the HA UI on update
<addon>/translations/en.yaml  # Names/descriptions of option groups in the HA config UI
<addon>/build.yaml            # Base images per architecture
```

## The two add-ons

| | `batcontrol/` (stable) | `batcontrol-development/` |
|---|---|---|
| Builds from | Upstream **release wheel** for tag `<version>` | Upstream **`main` branch** zip, wheel built with uv |
| `version` in config.yaml | Must equal an existing upstream release tag (e.g. `0.8.0`) | `X.Y.ZdevN` (e.g. `0.8.1dev3`) |
| When to touch | Only when upstream publishes a release | First landing spot for every upstream change |

Bumping `version` in `config.yaml` is what makes HA offer users an update — a Dockerfile or
options change without a version bump is invisible to users.

## Rules

- Every key under `options:` needs a matching entry under `schema:` in the same `config.yaml`.
  Schema syntax: `str`, `bool`, `int`, `float`, `password`, `list(a|b|c)`, ranges like
  `int(0,23)` / `float(0,1)`, trailing `?` = optional. Reference:
  https://developers.home-assistant.io/docs/add-ons/configuration/#options--schema
- `options:` values mirror the upstream reference config
  (`config/batcontrol_config_dummy.yaml` in MaStr/batcontrol): keep key names, nesting, order,
  and comments aligned. Not every upstream key is exposed (e.g. `logfile_path`,
  `logfile_enabled` are fixed by the add-on) — compare with the existing file before adding.
- Changes go to `batcontrol-development/` first. `batcontrol/` is only synced when upstream
  tags a release; then both `options:`/`schema:` and CHANGELOG are brought up to that release.
- CHANGELOG.md (development) keeps a `# Release X.Y.Z - in Development` section on top,
  grouped by category, referencing upstream PR numbers (`#NNN`).
- `translations/en.yaml` only describes top-level option groups — extend it when a new
  top-level block (like `dynamic_network_fees`) is added.
- Validate YAML after editing: `python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" <file>`.
  CI runs the official HA add-on linter on every add-on directory.

## Skills

- `port-batcontrol-change` (`.claude/skills/port-batcontrol-change/SKILL.md`): port a single
  upstream change — diff the upstream reference config, update `options:`/`schema:`, DOCS.md,
  CHANGELOG.md, translations, bump the dev version.
- `release-addon` (`.claude/skills/release-addon/SKILL.md`): promote a published upstream
  release into the stable add-on (sync from development, set `version` to the release tag,
  write the release CHANGELOG section) and open the next `devN` cycle in
  `batcontrol-development/`.
