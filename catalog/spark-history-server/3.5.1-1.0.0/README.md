# spark-history-server

OKDP package of the Spark History Server (completed Spark applications, event
logs on S3) behind the OKDP Spark web proxy (`/sparkui` proxies running
applications too). It **consumes** an `s3` connection, a connection the
instance names (the platform's store, in practice) or a bucket it claims,
and provides none.

A package in the okdp-compiler format: the former KuboCD modules are the
upstream charts of `Chart.yaml`, whose values `values.cel.yaml` computes from
the parameters below (`config.schema.json`), over OKDP's defaults
(`<chart>.defaults.yaml`, named in `okdp.yaml`). OKDP's own objects are in the
`objects` of `okdp.yaml`, rendered by the umbrella (`okdp.objects.yaml`):

| Former module | Now | Rendered as |
|---|---|---|
| main | chart `spark-history-server` (`oci://quay.io/okdp/charts/spark-history-server` 1.0.0) | `<release>-spark-history-server` |
| proxy | chart `spark-web-proxy` (`oci://quay.io/okdp/charts/spark-web-proxy` 0.1.0) | `<release>-spark-web-proxy`, ingress `<instance>.<namespace>.<suffix>` |
| (new) oidc-dcr | the `oidcClients` of `okdp.yaml`: the compiler adds chart `oidc-dcr` (the platform's oidc-dcr chart, `oci://quay.io/adaltas/oidc-dcr` 0.6.0 by default) | Job `<release>-oidc-dcr` (pre-install/pre-upgrade hook), only with `global.okdp.oidc.clientProvisioning: dcr` |
| (new) | `objects`: ESO Password generator + ExternalSecret `<release>-auth-cookie` | with OIDC |

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `storage` | (required) | `s3` connection of the store holding the event logs, `s3a://<bucket>/event-logs`, the connection's bucket (`spark-events` when it has none); `internalUrl` is preferred over `apiUrl`. A name or a claim (below); no claim by default. |
| `s3SecretRef` | `""`: the connection's | Secret with `accessKey`/`secretKey`: this instance's own S3 identity. Empty: the `storage` connection's credentials Secret, a claimed bucket's identity; required when it has none. |
| `roleMapping` | empty lists | OIDC groups for the Spark ACLs: `admin_groups`, `history_admin_groups`, `modify_groups`, `view_groups`. |
| `cpu` | `0.5` | CPU limit (vCPU); request 250m. |
| `memoryGi` | `1` | Memory limit (GiB); request 512Mi. |

`storage` names a connection (`connections.<name>` of the platform, the
project or the instance: the platform's S3 store, a connection of
`platform.yaml`, in practice), an output of an instance of the namespace,
or a named claim of the project (`claims.<name>` of `project.yaml`), the
bucket the Spark jobs write their event logs to, whose declaration requests
`paths: [event-logs]` itself (a named claim's request is its
declaration's). Or the instance writes a claim, provisioned by the provider
of its class (`global.okdp.provisioning`): `{}`, of the default class;
`{class: <class>}`; `{provider: <namespace>/<instance>}`: a bucket of its
own, the prefix `event-logs` seeded (the package's `request`, `paths:
[event-logs]`). No claim by default. A claim's connection carries its
credentials Secret, copied into the namespace when its provider is of
another one.

Platform values read from `global.okdp`: `ingress.suffix`, `ingress.className`,
`tls.clusterIssuer`, `tls.caBundleSecret`, and when `oidc.enabled` (default true)
`oidc.issuerUri`, `oidc.scope`, `oidc.usePKCE`, `oidc.insecureSkipVerify`, `oidc.clientProvisioning`,
`oidc.dcr.registrationUrl|authMethod` (dcr). The OAuth client of the history
server's OIDC filter (keys `client_id`, `client_secret`) is read from Secret
`creds-<release>-oauth2` (`clientProvisioning: existing`) or
`<release>-<namespace>-dcr` (`dcr`: registered anonymously by the Job
`<release>-oidc-dcr`, redirect URI `https://<instance>.<namespace>.<suffix>/home`,
grants `authorization_code` and `refresh_token`, the platform scopes but `openid`,
plus `offline_access`). The CA bundle is the platform's,
`tls.caBundleSecret`: a Secret with `ca.crt` and `bundle.p12`, as a
trust-manager Bundle writes one, mounted at `/cacerts`, its `bundle.p12`
the JVM's trust store (`JAVA_TOOL_OPTIONS`), also trusted by the DCR Job by
default. Unset, none is mounted: the platform's certificates, and the S3
endpoint's, must chain to a public CA.

With OIDC, the filter's cookie encryption key is per instance: Secret
`<release>-auth-cookie` (key `cookie-cipher-secret-key`, 32 characters, AES-256),
generated once by an ESO `Password` generator and passed to the filter through
the `AUTH_COOKIE_ENCRYPTION_KEY` environment variable (the filter reads it when
the Spark property `cookie-cipher-secret-key` is unset). Deleting the Secret
rotates the key and only signs every user out.

S3 certificate checking (`-Dcom.amazonaws.sdk.disableCertChecking`) is turned
off only when `global.okdp.oidc.insecureSkipVerify` is true; otherwise the S3
endpoint is verified against the CA bundle (`JAVA_TOOL_OPTIONS` trust store),
or the JVM's own trust store without one.

## Sign-in

With OIDC on, an `oauth2-proxy` (`okdpSignInProxy`, Deployment, Service and
Ingress `<release>-sign-in`) fronts the host and lets in only the groups of
the project (`platform.access.viewers`: the groups of `project.yaml` access
and the platform's administrators) and of `roleMapping`; the chart's own
ingress is off. The history server's and the web proxy's UIs have no group
check of their own (`spark.history.ui.acls` governs the applications, not the
listing; driver UIs under `/sparkui` are unfiltered unless the job sets
`spark.ui.filters`). The OAuth client registers
`https://<instance>.<namespace>.<suffix>/oauth2/callback` beside the filter's
`/home` redirect. No group at all fails the compile.

## Upstream values

Any value of the `spark-history-server` and `spark-web-proxy` charts can be
set per instance, under `upstream.<chart>`, merged over the values computed
from the parameters (`upstream` in `okdp.yaml`):

```yaml
upstream:
  spark-history-server:
    image: {repository: mirror.example.org/okdp/spark}
    tolerations: [{key: dedicated, operator: Exists, effect: NoSchedule}]
    extraEnvs: [{name: EXTRA_FLAG, value: "1"}]    # appended to the chart's env
    config: {spark.history.retainedApplications: 100}
  spark-web-proxy:
    resources: {limits: {memory: 256Mi}}
```

The paths the platform relies on are refused, and the lists carrying the
chart's Secrets and CA bundle are appended to rather than replaced: see
`protect` and `append` in `okdp.yaml` (also listed in the schema
descriptions). Spark properties under `spark-history-server.config` have dots
in their names: the ones the platform sets (OIDC filter, ACLs, event log
directory, S3 endpoint and credentials, UI port) are protected in brackets,
each with the properties that start with it (`config["spark.admin.acls*"]`),
and the `errors` of `okdp.yaml` check that every property is a single-line
scalar with a plain property name (the `config` map is written as a
properties file). An image other than the OKDP Spark image must carry the
OKDP OIDC filter when OIDC is on. No key or value may contain `{{` (the schema
and okdp-compile both refuse it). `oidc-dcr` takes no upstream values.

## Changes from the KuboCD package

- The history Service is `<release>-spark-history-server` (was
  `<release>-main`), the proxy `<release>-spark-web-proxy`.
- OIDC follows `okdp.oidc`: enabled unless `global.okdp.oidc.enabled` is
  false (the package defaulted to false when the key was missing; the platform
  values set it). With OIDC off, the OAuth client Secret is no longer
  required.
- `clientProvisioning: dcr` is supported (the package required
  `creds-<release>-oauth2`).
- The OIDC filter cookie cipher key is generated per instance (it was a
  constant shared by every installation). Upgrading signs the current users
  out once.
- S3 certificate checking follows `global.okdp.oidc.insecureSkipVerify` (it
  was always off).
- Upstream values are given inline only, under `upstream.<chart>` (no
  `upstream.<chart>.yaml` file of the instance any more).
- The CA bundle is the platform's, `tls.caBundleSecret`, none when unset
  (was the Secret `certs-bundle`, always).
- `storage` takes a claim, a bucket of its own, when the instance writes
  one; the event log directory follows the connection's bucket (was
  `s3a://spark-events/event-logs` in OKDP's defaults); `s3SecretRef`
  follows the connection when empty.

## Known gaps

- The sign-in proxy (OIDC on, below) fronts the UI host only; without OIDC
  the UI, and a running job's driver UI under `/sparkui`, are open.
- The values ConfigMaps `<release>-<chart>-values` of the former
  okdp-lib-chart charts are gone: the compiled values (`instance.values.yaml`)
  replace them.

## Tests

Its cases, `ci/*-values.yaml`, compile by hand as the repository's
[README](../../../README.md#working-on-a-package) shows.

`ci/<case>-platform.yaml` holds each case's platform values,
`ci/<case>-connections.yaml` its connections: the cases share none.
`ci/platform-store-values.yaml` names the platform's S3 store (a
connection of `ci/platform-store-platform.yaml`, its bucket and its
credentials Secret the connection's), `ci/claims-values.yaml` claims a
bucket of its own (`{}`), `ci/named-claim-values.yaml` names the project's
bucket `spark-events` (`ci/named-claim-project.yaml`); the claims are bound
to the repository's test provider (`ci/` at its root). Every case trusts
the platform's CA bundle (`tls.caBundleSecret` of its platform values) but
`ci/acls-no-oidc-values.yaml`.
