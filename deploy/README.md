# Deploying Paperclip on Kubernetes

```
docker/Dockerfile                      distroless multi-stage image build
deploy/kubernetes/base                 cluster-agnostic manifests (Kustomize)
deploy/kubernetes/components/
  cloudnativepg                        PostgreSQL via the CloudNativePG operator
  gateway-api                          HTTPRoute on an existing Gateway
  kubernetes-sandbox                   OPTIONAL RBAC for agent sandboxes (see below)
deploy/kubernetes/overlays/zolab       the zolab cluster
deploy/tekton/base                     Tekton pipeline + standalone GitHub Trigger
deploy/tekton/components/eventlistener OPTIONAL dedicated EventListener
deploy/tekton/overlays/zolab           zolab (shares an existing EventListener)
```

Images are published to `ghcr.io/zozo6015/paperclipai` by Tekton only; no
GitHub Actions are involved.

## The image

`docker/Dockerfile` installs the published npm release of
[Paperclip](https://github.com/paperclipai/paperclip) (`@paperclipai/server`,
pinned with its whole dependency tree by `docker/package.json` and
`docker/package-lock.json`) and ships it on
`gcr.io/distroless/nodejs24-debian13:nonroot` (UID 65532). Nothing is
compiled: the install runs on the build machine's own architecture and fetches
the target platform's prebuilt native packages. All base images are pinned by
digest.

It is *distroless plus a minimal tool overlay*, not pure distroless: the
Paperclip server itself executes `git`, `tar` and `sh` (workspace clones in
the heartbeat loop, sandbox payload packaging) and `npm` (plugin installs).
Those binaries and only the shared libraries missing from the distroless base
are copied in, with dpkg metadata so scanners still see them. No package
manager or compiler is included.

The Claude Code CLI (`claude`, pinned in `docker/package.json`) is included
for the `claude_local` adapter, together with `bash`, which its Bash tool
requires. The adapter starts `claude` as a child process in the server
container, with the agent workspace as its working directory, so it has to be
in the same image; a sidecar container cannot serve it. Other local CLIs
(codex, gemini, …) are not included: run those agents in sandboxes (the
`kubernetes-sandbox` component + upstream plugin) or through remote adapters.

Build locally:

```sh
docker buildx build -f docker/Dockerfile -t paperclip:dev .
# another upstream commit:
```

## Kubernetes

Security defaults in the base: namespace enforces Pod Security `restricted`;
non-root UID/GID 65532, read-only root filesystem, all capabilities dropped,
`RuntimeDefault` seccomp, no privilege escalation, no service-account token,
default-deny NetworkPolicies with explicit allows, one replica with a
`Recreate` strategy (the server holds in-process scheduler state on a
ReadWriteOnce volume).

### Secrets (never committed)

```sh
kubectl create namespace paperclip --dry-run=client -o yaml | kubectl apply -f -
kubectl -n paperclip create secret generic paperclip-secrets \
  --from-literal=BETTER_AUTH_SECRET="$(openssl rand -base64 48)" \
  --from-literal=PAPERCLIP_SECRETS_MASTER_KEY="$(openssl rand -base64 32)" \
  --from-literal=PAPERCLIP_AGENT_JWT_SECRET="$(openssl rand -base64 48)"
```

For `claude_local` agents, add one Claude Code credential to the same Secret:
`ANTHROPIC_API_KEY` (API billing) or `CLAUDE_CODE_OAUTH_TOKEN` (a Claude
subscription; create it with `claude setup-token` on a machine with a
browser). Both keys are optional, and the pod needs a restart to pick up a
change:

```sh
kubectl -n paperclip patch secret paperclip-secrets --type merge \
  -p '{"stringData":{"CLAUDE_CODE_OAUTH_TOKEN":"<token>"}}'
kubectl -n paperclip rollout restart deployment/paperclip
```

Back up `PAPERCLIP_SECRETS_MASTER_KEY`: it encrypts the company secrets stored
in the database and they are unrecoverable without it.

`DATABASE_URL` is read from Secret `paperclip-db-app`, key `uri`. The
`cloudnativepg` component makes CNPG generate it. With your own PostgreSQL:

