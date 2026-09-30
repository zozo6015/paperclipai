# syntax=docker/dockerfile:1.20
#
# Paperclip AI — hardened, distroless production image.
#
# The Paperclip source is not vendored in this repository: the `source` stage
# fetches upstream at a pinned commit (PAPERCLIP_REF), so a build is fully
# reproducible from this file alone. Bump PAPERCLIP_REF to upgrade.
#
# Stages
#   source    upstream git checkout at PAPERCLIP_REF (no .git)
#   build     pnpm + Rust toolchain; builds UI, plugin SDK, server (+ the
#             vendored paperclip-runnerd binary) and the Kubernetes
#             sandbox-provider plugin
#   prune     drops devDependencies and build-only trees from /app
#   tools     minimal runtime tool overlay (git, sh, tar, ps, ssh, tini) with
#             only the shared libraries the distroless base does not already have
#   runtime   gcr.io/distroless/nodejs24-debian13:nonroot + overlay + app
#
# Why not "pure" distroless: the Paperclip server itself shells out to git,
# tar and sh (workspace clones in the heartbeat loop, sandbox payload
# packaging, plugin installs via npm). Those binaries are copied in as a
# minimal overlay; there is still no package manager, no apt/dpkg tooling,
# no compilers and no coding-agent CLIs in the final image. Agents are meant
# to run in isolated pods via the Kubernetes sandbox-provider plugin.

ARG PAPERCLIP_REPO=https://github.com/paperclipai/paperclip.git
ARG PAPERCLIP_REF=d30b03bd8c17604cdab1533eeeeb087aba30e8b1
ARG NODE_BUILD_IMAGE=node:24-trixie-slim@sha256:8ec5d7557396cfe32d21c3f9c13072355ceab22b584578ca4bb28af31120cffe
ARG DEBIAN_IMAGE=debian:trixie-slim@sha256:a99cfc517144bc59b1978475ec53b46ecabec7e43635402ee5b77cc54cd1b20a
ARG RUNTIME_IMAGE=gcr.io/distroless/nodejs24-debian13:nonroot@sha256:bb6b03d81066993293a10feda7250e8e1cc034035fe9b61cfceededa7c8bf04d

# -----------------------------------------------------------------------------
FROM scratch AS source
ARG PAPERCLIP_REPO
ARG PAPERCLIP_REF
ADD --keep-git-dir=false ${PAPERCLIP_REPO}#${PAPERCLIP_REF} /

