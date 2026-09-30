# paperclipai

Hardened container image and Kubernetes deployment for
[Paperclip](https://github.com/paperclipai/paperclip).

- `docker/Dockerfile` — multi-stage build of upstream Paperclip onto a
  distroless, non-root runtime image (`ghcr.io/zozo6015/paperclipai`)
- `deploy/kubernetes` — Kustomize base, components and the `zolab` overlay
- `deploy/tekton` — Tekton pipeline that builds, scans and publishes the image

See [deploy/README.md](deploy/README.md).
