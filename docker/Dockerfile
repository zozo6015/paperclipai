# syntax=docker/dockerfile:1.20
#
# Paperclip AI image: the official upstream image plus the CodeRabbit CLI.
#
# The upstream image (ghcr.io/paperclipai/paperclip, multi-arch) already
# carries what agents run with: a Debian userland, git, gh, ripgrep, python3,
# ssh and the agent CLIs (claude, codex, opencode, gemini, kimi). This file
# only adds what upstream does not ship. Paperclip itself is not rebuilt.
#
# Targets
#   runtime   the image that is pushed
#   version   side target the CI reads to tag the image with the Paperclip
#             release it contains (`--target version`)

# Upgrade: bump both together (see README "Upgrading Paperclip").
ARG PAPERCLIP_IMAGE=ghcr.io/paperclipai/paperclip:2026.916.1@sha256:a02ac35ac41df911af477422ea0e781cf41d2b2c600c66f0a5ac9d8c63f52c2c
ARG PAPERCLIP_VERSION=2026.916.1

# -----------------------------------------------------------------------------
FROM ${PAPERCLIP_IMAGE} AS runtime
LABEL org.opencontainers.image.title="paperclip" \
      org.opencontainers.image.description="Paperclip AI (official image) + CodeRabbit CLI" \
      org.opencontainers.image.source="https://github.com/zozo6015/paperclipai" \
      org.opencontainers.image.licenses="MIT"

# The upstream image leaves the user as root (its entrypoint drops to `node`
# when started as root, and runs as-is when started unprivileged).
USER root

# CodeRabbit CLI, installed with its official installer
# (`curl -fsSL https://cli.coderabbit.ai/install.sh | sh`). The installer
# targets the invoking user's home, so it runs against a throwaway HOME and
# the resulting binary is moved to /usr/local/bin for every user. The
# installer always fetches the current release; the BuildKit layer cache
# keeps it fixed until CODERABBIT_CACHE_EPOCH is changed.
ARG CODERABBIT_CACHE_EPOCH=""
RUN set -eux; \
    echo "coderabbit-cache-epoch: ${CODERABBIT_CACHE_EPOCH}"; \
    apt-get update; \
    apt-get install -y --no-install-recommends unzip; \
    rm -rf /var/lib/apt/lists/*; \
    tmp="$(mktemp -d)"; \
    curl -fsSL https://cli.coderabbit.ai/install.sh -o "$tmp/install.sh"; \
    HOME="$tmp/home" SHELL=/bin/sh sh "$tmp/install.sh" </dev/null; \
    found="$(find "$tmp/home" -name coderabbit \( -type f -o -type l \) | head -n1)"; \
    test -n "$found"; \
    install -m 0755 "$(readlink -f "$found")" /usr/local/bin/coderabbit; \
    ln -sf coderabbit /usr/local/bin/cr; \
    rm -rf "$tmp"; \
    coderabbit --version

# Its interactive login (`coderabbit auth login`) stores credentials under
# $HOME, which is /paperclip, the persistent volume.

# -----------------------------------------------------------------------------
FROM ${PAPERCLIP_IMAGE} AS version-src
ARG PAPERCLIP_VERSION
RUN printf '{\n  "version": "%s"\n}\n' "$PAPERCLIP_VERSION" > /tmp/server-package.json

FROM scratch AS version
COPY --from=version-src /tmp/server-package.json /server-package.json

# -----------------------------------------------------------------------------
FROM runtime