# -----------------------------------------------------------------------------
FROM ${NODE_BUILD_IMAGE} AS build
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
       ca-certificates curl git python3 gcc g++ make libc6-dev pkg-config \
  && rm -rf /var/lib/apt/lists/*

# Rust toolchain for the vendored paperclip-runnerd binary. rustup-init is
# version-pinned and checksum-verified (same pins as upstream); the compiler
# version itself comes from upstream's rust-toolchain.toml.
ENV RUSTUP_HOME=/usr/local/rustup \
    CARGO_HOME=/usr/local/cargo \
    PATH=/usr/local/cargo/bin:$PATH
ARG RUSTUP_VERSION=1.29.0
ARG RUSTUP_SHA256_AMD64=4acc9acc76d5079515b46346a485974457b5a79893cfb01112423c89aeb5aa10
ARG RUSTUP_SHA256_ARM64=9732d6c5e2a098d3521fca8145d826ae0aaa067ef2385ead08e6feac88fa5792
RUN set -eux; \
    case "$(dpkg --print-architecture)" in \
      amd64) target=x86_64-unknown-linux-gnu;  sha="$RUSTUP_SHA256_AMD64" ;; \
      arm64) target=aarch64-unknown-linux-gnu; sha="$RUSTUP_SHA256_ARM64" ;; \
      *) echo "unsupported architecture" >&2; exit 1 ;; \
    esac; \
    curl -fsSLo /tmp/rustup-init "https://static.rust-lang.org/rustup/archive/${RUSTUP_VERSION}/${target}/rustup-init"; \
    echo "${sha}  /tmp/rustup-init" | sha256sum -c -; \
    chmod +x /tmp/rustup-init; \
    /tmp/rustup-init -y --no-modify-path --profile minimal --default-toolchain none; \
    rm /tmp/rustup-init

WORKDIR /app
COPY --from=source / /app/
RUN cd packages/paperclip-runner && rustup show

ENV CI=true \
    PNPM_HOME=/pnpm \
    PATH=/pnpm:$PATH \
    NODE_OPTIONS=--max-old-space-size=4096
RUN corepack enable && pnpm config set store-dir /pnpm/store

RUN --mount=type=cache,id=paperclip-pnpm,target=/pnpm/store \
    pnpm install --frozen-lockfile

ARG PAPERCLIP_BUILD_COMMIT=""
RUN --mount=type=cache,id=paperclip-cargo-registry,target=/usr/local/cargo/registry \
    set -eux; \
    pnpm --filter @paperclipai/ui build; \
    pnpm --filter @paperclipai/plugin-sdk build; \
    pnpm --filter @paperclipai/server build; \
    test -f server/dist/index.js; \
    test -f ui/dist/index.html

# Pre-build the Kubernetes sandbox-provider plugin so it can be installed from
# the image (`--local`) without npm/pnpm/tsc at runtime. Upstream excludes the
# sandbox providers from the pnpm workspace and installs them standalone.
RUN --mount=type=cache,id=paperclip-pnpm,target=/pnpm/store \
    set -eux; \
    dir=packages/plugins/sandbox-providers/kubernetes; \
    pnpm -C "$dir" install --ignore-workspace --no-lockfile; \
    pnpm -C "$dir" build; \
    test -f "$dir/dist/manifest.js"

# -----------------------------------------------------------------------------
FROM build AS prune
# The server runs through the tsx ESM loader (workspace packages export .ts
# sources), but tsx is a devDependency. Install the exact same version into
# an isolated prefix before pruning devDependencies from the workspace.
RUN set -eux; \
    tsx_version="$(node -p 'require("/app/server/node_modules/tsx/package.json").version')"; \
    npm install --global-style --omit=dev --no-audit --no-fund --ignore-scripts \
      --prefix /opt/tsx "tsx@${tsx_version}"; \
    test -f /opt/tsx/node_modules/tsx/dist/loader.mjs
RUN --mount=type=cache,id=paperclip-pnpm,target=/pnpm/store \
    set -eux; \
    rm -rf packages/paperclip-runner/runner/target /usr/local/cargo /usr/local/rustup; \
    pnpm install --prod --frozen-lockfile --offline --ignore-scripts; \
    rm -rf ui/src ui/public ui/node_modules/.vite docs doc evals tests screenshots report releases \
           design announcements .github; \
    find . -name '*.map' -path '*/dist/*' -delete; \
    find . -type d \( -name __tests__ -o -name test -o -name tests \) -path '*/packages/*' \
         -not -path '*/node_modules/*' -prune -exec rm -rf {} +

# -----------------------------------------------------------------------------
FROM ${RUNTIME_IMAGE} AS distroless-ref

FROM ${DEBIAN_IMAGE} AS tools
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
       ca-certificates dash git openssh-client procps tar tini \
  && rm -rf /var/lib/apt/lists/*
COPY --from=distroless-ref / /distroless/
COPY --from=build /usr/local/lib/node_modules/npm /rootfs/usr/local/lib/node_modules/npm
# Assemble /rootfs: requested binaries + git helpers + every shared library
# they need that the distroless base does not already ship. The distroless
# base is merged-/usr (/bin -> usr/bin), so everything lands under /usr.
# Owning packages get a dpkg status stanza in /var/lib/dpkg/status.d (the
# distroless convention) so image scanners can still see and CVE-match them.
RUN set -eux; \
    mkdir -p /rootfs/usr/bin /rootfs/var/lib/dpkg/status.d; \
    bins="/usr/bin/dash /usr/bin/git /usr/bin/ssh /usr/bin/tar /usr/bin/ps /usr/bin/tini"; \
    helpers="$(find /usr/lib/git-core -type f -perm -u+x)"; \
    for f in $bins $helpers; do \
      mkdir -p "/rootfs$(dirname "$f")"; cp -a "$f" "/rootfs$f"; \
    done; \
    cp -a /usr/lib/git-core/. /rootfs/usr/lib/git-core/; \
    mkdir -p /rootfs/usr/share/git-core && cp -a /usr/share/git-core/templates /rootfs/usr/share/git-core/; \
    ln -s dash /rootfs/usr/bin/sh; \
    ln -s /nodejs/bin/node /rootfs/usr/bin/node; \
    printf '#!/bin/sh\nexec /nodejs/bin/node /usr/local/lib/node_modules/npm/bin/npm-cli.js "$@"\n' > /rootfs/usr/bin/npm; \
    printf '#!/bin/sh\nexec /nodejs/bin/node /usr/local/lib/node_modules/npm/bin/npx-cli.js "$@"\n' > /rootfs/usr/bin/npx; \
    chmod 0755 /rootfs/usr/bin/npm /rootfs/usr/bin/npx; \
    libs="$(for f in $bins $helpers; do ldd "$f" 2>/dev/null | awk '/=> \//{print $3}'; done | sort -u)"; \
    for lib in $libs; do \
      real="$(readlink -f "$lib")"; \
      if [ -e "/distroless$lib" ] || [ -e "/distroless$real" ]; then continue; fi; \
      mkdir -p "/rootfs$(dirname "$lib")"; \
      cp -a "$real" "/rootfs$real"; \
      [ "$lib" = "$real" ] || ln -sf "$(basename "$real")" "/rootfs$lib"; \
    done; \
    pkgs="$(for f in $bins $helpers $libs; do dpkg -S "$(readlink -f "$f")" 2>/dev/null || dpkg -S "$f" 2>/dev/null || true; done \
            | cut -d: -f1 | tr ',' '\n' | tr -d ' ' | sort -u)"; \
    for p in $pkgs; do \
      name="${p%%:*}"; \
      [ -e "/distroless/var/lib/dpkg/status.d/$name" ] && continue; \
      dpkg-query -s "$p" > "/rootfs/var/lib/dpkg/status.d/$name"; \
    done; \
    # Smoke-test the overlay inside a chroot of distroless + overlay.
    mkdir -p /check && cp -a /distroless/. /check/ && cp -a /rootfs/. /check/; \
    chroot /check /usr/bin/sh -c 'git --version && tar --version | head -1 && ps --version && ssh -V && tini --version && node --version && npm --version'; \
    chroot /check /usr/lib/git-core/git-remote-https 2>&1 | grep -q 'remote-curl'

# -----------------------------------------------------------------------------
FROM ${RUNTIME_IMAGE} AS runtime
ARG PAPERCLIP_REF
ARG PAPERCLIP_BUILD_VERSION=""
LABEL org.opencontainers.image.title="paperclip" \
      org.opencontainers.image.description="Paperclip AI server (distroless, non-root)" \
      org.opencontainers.image.source="https://github.com/zozo6015/paperclipai" \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.revision="${PAPERCLIP_REF}" \
      org.opencontainers.image.version="${PAPERCLIP_BUILD_VERSION}"

COPY --from=tools /rootfs/ /
COPY --from=prune /opt/tsx /opt/tsx
# Application code is root-owned and read-only for the runtime user. Only
# PAPERCLIP_HOME (a mounted volume) and /tmp are writable.
COPY --from=prune /app /app

ENV NODE_ENV=production \
    HOME=/paperclip \
    PAPERCLIP_HOME=/paperclip \
    PAPERCLIP_INSTANCE_ID=default \
    PAPERCLIP_CONFIG=/paperclip/instances/default/config.json \
    PAPERCLIP_DEPLOYMENT_MODE=authenticated \
    PAPERCLIP_DEPLOYMENT_EXPOSURE=private \
    PAPERCLIP_BUILD_COMMIT=${PAPERCLIP_REF} \
    PAPERCLIP_BUILD_VERSION=${PAPERCLIP_BUILD_VERSION} \
    HOST=0.0.0.0 \
    PORT=3100 \
    SERVE_UI=true \
    NPM_CONFIG_UPDATE_NOTIFIER=false \
    NPM_CONFIG_CACHE=/tmp/.npm \
    PATH=/nodejs/bin:/usr/local/bin:/usr/bin

WORKDIR /app
USER 65532:65532
EXPOSE 3100

# tini is PID 1 so orphaned git/ssh/tar children of agent runs are reaped.
ENTRYPOINT ["/usr/bin/tini", "--", "/nodejs/bin/node"]
CMD ["--import", "/opt/tsx/node_modules/tsx/dist/loader.mjs", "server/dist/index.js"]
