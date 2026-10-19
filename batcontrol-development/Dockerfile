ARG BUILD_FROM
# Refers to the config.yaml
ARG BUILD_VERSION

# Stage 1: Fetch the upstream source once
#
# Pinned to the build host so the zip is downloaded and unpacked a single
# time, not once per target architecture under emulation. Source files are
# architecture independent.
FROM --platform=$BUILDPLATFORM alpine:3.24 AS source

# Fetch the source code as zip from Github MaStr/batcontrol
# https://github.com/MaStr/batcontrol/archive/refs/heads/main.zip
# and extract it to /batcontrol
RUN apk add --no-cache unzip && \
    mkdir -p /batcontrol && \
    wget -O /tmp/batcontrol.zip \
    https://github.com/MaStr/batcontrol/archive/refs/heads/main.zip && \
    unzip -q /tmp/batcontrol.zip -d /tmp && \
    mv /tmp/batcontrol-main/* /batcontrol/ && \
    rm -rf /tmp/batcontrol.zip /tmp/batcontrol-main

# Stage 2: Build the dashboard
#
# A dedicated Node image instead of "apk add nodejs npm" into the Python
# stage: the Node version is pinned here rather than inherited from whatever
# the base image's package repository ships, and the toolchain stays out of
# the stage that produces the wheel.
#
# Also pinned to the build host - the output is static files, so npm runs
# once, natively, instead of once per architecture under QEMU.
FROM --platform=$BUILDPLATFORM node:26-alpine AS frontend

# Same layout as the repository, so vite's outDir resolves to
# /build/src/batcontrol/web/static.
WORKDIR /build/frontend
COPY --from=source /batcontrol/frontend ./
# Also runs the offline asset check: the build fails if the dashboard would
# load a font, stylesheet or script from the network. A Home Assistant box
# may have no internet egress at all, and ingress would not reach one anyway.
RUN npm ci && npm run build

ARG BUILD_FROM

# Stage 3: Build the wheel
FROM $BUILD_FROM AS builder

ARG BUILD_VERSION

# Install uv for fast, reliable Python packaging
COPY --from=ghcr.io/astral-sh/uv:0.11 /uv /usr/local/bin/uv

COPY --from=source /batcontrol /batcontrol
# The dashboard travels inside the wheel as package data.
COPY --from=frontend /build/src/batcontrol/web/static /batcontrol/src/batcontrol/web/static

# Install build dependencies and build the wheel package
RUN apk add --no-cache python3-dev
WORKDIR /batcontrol
RUN uv build --wheel --out-dir /wheels

# A wheel without the dashboard installs fine and then serves a hint page
# where the panel should be. Fail the build instead.
RUN python3 scripts/verify_wheel_dashboard.py /wheels/*.whl

ARG BUILD_FROM

# Stage 4: Build the final image
FROM $BUILD_FROM
# Refers to the config.yaml
ARG BUILD_VERSION
LABEL VERSION="${BUILD_VERSION}"
LABEL org.opencontainers.image.source = "https://github.com/MaStr/batcontrol_ha_addon"
LABEL org.opencontainers.image.description="(SNAPSHOT) Alternate batcontrol add-on for Home Assistant"
LABEL org.opencontainers.image.licenses=MIT

# Copy the built wheel from the builder stage and install it
COPY --from=builder /wheels /wheels

# Install runtime dependencies
RUN apk add --no-cache \
            tzdata

# Prefer piwheels for compatible wheels and fall back to PyPI
RUN pip install --no-cache-dir --extra-index-url https://piwheels.org/simple --prefer-binary /wheels/*.whl

ENV BATCONTROL_VERSION=${BUILD_VERSION}
ENV BATCONTROL_GIT_SHA=${GIT_SHA}
ENV BATCONTROL_RUNTIME_ENV="home_assistant_addon-ng"

# Set default timezone to UTC, override with -e TZ=Europe/Berlin or similar
# when starting the container
# or set the timezone in docker-compose.yml in the environment section,
ENV TZ=UTC

# Create the app directory and copy the app files
RUN mkdir -p /app /app/logs /app/config
WORKDIR /app

# The load profiles to all locations where it is needed
COPY --from=source /batcontrol/config/load_profile_default.csv ./config/load_profile.csv
COPY --from=source /batcontrol/config/load_profile_default.csv ./default_load_profile.csv

COPY ../LICENSE  ./
COPY --from=source /batcontrol/entrypoint_ha.sh ./
COPY --from=source /batcontrol/config ./config_template
# Licenses of the fonts bundled with the dashboard (SIL OFL 1.1).
COPY --from=source /batcontrol/THIRD-PARTY-LICENSES.md ./

# Set the scripts as executable
RUN chmod +x entrypoint_ha.sh

# The read-only API, dashboard and MCP server. Reached through ingress; the
# port is only published when the user maps it in the add-on configuration.
EXPOSE 8080

# COPY ./src/run_infinite.sh ./run_infinite.sh
# CMD [ "./run_infinite.sh" ]
CMD [ "./entrypoint_ha.sh" ]
