# syntax=docker/dockerfile:1.20
#
# Paperclip AI — hardened, distroless, multi-arch production image.
#
# Paperclip is installed from its published npm release (the same package the
# upstream quickstart installs), pinned together with its whole dependency
# tree by docker/package.json + docker/package-lock.json. Nothing is compiled.
#
# Stages
#   install   runs on the BUILD platform (never emulated): `npm ci` fetches the
#             prebuilt native packages for the TARGET platform via --os/--cpu,
#             with install scripts disabled, so nothing target-specific runs
#   tools     minimal runtime tool overlay (git, sh, tar, ps, ssh, tini, npm)
#             with only the shared libraries the distroless base lacks
#   runtime   gcr.io/distroless/nodejs24-debian13:nonroot + overlay + app
#
# Why not "pure" distroless: the Paperclip server itself shells out to git,
# tar and sh (workspace clones in the heartbeat loop, sandbox payload
# packaging) and npm (plugin installs). Those binaries are copied in as a
# minimal overlay; there is still no package manager, no apt/dpkg tooling and
# no compiler in the final image.

ARG NODE_BUILD_IMAGE=node:24-trixie-slim@sha256:8ec5d7557396cfe32d21c3f9c13072355ceab22b584578ca4bb28af31120cffe
ARG DEBIAN_IMAGE=debian:trixie-slim@sha256:a99cfc517144bc59b1978475ec53b46ecabec7e43635402ee5b77cc54cd1b20a
ARG RUNTIME_IMAGE=gcr.io/distroless/nodejs24-debian13:nonroot@sha256:bb6b03d81066993293a10feda7250e8e1cc034035fe9b61cfceededa7c8bf04d

