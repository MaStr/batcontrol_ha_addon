# CLAUDE.md — batcontrol_ha_addon

Home Assistant add-on repository packaging [MaStr/batcontrol](https://github.com/MaStr/batcontrol).
There is **no application code here** — only HA add-on packaging, configuration manifests, and
user documentation. The Python source lives in the upstream `batcontrol` repo.

## Layout

```
repository.yaml               # HA add-on repository manifest (name, url, maintainer)
batcontrol/                   # STABLE add-on ("Batcontrol-next")
  config.yaml                 # Add-on manifest: name/version/slug + options: + schema:
  Dockerfile                  # Image build; BUILD_VERSION is taken from config.yaml version
  DOCS.md                     # User-facing parameter docs shown in the HA UI
  CHANGELOG.md                # Shown in the HA UI on update
  translations/en.yaml        # Names/descriptions of option groups in the HA config UI
  build.yaml                  # Base images per architecture
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
| Dashboard assets | Already inside the release wheel - **needs no Node** | Built in a `node:24-alpine` stage from the fetched source |
| `version` in config.yaml | Must equal an existing upstream release tag (e.g. `0.8.0`) | `X.Y.ZdevN` (e.g. `0.8.1dev3`) |
| When to touch | Only when upstream publishes a release | First landing spot for every upstream change |

Bumping `version` in `config.yaml` is what makes HA offer users an update — a Dockerfile or
options change without a version bump is invisible to users.

## The dashboard and the read-only API

Upstream batcontrol serves a read-only dashboard, HTTP API and MCP server from the same process
as the control loop. In this add-on that is the **Batcontrol** sidebar entry, served through
ingress.

- The manifest carries `ingress: true`, `ingress_port: 8080`, `panel_icon`, `panel_title` and an
  **unmapped** `ports: {8080/tcp: null}` for optional direct access.
- `api.host` must stay **`0.0.0.0`** here. Ingress connects from the supervisor, i.e. from
  outside the container, so upstream's `127.0.0.1` default makes the panel unreachable. This is
  the one place where the add-on deliberately differs from the upstream default.
- `api.enabled` defaults to **true** in this add-on (upstream defaults to false), because the
  sidebar panel needs it.
- A mapped port bypasses Home Assistant's authentication; ingress does not. DOCS.md tells users
  to set `auth_token` before mapping it.
- The dashboard assets live inside the wheel as package data. **Stable therefore needs no Node
  at all** - it installs the released wheel, which already carries them. Only the development
  add-on builds them, because it builds the wheel itself from the `main` zip.
- The development Dockerfile uses a dedicated `node:24-alpine` stage rather than
  `apk add nodejs npm`: the Node version is pinned instead of inherited from the base image's
  package repository, and `--platform=$BUILDPLATFORM` keeps the npm build off QEMU in the
  aarch64 build. Its builder stage runs `scripts/verify_wheel_dashboard.py`, which fails the
  build if the wheel has no dashboard - otherwise that failure only shows up as a hint page in
  the user's sidebar.

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