```sh
kubectl -n paperclip create secret generic paperclip-db-app \
  --from-literal=uri='postgres://paperclip:<password>@<host>:5432/paperclip?sslmode=require'
```

### zolab

Cluster facts used by the overlay: Cilium Gateway API with Gateway `eg` in
`kube-system`, CloudNativePG on `synology-iscsi` (block storage), and
Paperclip's `paperclip-data` volume on `nfs-isolated` (NFS). PostgreSQL stays
on iSCSI because NFS is not recommended for database files.

The CNPG operator runs in `cnpg-system`. Paperclip is private (not reachable
from the internet), hence `PAPERCLIP_DEPLOYMENT_EXPOSURE=private`.

Hostnames are kept out of this public repository. They live in a git-ignored
`hostnames.env` next to the zolab Kubernetes overlay and are injected at build time:

```sh
cp deploy/kubernetes/overlays/zolab/hostnames.env.example \
   deploy/kubernetes/overlays/zolab/hostnames.env   # then edit
```

1. Make sure the `eg` listener serving that hostname has TLS and
   `allowedRoutes` that admit the `paperclip` namespace.
2. Create the secrets above, then:

```sh
kubectl apply -k deploy/kubernetes/overlays/zolab
kubectl -n paperclip get cluster,pods,httproute
```

Pin the image in the overlay (`newTag: sha-<commit>`) once Tekton has pushed one.

### Any other cluster

Create an overlay next to `overlays/zolab` that references `../../base` and
the components you need, then set: the image, `storageClassName` of the
`paperclip-data` PVC (and the CNPG Cluster), `PAPERCLIP_PUBLIC_URL`, and
either the `gateway-api` component (patch `parentRefs`/`hostnames`) or your
own Ingress. The base NetworkPolicy admits HTTP on port 3100 from any source;
narrow it to your ingress controller. The Cilium policy in the zolab overlay
is Cilium-specific.

### Agent sandboxes (optional)

Enable `../../components/kubernetes-sandbox` in the overlay. The plugin is
published on npm as `@paperclipai/plugin-kubernetes`. Install it with
`paperclipai plugin install @paperclipai/plugin-kubernetes`, run from a machine
logged in to your instance: the CLI calls the server's authenticated
`/api/plugins/install` endpoint, and the server installs it with the image's
`npm`. Then create a `kubernetes` sandbox environment with `inCluster: true` (see
upstream `packages/plugins/sandbox-providers/kubernetes/README.md`). The
component grants a ClusterRole that includes Secrets and `pods/exec` in all
namespaces, because the provider creates one namespace per company at run time.
Agent pods call back to the server, so also allow ingress to port 3100 from
the `paperclip-*` tenant namespaces.

## Tekton CI

Needs Tekton Pipelines and Tekton Triggers (with core interceptors) installed.

Pipeline `paperclip-image`: BuildKit (rootless) builds straight from the git
commit with SBOM and provenance attestations and a registry cache, then pushes
`sha-<commit>`. Trivy fails the run on fixable HIGH/CRITICAL findings, and
only a clean image is tagged `latest` (push to `main`) or `vX.Y.Z` (push of a
`v*` tag), plus the version of the Paperclip release inside it (e.g.
`2026.916.1`, read from the installed `@paperclipai/server`).

Images are multi-arch (`linux/amd64,linux/arm64`, pipeline parameter
`platforms`). The build node's own architecture builds natively; the other one
runs under the QEMU user emulators bundled in the BuildKit image (the amd64
image emulates arm64 and vice versa), so nodes need no binfmt/QEMU setup and
the build can run on either architecture. Only the small `tools` stage and the
image smoke test run emulated. The heavy `npm install` always runs natively.
Trivy scans every platform.

The `paperclip-ci` namespace is labelled Pod Security `privileged` because
rootless BuildKit needs `Unconfined` seccomp/AppArmor to create user
namespaces. The build still runs as UID 1000 with no added capabilities;
keep that namespace for builds only.

