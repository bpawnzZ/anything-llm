# Setup base image
FROM ubuntu:jammy-20240627.1 AS base

# Build arguments
ARG ARG_UID=1000
ARG ARG_GID=1000

# Shared installation stage
FROM base AS build-base
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Install system dependencies
RUN DEBIAN_FRONTEND=noninteractive apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends \
        unzip curl gnupg libgfortran5 libgbm1 tzdata netcat \
        libasound2 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 libexpat1 libfontconfig1 \
        libgcc1 libglib2.0-0 libgtk-3-0 libnspr4 libpango-1.0-0 libx11-6 libx11-xcb1 libxcb1 \
        libxcomposite1 libxcursor1 libxdamage1 libxext6 libxfixes3 libxi6 libxrandr2 libxrender1 \
        libxss1 libxtst6 ca-certificates fonts-liberation libappindicator1 libnss3 lsb-release \
        xdg-utils git build-essential ffmpeg && \
    # Install Node.js
    mkdir -p /etc/apt/keyrings && \
    curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg && \
    echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_18.x nodistro main" | tee /etc/apt/sources.list.d/nodesource.list && \
    apt-get update && \
    apt-get install -yq --no-install-recommends nodejs && \
    # Install Yarn
    npm install -g yarn && \
    # Clean up
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Create user and directories
RUN groupadd -g "$ARG_GID" anythingllm && \
    useradd -l -u "$ARG_UID" -m -d /app -s /bin/bash -g anythingllm anythingllm && \
    mkdir -p /app/frontend/ /app/server/ /app/collector/ && \
    chown -R anythingllm:anythingllm /app

# Architecture-specific stages
FROM build-base AS build-arm64
RUN echo "Preparing build of AnythingLLM image for arm64 architecture"
# Puppeteer ARM64 specific setup
RUN curl https://playwright.azureedge.net/builds/chromium/1088/chromium-linux-arm64.zip -o chrome-linux.zip && \
    unzip chrome-linux.zip && \
    rm -rf chrome-linux.zip

ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true
ENV CHROME_PATH=/app/chrome-linux/chrome
ENV PUPPETEER_EXECUTABLE_PATH=/app/chrome-linux/chrome

FROM build-base AS build-amd64
RUN echo "Preparing build of AnythingLLM image for non-ARM architecture"

# Common build stage
FROM build-${TARGETARCH} AS build
USER anythingllm
WORKDIR /app

# Copy helper scripts
COPY --chown=anythingllm:anythingllm ./docker/docker-entrypoint.sh /usr/local/bin/
COPY --chown=anythingllm:anythingllm ./docker/docker-healthcheck.sh /usr/local/bin/
COPY --chown=anythingllm:anythingllm ./docker/.env.example /app/server/.env

RUN chmod +x /usr/local/bin/docker-entrypoint.sh && \
    chmod +x /usr/local/bin/docker-healthcheck.sh

# Frontend build stage
FROM build AS frontend-build
COPY --chown=anythingllm:anythingllm ./frontend /app/frontend/
WORKDIR /app/frontend

RUN yarn install --network-timeout 100000 && \
    yarn build && \
    cp -r dist /tmp/frontend-build && \
    rm -rf * && \
    cp -r /tmp/frontend-build dist && \
    rm -rf /tmp/frontend-build

# Backend build stage
FROM build AS backend-build
COPY ./server /app/server/
WORKDIR /app/server

RUN yarn install --production --network-timeout 100000 && \
    yarn cache clean || true

# Collector build stage
COPY ./collector/ ./collector/
WORKDIR /app/collector

ENV PUPPETEER_DOWNLOAD_BASE_URL=https://storage.googleapis.com/chrome-for-testing-public

RUN yarn install --production --network-timeout 100000 && \
    yarn cache clean || true

# Compile Llama.cpp bindings
USER root
WORKDIR /app/server
RUN npx node-llama-cpp download
WORKDIR /app
USER anythingllm

# Production build stage
FROM backend-build AS production-build
WORKDIR /app
COPY --chown=anythingllm:anythingllm --from=frontend-build /app/frontend/dist /app/server/public

USER root
RUN chown -R anythingllm:anythingllm /app/server && \
    chown -R anythingllm:anythingllm /app/collector
USER anythingllm

# Environment and final setup
ENV NODE_ENV=production
ENV ANYTHING_LLM_RUNTIME=docker

HEALTHCHECK --interval=1m --timeout=10s --start-period=1m \
  CMD /bin/bash /usr/local/bin/docker-healthcheck.sh || exit 1

ENTRYPOINT ["/bin/bash", "/usr/local/bin/docker-entrypoint.sh"]
