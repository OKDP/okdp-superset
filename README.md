[![ci](https://github.com/OKDP/okdp-superset/actions/workflows/ci.yml/badge.svg)](https://github.com/OKDP/okdp-superset/actions/workflows/ci.yml)
[![release-please](https://github.com/OKDP/okdp-superset/actions/workflows/release-please.yml/badge.svg)](https://github.com/OKDP/okdp-superset/actions/workflows/release-please.yml)&ensp;&ensp;
[![Release](https://img.shields.io/github/v/release/OKDP/okdp-superset)](https://github.com/OKDP/okdp-superset/releases/latest)
[![Superset](https://img.shields.io/badge/superset-6.0.0-blue.svg)](https://superset.apache.org/)&ensp;&ensp;
[![Helm](https://img.shields.io/badge/helm-v3-purple.svg)](https://helm.sh/)
[![License Apache2](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](http://www.apache.org/licenses/LICENSE-2.0)
<a href="https://okdp.io">
  <img src="https://okdp.io/logos/okdp-notext.svg" height="20px" style="margin: 0 2px;" />
</a>

<p align="center">
    <img width="400px" height="auto" src="https://okdp.io/logos/okdp-inverted.png" />
</p>

OKDP-flavored distribution of [Apache Superset](https://superset.apache.org/): custom Docker images and a Helm chart that wraps the official one with OAuth2/OIDC, Trino support and externalized secrets.

## What is OKDP Superset?

This repository provides:
- A custom **Docker image** (`quay.io/okdp/superset`) built on top of `apache/superset` with Trino, PostgreSQL and Authlib pre-installed — plus security-patched rebuilds of the upstream `-dockerize` and `-websocket` variants used by the Helm chart.
- A **Helm chart** that wraps the official [Apache Superset chart](https://github.com/apache/superset/tree/master/helm/superset) and adds:
  - Custom OAuth2/OIDC providers (Keycloak, Dex)
  - Optional OAuth2 for Trino
  - Externalized secrets (database, Redis, OAuth2, Superset)
  - High-level configuration overrides

## Prerequisites

- Kubernetes cluster
- [Helm](https://helm.sh/) v3+

## Quick Start

```sh
helm upgrade --cleanup-on-fail --install my-release \
  oci://quay.io/okdp/charts/superset --version 0.15.2-2.0 \
  --set okdp.superset.superset_secret_key=$(openssl rand -base64 42)
```

> The `superset_secret_key` is used by Flask to sign session cookies and encrypt sensitive data in Superset's metadata DB. Generate a strong unique value with `openssl rand -base64 42`.
> Replace `0.15.2-2.0` with the latest chart version from [Releases](https://github.com/OKDP/okdp-superset/releases).

## Installation

See the [Helm chart README](helm/superset/README.md) for the full configuration reference and customization guide.

A ready-to-use sample is provided in [`helm/superset/sample-values.yaml`](helm/superset/sample-values.yaml) (Keycloak OAuth2, ingress, examples loading):

```sh
helm upgrade --cleanup-on-fail --install my-release \
  oci://quay.io/okdp/charts/superset --version 0.15.2-2.0 \
  --values helm/superset/sample-values.yaml
```

## Try it in the OKDP Sandbox

Superset is pre-integrated in the [OKDP Sandbox](https://github.com/OKDP/okdp-sandbox), a complete local Kubernetes data platform (Keycloak, SeaweedFS, Spark, Airflow, JupyterHub, Trino, Superset).

## Images

Published to `quay.io/okdp`:
- `quay.io/okdp/superset:<version>`
- `quay.io/okdp/superset:<version>-dockerize`
- `quay.io/okdp/superset:<version>-websocket`

---

**Built for the OKDP Community**
<a href="https://okdp.io">
  <img src="https://okdp.io/logos/okdp-notext.svg" height="20px" style="margin: 0 2px;" />
</a>
