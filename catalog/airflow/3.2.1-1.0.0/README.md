# airflow

OKDP package of [Apache Airflow](https://airflow.apache.org/) 3.2.1 with the
KubernetesExecutor, OIDC login (Flask AppBuilder) and DAGs from git-sync. It
**consumes** a `database-server` and an `s3` connection, by default a
metadata database of its own, claimed of the provider of the platform's
default class, and provides none.

A package in the okdp-compiler format: the former KuboCD modules are the
upstream chart of `Chart.yaml`, whose values `values.cel.yaml` computes from
the parameters below (`config.schema.json`), over OKDP's defaults
(`airflow.defaults.yaml`), the OAuth client of `okdp.yaml` (`oidcClients`)
and OKDP's own objects (`objects` of `okdp.yaml`, rendered by the umbrella):

| Former module | Now | Compiled when |
|---|---|---|
| `oidc-dcr` | `oidcClients`: the compiler adds chart `oidc-dcr` (the platform's oidc-dcr chart, `oci://quay.io/adaltas/oidc-dcr` 0.6.0 by default): Job `<release>-oidc-dcr` (pre-install/pre-upgrade hook) | only with `global.okdp.oidc.clientProvisioning: dcr` |
| `internal-secrets` | `objects`: ESO `Password` generators + `ExternalSecret` `<release>-internal` | always |
| `main` | chart `airflow` 1.22.0 (`https://airflow.apache.org`), rendered as published, its bundled `postgresql` subchart disabled (`postgresql.enabled: false`, protected): `<release>-*`, ingress `<instance>.<namespace>.<suffix>` | always |

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `db` | `{}`: a database of its own | `database-server` connection of the metadata database, a claim or a name (below). Required of it: its credentials Secret (`secretRef`, `username`/`password`). |
| `metadataSecret` | `""`: built from `db` | Secret with key `connection`: the full SQLAlchemy connection string. Empty: built from `db` (below). |
| `storage` | (required) | `s3` connection the DAGs use (`AWS_ENDPOINT_URL_S3` = `internalUrl`, else `apiUrl`; `AWS_REGION`), a name or a claim (below). |
| `s3SecretRef` | `""`: the connection's | Secret with `accessKey`/`secretKey` for the DAGs. Empty: the `storage` connection's credentials Secret, a claimed bucket's identity; required when it has none. |
| `oidcRoleMapping` | `{}` | OIDC group -> Airflow roles, e.g. `{airflow-admins: [Admin]}`, over the project's groups ([Sign-in](#sign-in)): a group of it replaces that group's roles, and its members sign in. Keys and roles may not contain `{{`, `}}` or line breaks (the API server config goes through the upstream `tpl`). |
| `dagsGitSync` | `{}` | Upstream `dags.gitSync` scalar values: `enabled`, `repo`, `branch`, `ref`, `rev`, `depth`, `maxFailures`, `subPath`, `period`, `wait`, `sshKeySecret`, `knownHosts`, `containerName`, `uid`, `httpPort` (other keys are refused: several reach the upstream `tpl` or are pasted as raw YAML); `ref` is mirrored onto `branch`; `credentialsSecret.name` names the git credentials Secret. |
| `schedulerMemoryGi`, `webserverMemoryGi`, `dagProcessorMemoryGi` | `1` | Memory limits (GiB). |
| `dagProcessorCpuCores` | `0.75` | DAG processor CPU limit. |

`db` and `storage` each take a claim, provisioned by the provider of its
class (`global.okdp.provisioning`): `{}`, of the default class, the default
of `db`; `{class: <class>}`; `{provider: <namespace>/<instance>}`; each with
a `request` the contract's claim schema allows (`x-okdp-claim`: a
database's `extensions`, a bucket's `paths`); or a name: a connection, a
named claim of the project (`claims.<name>` of `project.yaml`, e.g. the
bucket the project's services share), or an output of an instance of the
namespace. A claim's connection carries its credentials Secret, copied
into the namespace when its provider is of another one.

Without `metadataSecret`, the SQLAlchemy connection string is built from
the `db` connection: `postgresql://` (`mysql+mysqldb://` for the `mysql`
engine), its host, port and database, its user and password read by the
pods from its credentials Secret (`AIRFLOW_DB_USER`, `AIRFLOW_DB_PASSWORD`,
expanded by Kubernetes in `AIRFLOW__DATABASE__SQL_ALCHEMY_CONN` and
`AIRFLOW_CONN_AIRFLOW_DB`), the chart's own variables off. The password
must be URL-safe, as the providers' generated ones are; another one goes in
a `metadataSecret`.

Platform values read from `global.okdp`: `ingress.suffix`, `ingress.className`,
`tls.clusterIssuer`, `tls.caBundleSecret`, `oidc.*` (`enabled`, `issuerUri`,
`authUrl`, `tokenUrl`, `scope`, `insecureSkipVerify`, `clientProvisioning`,
`dcr.registrationUrl`, `dcr.authMethod: anonymous`), `proxy.*` (git-sync).
The OAuth client comes from Secret `creds-<release>-oauth2` (existing mode) or
`<release>-<namespace>-dcr` (dcr mode), keys `client_id`/`client_secret`. The
CA bundle is the platform's, `tls.caBundleSecret`: a Secret with `ca.crt`
(and `bundle.p12`), as a trust-manager Bundle writes one, mounted at
`/cacerts` in every Airflow container, `REQUESTS_CA_BUNDLE`,
`SSL_CERT_FILE` and `CURL_CA_BUNDLE` pointing at it; unset, none is mounted
and the platform's certificates must chain to a public CA.

## Sign-in

With `global.okdp.oidc.enabled`, users sign in with the platform's identity
provider, and only the people of the project do: the groups of `access:` in
the project's `project.yaml` (`okdp-project-<namespace>` an editor by
default) and the platform's administrators (`global.okdp.access.admins`),
`platform.access` of the compiler, with the roles of their group: a viewer
group is `Viewer`, an editor group `User`, an admin group (the platform's
administrators included) `Admin`. `oidcRoleMapping` adds groups, or sets
other roles for one of them. The realm's roles (`data_engineer`...) are
every project's: they give no access to this Airflow.

A user none of whose groups has an Airflow role is refused at the sign-in
and is not registered: the API server's `webserver_config.py` sets a
`SECURITY_MANAGER_CLASS` (`OkdpSecurityManager`, a
`FabAirflowSecurityManagerOverride`) whose `auth_user_oauth` checks the
groups claim against `AUTH_ROLES_MAPPING` first, and
`AUTH_USER_REGISTRATION_ROLE` is `Public`, a role with no permission, no
longer `Viewer` (the registration of any user of the realm). The compile
refuses an instance whose project gives no group a role, with no
`oidcRoleMapping` and no platform administrator: nobody could sign in. A
user registered by an earlier version stays in Airflow's database, with
the roles of its next sign-in, which it is refused when it is in no group:
delete it (Security > Users). Not run against a cluster: the sign-in is
the user's to verify (a user of another project, a member of the project).

## Generated secrets

`<release>-internal` (ESO, generated once): `api-secret-key` (32),
`jwt-secret` (64) and `fernet-key`, the base64 of 32 characters (the format the
upstream chart minted with `randAlphaNum` in a pre-install hook; that template
is now disabled through `fernetKeySecretName`). The chart's other random
templates are kept off by protected switches (`okdp.yaml`: the `*SecretName`
keys, `airflowVersion`, `redis.enabled`, `executor`). The Secret, its ExternalSecret
and generators are kept on uninstall (`helm.sh/resource-policy: keep`, Argo
`Delete=false`) and adopted again by the next install: the connections and
variables in the database are encrypted with `fernet-key`.

## Database migrations

`migrateDatabaseJob` stays a plain Job (`useHelmHooks: false`): as a
post-install hook it would wait, under Flux `--wait` and Argo PostSync, for
pods that wait for the migrations. Under Argo it is a `Sync` hook
(`argocd.argoproj.io/hook: Sync`, `BeforeHookCreation`), otherwise the Job
deleted by `ttlSecondsAfterFinished` would leave the application out of sync
and be recreated forever. Helm and Flux ignore these annotations. Migrations
are idempotent.

Argo order (`argocd.argoproj.io/sync-wave`, ignored by Helm and Flux):

| Wave | Objects |
| --- | --- |
| `-1` | `<release>-internal` ExternalSecret and its generators, read by the Job |
| `0` | the migration Job (Sync hook) and the configuration objects |
| `1` | the Deployments and StatefulSets, whose pods wait for the migrations |

The waves of the workloads are the chart's `apiServer`, `scheduler`,
`dagProcessor` and `triggerer` `annotations` values (`values.cel.yaml`); a
component an instance enables through upstream values (statsd, pgbouncer...)
stays in wave `0`, its pods do not wait for the migrations. An ExternalSecret
apply refused by the ESO webhook (not serving yet) fails the operation before
the hook exists, and the Argo retry converges. A failed Job fails the
operation since no workload of its wave is left unhealthy, and the retry
recreates it. The upstream chart has no value for the Job's
`activeDeadlineSeconds` and `backoffLimit`: see Known gaps. Under Flux the Job
is a plain Job that waits for its Secrets as before.

## Upstream values

Any value of the `airflow` chart can be set per instance, under
`upstream.airflow`, merged over the values computed from the parameters
(`upstream` in `okdp.yaml`):

```yaml
upstream:
  airflow:
    images: {airflow: {repository: mirror.example.org/apache/airflow}}
    nodeSelector: {node-role.kubernetes.io/data: ""}
    env: [{name: AIRFLOW__CORE__PARALLELISM, value: "64"}]
    volumes: [{name: plugins, configMap: {name: airflow-plugins}}]  # appended to the CA bundle's
    volumeMounts: [{name: plugins, mountPath: /opt/airflow/plugins}]
    config: {scheduler: {dag_dir_list_interval: "60"}}
```

The paths the platform relies on are refused, and the lists carrying the
chart's CA bundle and proxy variables are appended to rather than replaced: see
`protect` and `append` in `okdp.yaml` (also listed in the schema
description). `extraEnv` is refused (a templated string carrying the
OIDC client and S3 credentials): add variables with `env`, `secret` or
`extraEnvFrom`. An upstream value wins over the parameter it overlaps
(`apiServer.resources.limits.memory` over `webserverMemoryGi`). No key or value
may contain `{{` (the schema and okdp-compile both refuse it). `oidc-dcr`
takes no upstream values.

## Changes from the KuboCD package

- `<release>-internal` (was `creds-<release>-internal`), now also holding the
  Fernet key.
- Ingress TLS Secret `<release>-airflow-tls` (was `airflow-tls`).
- Resource names drop the KuboCD module suffix (`<release>-scheduler`, ...).
- The upstream `broker-url` pre-install hook Secret (unused with the
  KubernetesExecutor) is still rendered, as before.
- `db` is a database of its own by default, claimed, its connection string
  built from it unless `metadataSecret` is set (was required);
  `s3SecretRef` follows the `storage` connection when empty.
- Sign-in: only the project's groups and the platform's
  administrators sign in, with roles by their group, where the realm's roles
  registered every user as `Viewer`; `AUTH_USER_REGISTRATION_ROLE` is
  `Public`. A deployment that relied on the realm-wide roles lists their
  groups in `oidcRoleMapping`.

## Known gaps

What the former packages did to the rendered objects and the format cannot
express (okdp-compiler gap list):

- the migration Job no longer gets `activeDeadlineSeconds: 600` and
  `backoffLimit: 6` (the chart has no value for them): under Argo a run that
  cannot start (a Secret it reads is missing) keeps the operation waiting
  instead of failing it after 10 minutes; the Kubernetes default backoff
  limit is 6 anyway;
- `app.kubernetes.io/instance` now comes from the airflow chart's `labels`
  value: the chart also puts it on the KubernetesExecutor task pods
  (`pod_template_file`), and the content checksums of the configuration
  ConfigMaps (`checksum/airflow-config`, `checksum/api-server-config`)
  change with it (one rollout when moving to this format).

## Tests

Its cases, `ci/*-values.yaml`, compile by hand as the repository's
[README](../../../README.md#working-on-a-package) shows.

`ci/claims-values.yaml` claims its metadata database by default (the
connection string built from it) and names the project's bucket `lake`
(`ci/claims-project.yaml`), bound to the repository's test providers (`ci/`
at its root); `ci/mysql-values.yaml` builds a MySQL connection string from
an external connection; the others give a `metadataSecret`.
`ci/no-connections-values.yaml` and `ci/oidc-endpoints-values.yaml` must
fail with the messages of the errors section, as `ci/no-group-values.yaml`
(`access: {}`, no administrator: nobody could sign in);
`ci/project-groups-values.yaml` signs in the groups of its project file
(`ci/project-groups-project.yaml`) and the platform's administrators with no
`oidcRoleMapping`. Every case trusts the
platform's CA bundle (`tls.caBundleSecret` of its platform values) but
`ci/oidc-off-values.yaml`.
