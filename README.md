# Kubernetes study lab — dev-jiseok / 이지석

Cloud Club 10th Kubernetes Study, weeks 1–3. Personal private repository.

- `infra/`: kind configuration, week 1 Pod, Argo CD Application.
- `network/`: week 2 Pod networking and Envoy Gateway (80:20) resources.
- `deploy/`: plain YAML, Kustomize, Helm, direct CI and GitOps manifests.
- `app/`: versioned Nginx JSON API.
- `.github/workflows/ci.yml`: SHA-tagged multi-platform GHCR build and delivery.
- `evidence/`: observed outputs; no credentials.

Use only the dedicated `kind-jiseok-netlab` context. Credentials and runner registration are kept outside this repository.
