# hive-metastore

OKDP package of the [Hive Metastore](https://hive.apache.org/): table schemas,
partitions and data locations for Trino, Spark and Hive, over an S3 warehouse
and a SQL database. It **provides** a `hive` connection and **consumes** a
`database-server` and an `s3` connection: by default a database of its
own, claimed of the provider of the platform's default class; the object
store a connection the instance names (the platform's, in practice), or a
bucket it claims.

A package in the okdp-compiler format: `values.cel.yaml` computes the values
of the upstream chart
[`oci://quay.io/okdp/charts/hive-metastore`](https://github.com/okdp/hive-metastore)
1.4.0, the dependency of `Chart.yaml`, from the parameters below
(`config.schema.json`), over OKDP's defaults
(`hive-metastore.defaults.yaml`); `okdp.yaml` holds the rules of the
instance, what it publishes and what an instance may override.

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `db` | `{}`: a database of its own | `database-server` connection hosting the metastore schema, a claim or a name (below). Required of it: its credentials Secret (`secretRef`), with `username` and `password`. |
| `storage` | (required) | `s3` connection of the object store holding the warehouse, a name or a claim (below). No claim by default. |
| `s3SecretRef` | `""`: the connection's | Secret with `accessKey`/`secretKey`: this metastore's own S3 identity. Empty: the `storage` connection's credentials Secret, a claimed bucket's identity; required when it has none. |
| `warehouseBucket` | `""`: the connection's | Bucket used as the warehouse directory. Empty: the `storage` connection's bucket, a claimed bucket's; required when it has none. |
| `cpu` | `0.5` | CPU request (vCPU); the limit is twice the request. |
| `memoryGi` | `0.5` | Memory request (GiB); the limit is twice the request. |

No platform value is read but the classes of the claims
(`global.okdp.provisioning`). `db` and `storage` each take:

- a claim, a database or a bucket of its own, provisioned by the provider
  of its class (cnpg-postgresql for databases, seaweedfs for buckets):
  `{}`, of the default class of the contract, `db`'s default
  (`storage` claims nothing unless the instance writes the claim);
  `{class: <class>}`, a class of the platform or of the project;
  `{provider: <namespace>/<instance>}`; each with a `request`, what the
  contract's claim schema allows (`x-okdp-claim`: a database's
  `extensions`, a bucket's `paths`). The connection the provider publishes
  for the claim, under the claim's ID (`<namespace>.<instance>.db`), holds
  the database or the bucket and its credentials Secret, copied into this
  namespace when the provider is of another one;
- a name: a connection (`connections.<name>` of the platform, the project
  or the instance: the platform's S3 store, a connection of
  `platform.yaml`, in practice), a named claim of the project
  (`claims.<name>` of `project.yaml`, a database or a bucket shared by
  every instance that names it), or an output of another instance of the
  namespace (e.g. `demo-pg-hive`, a database a cnpg-postgresql instance
  publishes).

A claim's provider, or the instance of an output, is the instance this
one's release waits for.

## Upstream values

Any value of the `hive-metastore` chart can be set per instance, under
`upstream.hive-metastore`, merged over the values computed from the
parameters (`upstream` in `okdp.yaml`):

```yaml
upstream:
  hive-metastore:
    image: {repository: mirror.example.org/okdp/hive-metastore}
    tolerations: [{key: dedicated, operator: Exists, effect: NoSchedule}]
    extraEnvRaw: [{name: HADOOP_HEAPSIZE, value: "2048"}]   # appended to the computed env
    s3: {requestTimeout: 60000}
```

The paths the platform relies on are refused (the Service name and port the
`hive` connection publishes, the database and S3 credentials, the storage
backend, the schema Job's hook annotations), and `extraEnvRaw`, which carries
the database user Secret, is appended to rather than replaced: see `protect`
and `append` in `okdp.yaml` (also listed in the schema description). An
upstream value wins over the parameter it overlaps (`resources` over
`cpu`/`memoryGi`). No key or value may contain `{{` (the schema and
okdp-compile both refuse it).

## Provided connection

`<release>` (contract `hive`), in `okdp.info.yaml`:

```yaml
thriftUri: thrift://<release>-hive-metastore.<namespace>.svc:9083
```

A consumer in the same namespace references it by its name, the
instance's (`hive` for `projects/demo/instances/hive/`, the release
`demo-hive`), without a connection file.

## Hooks

The upstream schema initialisation Job is a `post-install,post-upgrade` hook
(Argo: PostSync, run on every sync). It is idempotent: it initialises the
schema only when `metastore_db_properties` is missing.

## Changes from the KuboCD package

- The Service is `<release>-hive-metastore` (was `<release>`).
- The connection published was `kcd-<release>-metastore`; it is now `<release>`.
- `db` claims a database of its own by default; `storage` takes a claim
  too, when the instance writes one; `s3SecretRef` and `warehouseBucket`
  follow the `storage` connection when empty (were required, and `hive`).

## Known gaps

What the okdp-lib-chart chart did to the rendered objects and values cannot
express:

- the values ConfigMap `<release>-hive-metastore-values` (audit of the merged
  values) is gone: the compiled values (`instance.values.yaml`) replace it.

## Tests

Its cases, `ci/*-values.yaml`, compile by hand as the repository's
[README](../../../README.md#working-on-a-package) shows.

`ci/platform-store-values.yaml` claims its database by default, its
warehouse a bucket of the platform's S3 store (a connection of
`ci/platform-store-platform.yaml`, its bucket the connection's, its
credentials from the platform's secret store, `secretRef.remoteKey`), `ci/claim-class-values.yaml` writes the claims out (a class,
a provider, requests),
`ci/named-claims-values.yaml` names the project's named claims
(`ci/named-claims-project.yaml`); the providers are the repository's test
providers (`ci/` at its root). `ci/no-connections-values.yaml`,
`ci/db-without-secret-values.yaml` (what `db` requires) and
`ci/claim-request-values.yaml` (the claim schema) must fail with their
message.
