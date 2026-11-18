# okdp-examples

OKDP package seeding the medallion lakehouse examples: a
`post-install,post-upgrade` Job creates the buckets and the Spark event log
directory, downloads the NYC TLC trip data into the bronze bucket, declares
the bronze tables in Trino and provisions the Polaris realms (catalogs,
roles, principals). It **consumes** `s3`, `trino` and `iceberg-catalog`
connections and provides none.

A package in the okdp-compiler format: the former KuboCD module `main` is
the upstream chart `oci://quay.io/okdp/charts/okdp-examples` 1.3.0, the
dependency of `Chart.yaml`, whose values `values.cel.yaml` computes from the
parameters (`config.schema.json`), over OKDP's defaults
(`okdp-examples.defaults.yaml`).

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `storage` | (required) | `s3` connection of the store (its `internalUrl`, else `apiUrl`), or an output of an instance of the namespace, by its name. |
| `s3SecretRef` | `""`: the connection's | Secret with `accessKey`/`secretKey`: the job's S3 identity, one that creates buckets. Empty: the `storage` connection's credentials Secret; required when it has none. |
| `trino` | (required) | `trino` connection, or the output of a trino instance of this namespace by its name, the instance's (its `url`). |
| `trinoOidcSecretRef` | (required) | Secret with `client_id`/`client_secret` used against Trino. |
| `polaris` | (required) | `iceberg-catalog` connection, or the output of a polaris instance of this namespace by its name (its `internalUri`, else `uri`, without `/api/catalog`). |
| `polarisRealms` | (required) | polaris-admin model; per realm one principal with role `catalog_admin` and `credentialsSecret: {name, clientIdKey, clientSecretKey}`. |
| `bronzeBucket` / `silverBucket` / `goldBucket` | `bronze` / `silver` / `gold` | Buckets. |
| `bronzePrefix` / `bronzeDataUrl` | `mobility/nyc_tlc` / NYC TLC CloudFront | Bronze layout and source. |
| `sparkEventsLogDir` | `s3a://spark-events/event-logs` | Created if missing. |
| `cpu` / `memoryGi` | `0.5` / `0.5` | Job requests; limits are twice the requests. |

The job creates the buckets it fills (`bronzeBucket`, `silverBucket`,
`goldBucket`) with its own identity: `storage` claims no bucket, as a
claimed bucket comes with an identity limited to it.

Platform values read from `global.okdp`: `oidc.issuerUri`, `proxy`,
`tls.caBundleSecret`: the platform's CA bundle, a Secret with `ca.crt` and
`bundle.p12` as a trust-manager Bundle writes one, mounted at `/cacerts`
(`CA_CERT_PATH` and `CURL_CA_BUNDLE` its `ca.crt`, the trino CLI's trust
store its `bundle.p12`, `TRINO_TRUSTSTORE_ARGS`); unset, none is mounted
and the platform's certificates must chain to a public CA.

The chart signs no user in, so `oidc.clientProvisioning` changes nothing here:
`trinoOidcSecretRef` and the realm's `credentialsSecret` are service accounts
(client credentials grant) whose names the Trino OPA rules and the Polaris
principals refer to (`service-account-<client id>`), so they are created in
Keycloak beforehand in both modes, not registered by DCR (a registered client
gets a generated client id).

## Upstream values

Any value of the `okdp-examples` chart can be set per instance, under
`upstream.okdp-examples`, merged over the values computed from the parameters
(`upstream` in `okdp.yaml`):

```yaml
upstream:
  okdp-examples:
    image: {repository: mirror.example.org/okdp/okdp-examples}
    job: {backoffLimit: 4}
    tolerations: [{key: dedicated, operator: Exists, effect: NoSchedule}]
    extraEnvRaw: [{name: EXTRA_FLAG, value: "1"}]           # appended to the chart's env
    commands:                                               # an extra step
      04-custom: {01-hello: echo hello}
```

The paths the platform relies on are refused (the object names, the Job's hook
annotations, the realms file generated from `polarisRealms`), and the lists
carrying the chart's Secrets and CA bundle are appended to rather than
replaced: see `protect` and `append` in `okdp.yaml` (also listed in the
schema description). An upstream value wins over the parameter it overlaps
(`resources` over `cpu`/`memoryGi`), and a `commands` step of the same group
and name replaces the chart's. No key or value may contain `{{` (the schema and
okdp-compile both refuse it).

## Hooks

The seed Job is a `post-install,post-upgrade` hook (Argo: PostSync, run on
every sync), idempotent (`mc mb ... || true`, `CREATE ... IF NOT EXISTS`).

## Changes from the KuboCD package

- The CA bundle is the platform's, `tls.caBundleSecret`, none when unset
  (was the Secret `certs-bundle`, always); the trino steps pass the trust
  store options of `TRINO_TRUSTSTORE_ARGS`, set with the bundle.
- Objects are named after the release (`fullnameOverride`).
- `s3SecretRef` follows the `storage` connection when empty (was required).
- The upstream chart still names its OPA ConfigMaps
  `okdp-examples-opa-trino-data|rules` whatever the release (pre-existing: two
  instances in one namespace collide).

## Known gaps

What the former packages did that the okdp-compiler format cannot express
the same way:

- the values ConfigMap `<release>-okdp-examples-values` is gone: the compiled
  values replace it.

## Tests

Its cases, `ci/*-values.yaml`, compile by hand as the repository's
[README](../../../README.md#working-on-a-package) shows.

`ci/internal-refs-values.yaml` references the trino and the polaris
instances of its project (`ci/internal-refs-instances/`) by the names of
their outputs: its check checks the polaris instance against its charts
too, polaris-console 0.3.0 must be published (polaris' known gaps).
`ci/no-connections-values.yaml` and `ci/realm-without-admin-values.yaml`
must fail with the messages of the errors section.
`ci/external-refs-proxy-values.yaml` trusts no CA bundle (none in its
platform values), the others the platform's.