The webhook logic is a standalone `Trigger` (`paperclip-github-push`) in
`paperclip-ci`. It validates the GitHub signature with *its own* secret,
filters for `main` / `v*` pushes, and creates PipelineRuns in `paperclip-ci`
as ServiceAccount `paperclip-triggers`, which may only create PipelineRuns
there. Serve it with one of two EventListener options (below).

1. Secrets:

   ```sh
   kubectl create namespace paperclip-ci --dry-run=client -o yaml | kubectl apply -f -
   # GitHub PAT (classic) with write:packages, or a fine-grained token with packages write
   kubectl -n paperclip-ci create secret docker-registry ghcr-credentials \
     --docker-server=ghcr.io --docker-username=<github user> --docker-password='<token>'
   kubectl -n paperclip-ci create secret generic github-webhook-secret \
     --from-literal=secretToken="$(openssl rand -hex 32)"
   ```

2. Pick an EventListener option and apply your overlay
   (`kubectl apply -k deploy/tekton/overlays/<your overlay>`):

   **Option A: dedicated EventListener.** Add
   `../../components/eventlistener` to the overlay's `components`, then expose
   Service `el-paperclip-github` (port 8080) with your own Ingress/HTTPRoute.

   **Option B: share an EventListener you already run.** Use the base alone
   (as `overlays/zolab` does). Then, on the existing listener
   (`<el-namespace>/<el-name>`):

   ```sh
   # Inspect first: note its ServiceAccount and any namespaceSelector / labelSelector.
   kubectl -n <el-namespace> get eventlistener <el-name> \
     -o jsonpath='{.spec.serviceAccountName}{"\n"}{.spec.namespaceSelector}{"\n"}{.spec.labelSelector}{"\n"}'

   # Also watch Triggers in paperclip-ci. Keep the listener's own namespace
   # and any names already listed: a merge patch replaces the whole array.
   kubectl -n <el-namespace> patch eventlistener <el-name> --type merge \
     -p '{"spec":{"namespaceSelector":{"matchNames":["<el-namespace>","paperclip-ci"]}}}'

   # Tekton requires this ClusterRoleBinding for listeners using namespaceSelector.
   SA="$(kubectl -n <el-namespace> get eventlistener <el-name> -o jsonpath='{.spec.serviceAccountName}')"
   kubectl create clusterrolebinding <el-name>-eventlistener-namespaces \
     --clusterrole=tekton-triggers-eventlistener-roles \
     --serviceaccount="<el-namespace>:${SA}"
   ```

   Caveats:
   - If the listener is managed from git, make the change there, or it will
     be reverted.
   - If it has a `labelSelector`, give the Trigger matching labels.
   - The ClusterRoleBinding lets that listener read Tekton Triggers, create
     PipelineRuns and impersonate ServiceAccounts in every namespace. This is
     the grant upstream Tekton documents for this mode.
   - Webhooks from other repositories share the URL. The Paperclip Trigger
     ignores them (wrong signature), but the listener's existing triggers
     ignore Paperclip's pushes only if they validate their own webhook secret.

3. In GitHub (repository → Settings → Webhooks): the listener's URL, content
   type `application/json`, the secret from step 1, event "Just the push event".
4. Manual run without a push:

   ```sh
   kubectl -n paperclip-ci create -f - <<'EOF'
   apiVersion: tekton.dev/v1
   kind: PipelineRun
   metadata:
     generateName: paperclip-image-manual-
   spec:
     pipelineRef: {name: paperclip-image}
     params:
       - {name: git-url, value: https://github.com/zozo6015/paperclipai.git}
       - {name: git-revision, value: <commit sha>}
       - {name: release-tags, value: [latest]}
     timeouts: {pipeline: 2h0m0s}
     taskRunTemplate: {serviceAccountName: paperclip-build}
     workspaces:
       - name: dockerconfig
         secret:
           secretName: ghcr-credentials
           items: [{key: .dockerconfigjson, path: config.json}]
   EOF
   ```

The first push creates the ghcr package as private. Either make it public or
add an `imagePullSecret` to the `paperclip` ServiceAccount.

BuildKit fetches the build context from GitHub itself, so the repository must
be reachable without credentials (it is public).
