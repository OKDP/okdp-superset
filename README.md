<!-- Section 0 — Product image -->

<img src="https://superset.apache.org/img/superset-logo-horiz.svg" alt="Apache Superset" height="80" align="right" />

<!-- Section 1 — Badges -->

[![ci](https://github.com/OKDP/okdp-superset/actions/workflows/ci.yml/badge.svg)](https://github.com/OKDP/okdp-superset/actions/workflows/ci.yml)
[![release-please](https://github.com/OKDP/okdp-superset/actions/workflows/release-please.yml/badge.svg)](https://github.com/OKDP/okdp-superset/actions/workflows/release-please.yml)
[![Release](https://img.shields.io/github/v/release/OKDP/okdp-superset)](https://github.com/OKDP/okdp-superset/releases/latest)
[![License Apache2](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](http://www.apache.org/licenses/LICENSE-2.0)
<a href="https://okdp.io">
  <img src="https://okdp.io/logos/okdp-notext.svg" height="20px" style="margin: 0 2px;" />
</a>

<!-- Section 2 — Project name + short description -->

# OKDP Superset

OKDP-flavored distribution of [Apache Superset](https://superset.apache.org/) — a custom Docker image and a Helm chart that wraps the [official Apache Superset chart](https://github.com/apache/superset/tree/master/helm/superset) with OAuth2/OIDC providers (Keycloak, Dex), optional OAuth2 for Trino, and externalized Kubernetes Secrets. Published to [`quay.io/okdp`](https://quay.io/organization/okdp).

<!-- Section 3 — What the project does -->

## What the project does

This repository builds and publishes:

- A **Docker image** ([`docker/Dockerfile-superset`](docker/Dockerfile-superset)) → [`quay.io/okdp/superset`](https://quay.io/repository/okdp/superset), plus security-patched rebuilds of the upstream `-dockerize` ([`docker/Dockerfile-dockerize`](docker/Dockerfile-dockerize)) and `-websocket` ([`docker/Dockerfile-websocket`](docker/Dockerfile-websocket)) variants used as init/websocket sidecars by the Helm chart.
- A **Helm chart** ([`helm/superset/`](helm/superset/)) → [`quay.io/okdp/charts/superset`](https://quay.io/repository/okdp/charts/superset) — depends on the upstream [`superset`](https://apache.github.io/superset) chart (version `0.15.2`, see [`helm/superset/Chart.yaml`](helm/superset/Chart.yaml)) and overrides it with OKDP-specific defaults.

Key characteristics:

- **Built from `apache/superset` with extra Python dependencies pre-installed** — the image bundles `apache-superset[trino,postgres]>=6.0.0,<7.0.0` and `Authlib` ([`docker/requirements.txt`](docker/requirements.txt)) so Trino, PostgreSQL and OAuth2/OIDC work out of the box without per-deploy `pip install`.
- **Custom OAuth2/OIDC providers — Keycloak and Dex** — the chart ships a `CustomSsoSecurityManager` ([`helm/superset/values.yaml#L412-L462`](helm/superset/values.yaml#L412-L462)) injected as a default `configOverride`, with explicit `if provider == 'dex'` / `elif provider == 'keycloak'` branches mapping `groups` / `roles` to Superset roles ([`oauth2_superset_roles_mapping`](helm/superset/values.yaml#L164)).
- **Optional OAuth2 for Trino** as a datasource — wires `DATABASE_OAUTH2_REDIRECT_URI` and `DATABASE_OAUTH2_CLIENTS["Trino"]` from env vars ([`helm/superset/values.yaml#L393-L410`](helm/superset/values.yaml#L393-L410), [`helm/superset/templates/secret-trino-oauth2-env.yaml`](helm/superset/templates/secret-trino-oauth2-env.yaml)).
- **Externalized credentials in five Kubernetes Secrets** — database, Redis, OAuth2, Superset, and Trino OAuth2 ([`helm/superset/templates/secret-*-env.yaml`](helm/superset/templates/)) — so passwords and tokens are managed outside `values.yaml`.
- **Multi-arch images** for `linux/amd64` and `linux/arm64` ([`.github/workflows/docker-publish-template.yml#L119`](.github/workflows/docker-publish-template.yml#L119), [`.github/workflows/docker-build-test-push-template.yml#L98`](.github/workflows/docker-build-test-push-template.yml#L98)).

<!-- Section 4 — Architecture -->

## Architecture

<p align="center">
  <img src="docs/assets/architecture.svg" alt="OKDP Superset — deployment architecture" />
</p>

A `helm install` of the chart creates the Superset web Deployment, the Celery worker Deployment, plus an embedded PostgreSQL ([`helm/superset/values.yaml#L1106`](helm/superset/values.yaml#L1106)) and Redis ([`helm/superset/values.yaml#L1151`](helm/superset/values.yaml#L1151)) — both of which can be disabled to bring your own (`superset.postgresql.enabled: false` / `superset.redis.enabled: false`). External IdPs (Keycloak, Dex) and the Trino coordinator are reached over the network; the chart wires the corresponding env vars and Python `configOverrides` for you.

<!-- Section 5 — Prerequisites -->

## Prerequisites

- Kubernetes cluster (>= 1.19)
- [Helm](https://helm.sh/) >= 3
- `openssl` (to generate the Flask secret key in [Quick Start](#quick-start))

<!-- Section 6 — Quick Start + 6.1 Expected result -->

## Quick Start

```sh
helm upgrade --cleanup-on-fail --install my-release \
  oci://quay.io/okdp/charts/superset --version 0.15.2-2.0 \
  --set okdp.superset.superset_secret_key=$(openssl rand -base64 42)
```

> `okdp.superset.superset_secret_key` is used by Flask to sign session cookies and encrypt sensitive data in the Superset metadata DB. Generate a strong unique value with `openssl rand -base64 42` and treat it as a long-lived secret. Replace `0.15.2-2.0` with the [latest released chart version](https://github.com/OKDP/okdp-superset/releases).

### Expected result

```sh
kubectl get pods
# NAME                                          READY   STATUS      RESTARTS   AGE
# my-release-postgresql-0                       1/1     Running     0          2m
# my-release-redis-master-0                     1/1     Running     0          2m
# my-release-superset-78c5797746-vg8fx          1/1     Running     0          2m
# my-release-superset-init-db-t4fjw             0/1     Completed   0          2m
# my-release-superset-worker-848cd5895d-2svb4   1/1     Running     0          2m
```

Port-forward the Superset web service and check the health endpoint:

```sh
kubectl port-forward svc/my-release-superset 8088:8088
curl -sL -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8088/health
# 200
```

Open <http://127.0.0.1:8088> and log in with the default admin user (`admin` / `admin`, [`helm/superset/values.yaml#L1030-L1036`](helm/superset/values.yaml#L1030-L1036)). **Change the password immediately**, or enable OAuth2 (see [Configuration](#configuration)).

<!-- Section 7 — Installation -->

## Installation

A ready-to-use values file is provided at [`helm/superset/sample-values.yaml`](helm/superset/sample-values.yaml) — Keycloak OAuth2, nginx Ingress with cert-manager (Let's Encrypt staging), examples loading at install time.

```sh
helm upgrade --cleanup-on-fail --install my-release \
  oci://quay.io/okdp/charts/superset --version 0.15.2-2.0 \
  --values helm/superset/sample-values.yaml
```

A complete reference of the chart values is available in the [Helm chart README](helm/superset/README.md) (auto-generated from [`helm/superset/values.yaml`](helm/superset/values.yaml) by [helm-docs](https://github.com/norwoodj/helm-docs)).

<!-- Section 8 — Configuration -->

## Configuration

The chart exposes two top-level value sections:

- **`okdp:`** — OKDP-specific high-level configuration, documented below.
- **`superset:`** — passthrough to the upstream [`superset`](https://apache.github.io/superset) subchart; any upstream value can be set under this prefix.

OKDP-specific keys (all under `okdp:`, defined between [`values.yaml#L25-L185`](helm/superset/values.yaml#L25-L185)):

| Key | Default | Reference | Description |
|:----|:--------|:----------|:------------|
| `okdp.superset.superset_secret_key` | *(insecure placeholder — override)* | [`#L38`](helm/superset/values.yaml#L38) | Flask session cookie signing key. Generate with `openssl rand -base64 42`. |
| `okdp.superset.extra_config` | `PERMANENT_SESSION_LIFETIME = 8h`, `ENABLE_PROXY_FIX = True` | [`#L43`](helm/superset/values.yaml#L43) | Extra Python config appended to `superset_config.py`. |
| `okdp.superset.feature_flags` | *(empty)* | [`#L77`](helm/superset/values.yaml#L77) | Superset feature flags injected into `FEATURE_FLAGS`. |
| `okdp.superset.load_examples.enabled` | `false` | [`#L80`](helm/superset/values.yaml#L80) | If `true`, the init Job loads the upstream Superset example dashboards. |
| `okdp.oauth.enabled` | `false` | [`#L94`](helm/superset/values.yaml#L94) | Toggle OAuth2/OIDC. |
| `okdp.oauth.provider` | `""` | [`#L97`](helm/superset/values.yaml#L97) | One of `keycloak`, `dex`. |
| `okdp.oauth.base_url` / `client_id` / `client_secret` / `scope` / `use_pkce` / `ssl_certificate_verify` | — | [`#L100-L117`](helm/superset/values.yaml#L100-L117) | OAuth2/OIDC client parameters. |
| `okdp.oauth.oauth2_superset_roles_mapping` | *(empty)* | [`#L164`](helm/superset/values.yaml#L164) | Map IdP groups/roles to Superset roles. |
| `okdp.trino.oauth.enabled` | `false` | [`#L170`](helm/superset/values.yaml#L170) | Enable OAuth2 for the Trino datasource. |
| `okdp.trino.oauth.base_url` / `client_id` / `client_secret` / `scope` | — | [`#L173-L184`](helm/superset/values.yaml#L173-L184) | Trino OAuth2 client parameters. |

Upstream Apache Superset chart values (everything else, e.g. `superset.image`, `superset.ingress`, `superset.init.adminUser`, `superset.postgresql.enabled`, `superset.redis.enabled`) are passed through to the [official subchart](https://apache.github.io/superset) — see the [auto-generated Helm chart README](helm/superset/README.md) for the complete reference.

<!-- Section 10 — Images -->

## Images

All artefacts are published under [`quay.io/okdp`](https://quay.io/organization/okdp):

| Artefact | Registry | Description |
|:---------|:---------|:------------|
| Docker image | [`quay.io/okdp/superset`](https://quay.io/repository/okdp/superset) | Apache Superset + Trino + PostgreSQL drivers + Authlib pre-installed. |
| `-dockerize` variant | [`quay.io/okdp/superset:<v>-dockerize`](https://quay.io/repository/okdp/superset) | Security-patched rebuild of `apache/superset:<v>-dockerize` (init utility). |
| `-websocket` variant | [`quay.io/okdp/superset:<v>-websocket`](https://quay.io/repository/okdp/superset) | Security-patched rebuild of `apache/superset:<v>-websocket` (websocket sidecar). |
| Helm chart | [`quay.io/okdp/charts/superset`](https://quay.io/repository/okdp/charts/superset) | Wrapping chart with OKDP defaults. |

### Tag format

| Image | Tag examples |
|:------|:-------------|
| `quay.io/okdp/superset` | `6.0.0`, `6.0.0-1.0`, `6.0.0-dockerize`, `6.0.0-1.0-dockerize`, `6.0.0-websocket`, `6.0.0-1.0-websocket` |
| `quay.io/okdp/charts/superset` | `0.15.2-2.0`, `0.15.2-1.0`, `0.15.0-1.1` |

- `<superset-version>` matches the upstream Apache Superset release (e.g. `6.0.0`).
- `<superset-version>-<okdp-build>` adds an OKDP-specific build number (e.g. `6.0.0-1.0`) — bumped whenever the OKDP layer (`docker/requirements.txt`, base image security patches) changes without a Superset upgrade.
- Chart versions follow `<chart-version>-<okdp-build>` (e.g. `0.15.2-2.0`) where `<chart-version>` matches the upstream Apache Superset Helm chart.

<!-- Section 11 — OKDP integration -->

## OKDP integration

The [OKDP sandbox](https://github.com/OKDP/okdp-sandbox) installs this chart wired to Keycloak, PostgreSQL and Trino at:

→ [`okdp-sandbox/packages/okdp-packages/superset/superset.yaml`](https://github.com/OKDP/okdp-sandbox/blob/main/packages/okdp-packages/superset/superset.yaml)

The sandbox bundles the full stack (Keycloak, SeaweedFS, Spark, Airflow, JupyterHub, Trino, Superset) so you can validate the OAuth2 + Trino integration end-to-end without provisioning an external IdP.

<!-- Section 13 — Build -->

## Build

Build and publication are automated via GitHub Actions ([`.github/workflows/`](.github/workflows/)):

- [`ci.yml`](.github/workflows/ci.yml) — on every PR and push: build the three Docker images (`superset`, `-dockerize`, `-websocket`), lint the Helm chart, regenerate [helm-docs](https://github.com/norwoodj/helm-docs).
- [`release-please.yml`](.github/workflows/release-please.yml) — on merge to `main`, [release-please](https://github.com/googleapis/release-please) opens a release PR that bumps versions and updates the `CHANGELOG`. When merged, the Docker images are pushed to `quay.io/okdp/superset` ([`docker-publish-template.yml`](.github/workflows/docker-publish-template.yml)) and the Helm chart to `quay.io/okdp/charts/superset`.
- [`docker-rebuild.yml`](.github/workflows/docker-rebuild.yml) — weekly rebuild of the latest release tag (Tuesday 05:00 UTC, [`cron: "0 5 * * 2"`](.github/workflows/docker-rebuild.yml)) to pick up upstream base image security patches.
- [`helm-lint-template.yml`](.github/workflows/helm-lint-template.yml) — chart lint + `helm-docs` regeneration.

Images are built and published for `linux/amd64` and `linux/arm64`.

<!-- Section 14 — Test -->

## Test

After running the [Quick Start](#quick-start), check the Superset health endpoint:

```sh
kubectl port-forward svc/my-release-superset 8088:8088 &
curl -sL -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8088/health
```

### Expected result

```
200
```

For a deeper smoke test, run the [Installation](#installation) command with `--values helm/superset/sample-values.yaml` (which sets `okdp.superset.load_examples.enabled=true`) and confirm the example dashboards/datasets are loaded:

```sh
PG_PASS=$(kubectl get secret my-release-superset-db-env -o jsonpath='{.data.DB_PASS}' | base64 -d)
kubectl exec my-release-postgresql-0 -- \
  env PGPASSWORD="$PG_PASS" psql -U superset -d superset -tAc \
  "SELECT 'dashboards', count(*) FROM dashboards UNION ALL SELECT 'slices', count(*) FROM slices;"
```

### Expected result

```
dashboards|11
slices|121
```

<!-- Section 16 — License -->

## License

Apache License 2.0 — see [LICENSE](LICENSE).

<!-- Section 17 — Footer -->

---

**Built 🚀 for the OKDP Community**
<a href="https://okdp.io">
  <img src="https://okdp.io/logos/okdp-notext.svg" height="20px" style="margin: 0 2px;" />
</a>
