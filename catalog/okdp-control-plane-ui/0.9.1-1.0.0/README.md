# okdp-control-plane-ui

OKDP package of the console, the web UI for basic users. The same ingress
serves the single-page app and routes `/api` to the control-plane server
Service, so the browser stays same-origin and the API has no host of its own.

A package in the okdp-compiler format, with no upstream chart: `Chart.yaml`
has no dependency and there is no values file. The console's Deployment,
Service and Ingress are OKDP's own objects, written whole in the `objects`
of `okdp.yaml`; the compiled umbrella renders them (`okdp.objects.yaml`).
They follow `chart/` of the okdp-control-plane-ui repository (the
environment its `docker-entrypoint.sh` writes into `config.js`) and read
`global.okdp`. What the instance publishes (its URL and usage) is
`okdp.info.yaml`.

The package takes no upstream values (no `upstream.<chart>`): every object is
computed from the parameters below (`config.schema.json`).

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `host` | `okdp-ui.<ingress suffix>` | Console host. The default is the platform's one console and is served in a privileged project only ([Host](#host)). |
| `oidcAuthority` | `global.okdp.oidc.issuerUri` | Issuer the console redirects to (`OIDC_AUTHORITY`). |
| `oidcClientId` | `global.okdp.oidc.clientId`, else `okdp-ui` | `OIDC_CLIENT_ID`. |
| `rolesClaim` | `groups` | ID token claim carrying the roles (`OIDC_ROLES_CLAIM`). |
| `adminRole` | `platform_admin` | Role opening the administration screens (`OIDC_ADMIN_ROLE`). |
| `backendService` | derived | Service `/api` goes to. Empty: this release name with its trailing `okdp-control-plane-ui` replaced by `okdp-control-plane-server` (the server package names its Service after its release: components `okdp-control-plane-ui` and `okdp-control-plane-server` of project `okdp-system` give `okdp-system-okdp-control-plane-server`), else `okdp-control-plane-server`. |
| `backendPort` | `8093` | |
| `imageRepository` / `imageTag` | `quay.io/okdp/images/okdp-control-plane-ui` / `0.9.1` | An empty `imageTag` is the upstream half of the package version. |
| `resources` | `{}` | |

Platform values read from `global.okdp`: `ingress.suffix`,
`ingress.className`, `tls.clusterIssuer` (the last two
required, `okdp.yaml` `require`), `oidc.issuerUri`, `oidc.clientId`, `proxy`.

## Host

A project that is not privileged (`global.okdp.policy.privileged`) serves
only the hosts of the form `<label>.<namespace>.<ingress suffix>` or that the
platform lists for it (`global.okdp.policy.ingressHosts`): the compiler
refuses the others, so that no project takes a host another service, the
identity provider's included, answers at. The default host,
`okdp-ui.<ingress suffix>`, is the platform's one console, the host the
identity provider's client redirects to, and does not fit: the console
installs in the privileged project of the control plane (the sandbox's
`okdp-system`, as the server, whose ClusterRole needs it too), and the
package's test cases run in a privileged project. In a project that is not,
the default host is refused (case `unprivileged-default-host`); set `host`
to `<label>.<namespace>.<ingress suffix>` (case `unprivileged-host`), and
register that host with the identity provider.

## Security context

The pod meets Pod Security `restricted`: it runs as 1001 (the image user,
`runAsNonRoot`), with `seccompProfile: RuntimeDefault`, no privilege
escalation and every capability dropped, a read-only root filesystem. An init
container of the image renders `config.js` and the Content-Security-Policy
into the in-memory volume `/run/okdp-ui`, which nginx, alone in the console
container, mounts read-only; nginx writes its pid and temporary files to
`/tmp`, a volume of its own. It needs an image that reads them there
(okdp-control-plane-ui after 0.9.1). The pod spec is the compiler's
hardened pod, `okdpPodSpec` (the `podSpec` variable of `okdp.yaml` is its
argument: the containers, volumes and user; no `securityContext` is written
by hand), which also mounts no ServiceAccount token: the console serves
static files and never calls the API server. The Deployment, its labels,
the Service and the Ingress stay the package's (the ingress routes `/api`
to the server, which the fragment of a web workload does not).

## Changes

- The pod spec by `okdpPodSpec`, the compiler's fragment of a hardened pod,
  where it was written by hand: the render differs by
  `automountServiceAccountToken: false` (the default ServiceAccount's token
  was mounted) and the containers' own `runAsNonRoot` and
  `seccompProfile`, which the pod's already set; the Deployment, its
  selector and labels are unchanged, so the upgrade is in place.
- From the okdp-objects chart: the objects are the package's `objects`,
  rendered by the umbrella; their labels no longer carry `helm.sh/chart` and
  `app.kubernetes.io/version`.
- From the KuboCD package: `backendService` no longer defaults to
  `okdp-control-plane-server-main` (KuboCD named a Service
  `<release>-<module>`), it follows the server release name;
  `oidcAuthority` defaults to the platform issuer; the TLS Secret is
  `<release>-tls` (was `okdp-ui-tls`).
- 0.9.1-1.0.0: default image `0.9.1` (the cells of an object list typed
  from their schema), the parameters unchanged.

## Tests

Its cases, `ci/*-values.yaml`, compile by hand as the repository's
[README](../../../README.md#working-on-a-package) shows.
