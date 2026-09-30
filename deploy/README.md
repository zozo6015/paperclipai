# Deploying Paperclip on Kubernetes

```
docker/Dockerfile                      distroless multi-stage image build
deploy/kubernetes/base                 cluster-agnostic manifests (Kustomize)
deploy/kubernetes/components/
  cloudnativepg                        PostgreSQL via the CloudNativePG operator
  gateway-api                          HTTPRoute on an existing Gateway
  kubernetes-sandbox                   OPTIONAL RBAC for agent sandboxes (see below)
deploy/kubernetes/overlays/zolab       the zolab cluster
deploy/tekton/base                     Tekton pipeline + GitHub webhook trigger
deploy/tekton/overlays/zolab           webhook exposure on zolab
```

Images are published to `ghcr.io/zozo6015/paperclipai` by Tekton only; no
GitHub Actions are involved.

## The image

`docker/Dockerfile` fetches upstream
[paperclipai/paperclip](https://github.com/paperclipai/paperclip) at the
commit pinned in `ARG PAPERCLIP_REF`, builds it (pnpm + Rust for the vendored
`paperclip-runnerd`), prunes devDependencies and ships the result on
`gcr.io/distroless/nodejs24-debian13:nonroot` (UID 65532). All base images are
pinned by digest.

It is *distroless plus a minimal tool overlay*, not pure distroless: the
Paperclip server itself executes `git`, `tar` and `sh` (workspace clones in
the heartbeat loop, sandbox payload packaging) and `npm` (plugin installs).
Those binaries and only the shared libraries missing from the distroless base
are copied in, with dpkg metadata so scanners still see them. No package
manager, compiler or coding-agent CLI (claude, codex, …) is included.

Consequence: local agent adapters that shell out to a CLI inside the server
container do not work with this image. Run agents in sandboxes (the
`kubernetes-sandbox` component + upstream plugin) or through remote adapters.

Build locally:

```sh
docker buildx build -f docker/Dockerfile -t paperclip:dev .
# another upstream commit:
docker buildx build -f docker/Dockerfile --build-arg PAPERCLIP_REF=<sha> -t paperclip:dev .
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
  --from-literal=PAPERCLIP_SECRETS_MASTER_KEY="$(openssl rand -base64 32)"
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
`kube-system`, CloudNativePG, `synology-iscsi` for ReadWriteOnce volumes
(`nfs-isolated` is the RWX class; nothing here currently needs RWX).

The CNPG operator runs in `cnpg-system`. Paperclip is private (not reachable
from the internet), hence `PAPERCLIP_DEPLOYMENT_EXPOSURE=private`.

Hostnames are kept out of this public repository. They live in a git-ignored
`hostnames.env` next to each zolab overlay and are injected at build time:

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
prebuilt in the image at `/app/packages/plugins/sandbox-providers/kubernetes`;
install it as a local plugin from that path (the `paperclipai plugin install
--local <path>` CLI talks to the server's authenticated `/api/plugins/install`
endpoint, so run it from a machine logged in to your instance), then create a `kubernetes` sandbox environment with `inCluster: true` (see
upstream `packages/plugins/sandbox-providers/kubernetes/README.md`). The
component grants a ClusterRole that includes Secrets and `pods/exec` in all
namespaces, because the provider creates one namespace per company at run time.
Agent pods call back to the server, so also allow ingress to port 3100 from
the `paperclip-*` tenant namespaces.

## Tekton CI (zolab)

Needs Tekton Pipelines and Tekton Triggers (with core interceptors) installed.

Pipeline `paperclip-image`: BuildKit (rootless) builds straight from the git
commit with SBOM and provenance attestations and a registry cache, then pushes
`sha-<commit>`. Trivy fails the run on fixable HIGH/CRITICAL findings, and
only a clean image is tagged `latest` (push to `main`) or `vX.Y.Z` (push of a
`v*` tag).

The `paperclip-ci` namespace is labelled Pod Security `privileged` because
rootless BuildKit needs `Unconfined` seccomp/AppArmor to create user
namespaces. The build still runs as UID 1000 with no added capabilities;
keep that namespace for builds only.

1. Secrets:

   ```sh
   kubectl create namespace paperclip-ci --dry-run=client -o yaml | kubectl apply -f -
   # GitHub PAT (classic) with write:packages, or a fine-grained token with packages write
   kubectl -n paperclip-ci create secret docker-registry ghcr-credentials \
     --docker-server=ghcr.io --docker-username=zozo6015 --docker-password='<token>'
   kubectl -n paperclip-ci create secret generic github-webhook-secret \
     --from-literal=secretToken="$(openssl rand -hex 32)"
   ```

2. Copy `deploy/tekton/overlays/zolab/hostnames.env.example` to
   `hostnames.env` (git-ignored), set the webhook hostname, then apply:
   `kubectl apply -k deploy/tekton/overlays/zolab`
3. In GitHub (repository → Settings → Webhooks): payload URL
   `https://<webhook hostname>/`, content type `application/json`, the secret from
   step 1, event "Just the push event".
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
