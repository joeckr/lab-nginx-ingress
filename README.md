# lab-nginx-ingress

A Helm wrapper chart repository tailored for deploying a hardened, rootless NGINX Ingress Controller (`ingress-nginx`) to Kubernetes clusters enforcing Pod Security Standards (`restricted` profile).

> **Note**: This repository is an umbrella/wrapper Helm chart wrapping the official upstream [`ingress-nginx`](https://kubernetes.github.io/ingress-nginx) chart. It provides pre-configured, production-hardened defaults for clusters that enforce rootless execution across all namespaces.

---

## Features

- **Helm Wrapper Chart (`chart/`)**:
  - Overrides and configures the official upstream `ingress-nginx` chart (version `4.15.1`, controller `v1.15.1`).
  - **Rootless & Hardened by Default**: Strictly compliant with the Kubernetes **Pod Security Admission (PSA) `restricted` profile**.
  - **No Host Networking**: Sets `hostNetwork: false` and `dnsPolicy: ClusterFirst`, avoiding host namespace violations and ensuring full container isolation.
  - **Unprivileged User Execution**: Runs as non-root `UID 101` (`www-data`), `GID 82`, with `fsGroup: 82`.
  - **Least Privilege Capabilities**: Drops `ALL` capabilities and selectively adds only `NET_BIND_SERVICE` (permitted by PSS `restricted`) to bind ports 80 and 443 without root privileges or `dumb-init` execution errors.
  - **Service-Based Traffic Ingress**: Configured with a `LoadBalancer` service to route incoming traffic without requiring host port mapping.
  - **DaemonSet Architecture**: Deploys as a `DaemonSet` with control-plane tolerations for compact lab environments.
  - **Pre-Configured Ingress Settings**: Controller ConfigMap pre-tuned with `use-forwarded-headers: "true"`, `client-max-body-size: "50m"`, and `ssl-redirect: "false"`.
  - **Simplified Webhook Architecture**: Validation webhooks disabled by default to eliminate admission certgen bootstrap overhead in lab clusters.
- **Environment & Tooling (`mise` & `hk`)**:
  - `mise.toml`: Tool version management (`helm`, `betterleaks`, `addlicense`, `trivy`, `actionlint`, `shellcheck`, `zizmor`, `hk`, `pkl`, `tombi`, `yamllint`) and task aliases.
  - `hk.pkl`: Fast git hooks and project checks enforcing Conventional Commits, branch protection, secret scanning (`betterleaks`), Helm linting, workflow linting (`actionlint`), security audits (`zizmor`), YAML linting (`yamllint`), TOML linting (`tombi`), shell checking (`shellcheck`), and license checks (`addlicense`).
- **GitHub Actions CI (`.github/workflows/`)**:
  - Reusable workflows powered by [`joeckr/ci-templates`](https://github.com/joeckr/ci-templates):
    - `lint.yml`: Lints GitHub Actions workflow syntax (`actionlint`), Conventional Commits (`commitlint`), and shell scripts (`shellcheck`).
    - `security.yml`: Scans commits and PRs for secret leaks (`betterleaks`) and workflow security audits (`zizmor`).
    - `release.yml`: Automated Semantic Versioning, git tagging, release notes, `Chart.yaml` version syncing (`semantic`), and packaging/publishing Helm charts to GitHub Container Registry (GHCR) as OCI artifacts.
    - `test_release.yml`: PR dry-run preview for Semantic Versioning and Helm chart packaging.

---

## Directory Structure

```text
.
├── .github/
│   └── workflows/
│       ├── lint.yml             # Lints workflows, commits, and shell scripts
│       ├── security.yml         # Scans for secrets and audits workflow security
│       ├── release.yml          # Semantic release and Helm chart publishing to GHCR
│       └── test_release.yml     # PR dry-run test for releases and chart packaging
├── chart/
│   ├── Chart.yaml               # Wrapper chart definition with ingress-nginx dependency
│   ├── Chart.lock               # Pinned dependency lockfile
│   ├── values.yaml              # Hardened rootless configuration overrides
│   ├── .helmignore              # Ignore rules for chart packaging
│   └── templates/
│       └── .gitkeep             # Wrapper chart templates placeholder
├── scripts/
│   └── template.sh              # Helper script placeholder
├── compose.yml                  # Local lab service definition
├── hk.pkl                       # hk git hooks and checks configuration
├── mise.toml                    # Mise tools and tasks
└── README.md
```

---

## Security & Hardened Architecture

Modern hardened Kubernetes environments (such as Talos Linux or clusters with Pod Security Admission enabled) enforce least privilege and namespace isolation. This wrapper chart is specifically tailored to meet the Kubernetes **Pod Security Standards (PSS) `restricted` profile**.

### Hardened Container Standards (Kubernetes PSS `restricted`)

Under the Kubernetes **Pod Security Admission (PSA)** `restricted` profile:

- **Must Run As Non-Root**: The pod specification sets `podSecurityContext.runAsNonRoot: true` with non-root UID `101` (`www-data`) and GID `82`. Containers cannot execute as UID 0.
- **No Host Namespaces (`hostNetwork: false`)**: The controller runs inside its own network namespace, preventing unauthorized access to the host's network interfaces and loopback stack.
- **Linux Capabilities (`NET_BIND_SERVICE`)**: The container drops `ALL` capabilities and selectively adds back only `NET_BIND_SERVICE`. This complies with PSS `restricted` (which explicitly allows `NET_BIND_SERVICE`) and avoids kernel `EPERM` ("Operation not permitted") errors when `dumb-init` launches capability-bearing binaries under `allowPrivilegeEscalation: false`.
- **Disallow Privilege Escalation**: Pods enforce `securityContext.allowPrivilegeEscalation: false` to prevent child processes from acquiring additional privileges.
- **RuntimeDefault Seccomp**: Workloads enforce `seccompProfile: { type: RuntimeDefault }` to restrict syscalls to safe defaults.
- **Group Isolation**: Configured with `fsGroup: 82` matching the controller's runtime group.

### Compliance Matrix

| Security Dimension | Restricted Standard Requirement | Implementation in This Repo |
|---|---|---|
| **Workload Execution** | Non-root UID (`runAsNonRoot: true`) | UID `101` (`www-data`), GID `82`, `runAsNonRoot: true` |
| **Group Permissions** | Non-root GID / `fsGroup` | `fsGroup: 82` (`www-data`) |
| **Host Namespaces** | Forbidden (`hostNetwork: false`) | `hostNetwork: false`, `dnsPolicy: ClusterFirst` |
| **Capabilities** | Drop `ALL`, allow only `NET_BIND_SERVICE` | `capabilities.drop: ["ALL"]`, `capabilities.add: ["NET_BIND_SERVICE"]` |
| **Privilege Escalation** | Prohibited (`allowPrivilegeEscalation: false`) | `allowPrivilegeEscalation: false` |
| **Seccomp Profile** | `RuntimeDefault` or `Localhost` | `seccompProfile: { type: RuntimeDefault }` |
| **Service Account Token** | Enabled for controller API access | Managed via upstream chart RBAC & ServiceAccount |
| **Traffic Exposure** | Standard Kubernetes Service | `type: LoadBalancer` exposing ports `80` and `443` |

---

## Testing & Deployment Process

### 1. Build Chart Dependencies

Before linting or rendering, download the upstream `ingress-nginx` dependency tarball:

```bash
mise run helm-d
# or: helm dependency build chart/
```

### 2. Linting & Manifest Validation

Validate chart syntax and render Kubernetes manifests:

```bash
# Lint the chart
mise run helm-l
# or: helm lint chart/

# Render templates to rendered.yaml
mise run helm-t
# or: helm template test chart/ > rendered.yaml
```

### 3. Deploying to the Kubernetes Cluster

Deploy the controller to your active cluster:

```bash
mise run helm-i
# or: helm install test chart/
```

### 4. Verifying Health & Controller Status

Verify that all controller pods run cleanly without restarts:

```bash
# Check pod status
kubectl get pods -l app.kubernetes.io/name=ingress-nginx

# Inspect pod events for any security policy rejections
kubectl describe pod -l app.kubernetes.io/name=ingress-nginx

# Stream controller logs
kubectl logs -l app.kubernetes.io/name=ingress-nginx -f

# Verify the LoadBalancer service
kubectl get svc -l app.kubernetes.io/name=ingress-nginx
```

### 5. Uninstalling from the Cluster

Clean up the deployed release:

```bash
mise run helm-u
# or: helm uninstall test
```

---

## Local Development & Tasks

### Setup

```bash
# Install tools and git hooks
mise run install
```

### Available Tasks

Run tasks with `mise run <task>`:

| Task | Description | Command |
|---|---|---|
| `install` | Install tools and set up git hooks | `hk install --mise` |
| `hk` (or `check`) | Run all linters and hook checks | `hk check --all` |
| `helm-d` | Build Helm chart dependencies | `helm dependency build chart/` |
| `helm-l` | Lint Helm chart | `helm lint chart/` |
| `helm-t` | Render Helm chart templates to `rendered.yaml` | `helm template test chart/ > rendered.yaml` |
| `helm-i` | Install Helm chart to current Kubernetes cluster | `helm install test chart/` |
| `helm-u` | Uninstall Helm chart release from cluster | `helm uninstall test` |
| `trivy-fs` | Scan repository filesystem for security vulnerabilities | `trivy fs .` |

---

## Conventional Commits & Releases

Commits must follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:
- `feat: add new feature` -> Triggers a **minor** release (e.g., `v1.0.0` -> `v1.1.0`).
- `fix: resolve issue` -> Triggers a **patch** release (e.g., `v1.0.0` -> `v1.0.1`).
- `feat!: breaking change` -> Triggers a **major** release.
- `chore:`, `docs:`, `ci:`, `test:`, `refactor:` -> Maintenance changes (no release bump).

Upon merging to `main`, the `release.yml` workflow automatically computes the next version, updates `version` and `appVersion` in `chart/Chart.yaml`, creates a Git tag, publishes a GitHub Release, and packages and pushes the Helm chart to GHCR.

## Support

If you find this project useful, consider supporting my work on [Ko-fi](https://ko-fi.com/joeckr):

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/joeckr)

## License

Please refer to the `LICENSE` file for details.
