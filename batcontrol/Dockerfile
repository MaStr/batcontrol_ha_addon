# Multi-arch Home Assistant base image (amd64 + arm64), see
# https://github.com/home-assistant/docker-base
ARG BUILD_FROM=ghcr.io/home-assistant/base-python:3.14-alpine3.23
# Refers to the config.yaml
ARG BUILD_VERSION
FROM $BUILD_FROM AS builder

ARG BUILD_VERSION

# Download wheel from from releases into /wheels
# https://github.com/MaStr/batcontrol/releases/download/0.5.3/batcontrol-0.5.3-py3-none-any.whl
# and source code tar.gz
# https://github.com/MaStr/batcontrol/archive/refs/tags/0.5.3.tar.gz
RUN mkdir /wheels && \
    wget -O /wheels/batcontrol-"${BUILD_VERSION}"-py3-none-any.whl \
    https://github.com/MaStr/batcontrol/releases/download/${BUILD_VERSION}/batcontrol-${BUILD_VERSION}-py3-none-any.whl && \
    wget -O /batcontrol-"${BUILD_VERSION}".tar.gz \
    https://github.com/MaStr/batcontrol/archive/refs/tags/${BUILD_VERSION}.tar.gz

# Extract source to /batcontrol
RUN mkdir /batcontrol && \
    tar -xzf /batcontrol-"${BUILD_VERSION}".tar.gz -C /batcontrol --strip-components=1

ARG BUILD_FROM

# Stage 2: Build the final image
FROM $BUILD_FROM
# Refers to the config.yaml
ARG BUILD_VERSION
LABEL VERSION="${BUILD_VERSION}"
LABEL org.opencontainers.image.title="Home Assistant App: Batcontrol-next" \
      org.opencontainers.image.source="https://github.com/MaStr/batcontrol_ha_addon" \
      org.opencontainers.image.description="Alternate batcontrol add-on for Home Assistant" \
      org.opencontainers.image.licenses=MIT

# Copy the built wheel from the builder stage and install it
COPY --from=builder /wheels /wheels

# Install runtime dependencies
RUN apk add --no-cache \
            python3 \
            py3-pip \
            py3-setuptools \
            tzdata

COPY --from=builder /wheels /wheels
RUN python3 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --no-cache-dir --extra-index-url https://piwheels.org/simple --prefer-binary /wheels/*.whl

ENV BATCONTROL_VERSION=${BUILD_VERSION}
ENV BATCONTROL_RUNTIME_ENV="home_assistant_addon-ng"

# Set default timezone to UTC, override with -e TZ=Europe/Berlin or similar
# when starting the container
# or set the timezone in docker-compose.yml in the environment section,
ENV TZ=UTC

# Create the app directory and copy the app files
RUN mkdir -p /app /app/logs /app/config
WORKDIR /app

# The load profiles to all locations where it is needed
COPY --from=builder /batcontrol/config/load_profile_default.csv ./config/load_profile.csv
COPY --from=builder /batcontrol/config/load_profile_default.csv ./default_load_profile.csv

COPY ../LICENSE  ./
COPY --from=builder /batcontrol/entrypoint_ha.sh ./
# Licenses of the fonts bundled with the dashboard (SIL OFL 1.1).
COPY --from=builder /batcontrol/THIRD-PARTY-LICENSES.md ./
COPY --from=builder /batcontrol/config ./config_template

# Set the scripts as executable
RUN chmod +x entrypoint_ha.sh

# The read-only API, dashboard and MCP server. Reached through ingress; the
# port is only published when the user maps it in the add-on configuration.
EXPOSE 8080

# COPY ./src/run_infinite.sh ./run_infinite.sh
# CMD [ "./run_infinite.sh" ]
CMD [ "./entrypoint_ha.sh" ]