# -----------------------------------------------------------------------------
FROM --platform=$BUILDPLATFORM ${NODE_BUILD_IMAGE} AS install
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
ARG TARGETARCH
WORKDIR /opt/paperclip
COPY docker/package.json docker/package-lock.json ./
RUN --mount=type=cache,id=paperclip-npm,target=/root/.npm \
    set -eux; \
    case "$TARGETARCH" in \
      amd64) cpu=x64 ;; \
      arm64) cpu=arm64 ;; \
      *) echo "unsupported TARGETARCH: $TARGETARCH" >&2; exit 1 ;; \
    esac; \
    # `npm install` with the committed lockfile installs exactly the locked
    # versions. (`npm ci` refuses this tree: its sync check wants esbuild's
    # platform variants inside bundleDependencies, which are not published.)
    npm install --no-save --omit=dev --ignore-scripts --no-audit --no-fund \
      --os=linux --cpu="$cpu" --libc=glibc; \
    # Packages published with bundleDependencies (inside @paperclipai/server
    # and @paperclipai/adapter-utils) carry the esbuild binary of the machine
    # they were packed on. Give every esbuild copy its target-platform binary
    # (same version) and drop the others.
    find node_modules -path '*node_modules/esbuild/package.json' -print0 | \
    while IFS= read -r -d '' pj; do \
      nm="$(dirname "$(dirname "$pj")")"; \
      ver="$(node -p "require('./$pj').version")"; \
      want="$nm/@esbuild/linux-$cpu"; \
      if [ ! -d "$want" ]; then \
        npm pack --silent --pack-destination /tmp "@esbuild/linux-$cpu@$ver"; \
        mkdir -p "$want"; \
        tar -xzf "/tmp/esbuild-linux-$cpu-$ver.tgz" -C "$want" --strip-components=1; \
      fi; \
      for other in "$nm"/@esbuild/*; do [ "$other" = "$want" ] || rm -rf "$other"; done; \
    done; \
    test -f node_modules/@paperclipai/server/dist/index.js; \
    test -f node_modules/@paperclipai/server/ui-dist/index.html

# -----------------------------------------------------------------------------
FROM ${RUNTIME_IMAGE} AS distroless-ref

FROM ${DEBIAN_IMAGE} AS tools
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
       ca-certificates dash git openssh-client procps tar tini \
  && rm -rf /var/lib/apt/lists/*
COPY --from=distroless-ref / /distroless/
# npm is pure JavaScript, so the build-platform copy works on any target.
COPY --from=install /usr/local/lib/node_modules/npm /rootfs/usr/local/lib/node_modules/npm
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
      # ldd reports /lib/...; on merged-/usr systems that is /usr/lib/...
      case "$lib" in /lib/*|/lib64/*) lib="/usr$lib" ;; esac; \
      if [ -e "/distroless$lib" ] || [ -e "/distroless$real" ]; then continue; fi; \
      mkdir -p "/rootfs$(dirname "$real")" "/rootfs$(dirname "$lib")"; \
      cp -a "$real" "/rootfs$real"; \
      [ "$lib" = "$real" ] || ln -sf "$real" "/rootfs$lib"; \
    done; \
    # Anything outside /usr and /var would replace the distroless base's
    # merged-/usr symlinks (/lib -> usr/lib, /bin -> usr/bin) on COPY.
    for d in /rootfs/*; do \
      case "$d" in /rootfs/usr|/rootfs/var) ;; *) echo "unexpected top-level path in overlay: $d" >&2; exit 1 ;; esac; \
    done; \
    pkgs="$(for f in $bins $helpers $libs; do dpkg -S "$(readlink -f "$f")" 2>/dev/null || dpkg -S "$f" 2>/dev/null || true; done \
            | cut -d: -f1 | tr ',' '\n' | tr -d ' ' | sort -u)"; \
    for p in $pkgs; do \
      name="${p%%:*}"; \
      [ -e "/distroless/var/lib/dpkg/status.d/$name" ] && continue; \
      dpkg-query -s "$p" > "/rootfs/var/lib/dpkg/status.d/$name"; \
    done

# -----------------------------------------------------------------------------
FROM ${RUNTIME_IMAGE} AS runtime
LABEL org.opencontainers.image.title="paperclip" \
      org.opencontainers.image.description="Paperclip AI server (distroless, non-root)" \
      org.opencontainers.image.source="https://github.com/zozo6015/paperclipai" \
      org.opencontainers.image.licenses="MIT"

COPY --from=tools /rootfs/ /
# Application code is root-owned and read-only for the runtime user. Only
# PAPERCLIP_HOME (a mounted volume) and /tmp are writable.
COPY --from=install /opt/paperclip /opt/paperclip

ENV NODE_ENV=production \
    HOME=/paperclip \
    PAPERCLIP_HOME=/paperclip \
    PAPERCLIP_INSTANCE_ID=default \
    PAPERCLIP_CONFIG=/paperclip/instances/default/config.json \
    PAPERCLIP_DEPLOYMENT_MODE=authenticated \
    PAPERCLIP_DEPLOYMENT_EXPOSURE=private \
    HOST=0.0.0.0 \
    PORT=3100 \
    SERVE_UI=true \
    NPM_CONFIG_UPDATE_NOTIFIER=false \
    NPM_CONFIG_CACHE=/tmp/.npm \
    PATH=/nodejs/bin:/opt/paperclip/node_modules/.bin:/usr/local/bin:/usr/bin

# Smoke-test the tool overlay in the real final filesystem (under QEMU for a
# foreign target platform): every binary must start, i.e. all its shared
# libraries resolve. git-remote-https prints its usage and exits non-zero when
# it starts correctly. Kept free of JavaScript so no V8 runs under emulation.
# Adds only an empty layer.
RUN ["/usr/bin/sh", "-c", "set -e; git --version; tar --version >/dev/null; ps --version; ssh -V; tini --version; node --version; out=\"$(/usr/lib/git-core/git-remote-https 2>&1 || true)\"; case \"$out\" in *remote-curl*) ;; *) echo \"$out\" >&2; exit 1 ;; esac"]

WORKDIR /opt/paperclip/node_modules/@paperclipai/server
USER 65532:65532
EXPOSE 3100

# tini is PID 1 so orphaned git/ssh/tar children of agent runs are reaped.
ENTRYPOINT ["/usr/bin/tini", "--", "/nodejs/bin/node"]
CMD ["dist/index.js"]
