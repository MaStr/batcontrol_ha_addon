ARG BUILD_FROM
# Refers to the config.yaml
ARG BUILD_VERSION
FROM $BUILD_FROM AS builder

ARG BUILD_VERSION

# Download wheel from from releases into /wheels
# https://github.com/muexxl/batcontrol/releases/download/0.5.3/batcontrol-0.5.3-py3-none-any.whl
# and source code tar.gz
# https://github.com/muexxl/batcontrol/archive/refs/tags/0.5.3.tar.gz
RUN mkdir /wheels && \
    wget -O /wheels/batcontrol-"${BUILD_VERSION}"-py3-none-any.whl \
    https://github.com/muexxl/batcontrol/releases/download/${BUILD_VERSION}/batcontrol-${BUILD_VERSION}-py3-none-any.whl && \
    wget -O /batcontrol-"${BUILD_VERSION}".tar.gz \
    https://github.com/muexxl/batcontrol/archive/refs/tags/${BUILD_VERSION}.tar.gz

# Extract source to /batcontrol
RUN mkdir /batcontrol && \
    tar -xzf /batcontrol-"${BUILD_VERSION}".tar.gz -C /batcontrol --strip-components=1

ARG BUILD_FROM

# Stage 2: Build the final image
FROM $BUILD_FROM
# Refers to the config.yaml
ARG BUILD_VERSION
LABEL VERSION="${BUILD_VERSION}"

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
ENV BATCONTROL_GIT_SHA=${GIT_SHA}
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
COPY --from=builder /batcontrol/config ./config_template

# Set the scripts as executable
RUN chmod +x entrypoint_ha.sh

# COPY ./src/run_infinite.sh ./run_infinite.sh
# CMD [ "./run_infinite.sh" ]
CMD [ "./entrypoint_ha.sh" ]
