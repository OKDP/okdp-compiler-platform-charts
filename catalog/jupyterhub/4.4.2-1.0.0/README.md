# jupyterhub

OKDP package of [JupyterHub](https://jupyter.org/hub): multi-user notebooks
with OAuth sign-in against the platform OIDC provider, a jupyter-fs S3 file
browser and PySpark kernels running Spark on Kubernetes. It **consumes** an
`s3` connection and provides no connection.

A package in the okdp-compiler format: the former KuboCD modules are the
upstream charts of `Chart.yaml`, whose values `values.cel.yaml` computes from
the parameters (`config.schema.json`), over OKDP's defaults
(`<chart>.defaults.yaml`). OKDP's own objects (`objects` in `okdp.yaml`) are
rendered by the umbrella chart of the instance (`okdp.objects.yaml`):

| Former module | Now | Rendered when |
|---|---|---|
| `main` | chart `jupyterhub` (z2jh 4.4.2) | always |
| `spark-rbac` | chart `spark-rbac` (`oci://quay.io/okdp/charts` 1.0.1): ServiceAccount `<release>-spark` of the PySpark sessions, the chart's Role off | always |
| `oidc-dcr` | the mapping's `oidcClients`: the compiler adds chart `oidc-dcr` (the platform's oidc-dcr chart, `oci://quay.io/adaltas` 0.6.0 by default): Job `<release>-oidc-dcr` (pre-install/pre-upgrade hook) | `global.okdp.oidc.clientProvisioning: dcr` |
| (none) | OKDP's objects: ESO Password generators + ExternalSecret `<release>-hub-generated` (the hub passwords); Role and RoleBinding `<release>-spark` (the PySpark sessions, no secrets) | always |

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `storage` | (required) | `s3` connection of the object store behind the file browser (`apiUrl`): a connection, an output of an instance of the namespace or a named claim of the project (`claims.<name>` of `project.yaml`, a bucket the project's services share), by its name; or a claim, a bucket of its own (`{}`, `{class: <class>}`). |
| `s3SecretRef` | `""`: the connection's | Secret with `accessKey`/`secretKey`: this instance's own S3 identity. Empty: the `storage` connection's credentials Secret, a claimed bucket's identity; required when it has none. |
| `cpu` / `memoryGi` | `0.5` / `1` | Notebook pod guarantees; limits are twice the guarantees. |
| `fileBrowserLocations[]` | `[]` | `{name, uri}` jupyter-fs locations (`s3://bucket`). |
| `oidcRoleMapping` | `{}` | GenericOAuthenticator overrides (`allowed_groups`, `admin_groups`, ...). `allowed_groups` defaults to the project's editors and the platform's administrators (`platform.access`); no group at all fails the compile. |
| `welcomeNotebook` | `""` | `Welcome.ipynb` copied into each home. |
| `sparkConnections[]` | `[]` | `{name, properties, s3SecretRef?}`: Spark conf lines of a PySpark connection. **Was `connections`** (now the external connections map). |
| `pyspark[]` | `[]` | Names of the `sparkConnections` wired into the PySpark kernel. |
| `sparkDefaults` | `{}` | Spark conf defaults added to `PYSPARK_SUBMIT_ARGS`. |
| `sparkPvcPrefixes[]` | `[]` | Name prefixes of the PersistentVolumeClaims the pods of the PySpark sessions may mount (`pvcPrefixes` of `<release>-spark-pod-bounds`); none by default. Never a prefix of the notebooks' homes (`<release>-claim-`): refused. |

Placeholders replaced in `welcomeNotebook`, `sparkDefaults` and the
`properties` of `sparkConnections`, written `${name}` (an instance value
holds no `{{`: okdp-compile refuses it):

| Placeholder | Value | In |
|---|---|---|
| `${suffix}` | the platform's ingress suffix | all three |
| `${release}` | the release name | all three |
| `${namespace}` | the release namespace (also `NAMESPACE` in `sparkDefaults`) | all three |
| `${url}` | the hub URL | `welcomeNotebook` |
| `${connection}` | the PySpark connection's name | `properties` |
| `${storage.apiUrl}`, `${storage.region}` | the `storage` connection's | `properties` |
| `${oidc.tokenUrl}` | `global.okdp.oidc.tokenUrl` | `properties` |

Any other `${...}` (Spark's own `${env:VAR}`) is left as written. In
`properties`, `$(S3_ACCESS_KEY)`/`$(S3_SECRET_KEY)` become the connection's
own variables when it has an `s3SecretRef`.

Platform values read from `global.okdp`: `ingress.suffix`, `ingress.className`,
`tls.clusterIssuer`, `storageClass.workspace` (hub
database and notebook homes), `oidc.authUrl|tokenUrl|userinfoUrl|displayName|clientProvisioning`,
`oidc.dcr.registrationUrl|authMethod` (dcr), `proxy`.

## OAuth client

- `clientProvisioning: existing`: the client is created in Keycloak beforehand,
  Secret `creds-<release>-oauth2` with `client_id`, `client_secret`,
  `JUPYTERHUB_CRYPT_KEY`.
- `clientProvisioning: dcr`: the Job `<release>-oidc-dcr` registers it
  anonymously (`dcr.authMethod: anonymous`; redirect URI
  `https://<instance>.<namespace>.<suffix>/hub/oauth_callback`, grants
  `authorization_code` and `refresh_token`, scopes `profile email groups`) and
  writes Secret `<release>-<namespace>-dcr` (`client_id`, `client_secret`); the
  crypt key of the auth state is then the generated
  `hub.config.CryptKeeper.keys` of `<release>-hub-generated`. The Job trusts
  the platform's CA bundle (`oidc.dcr.caSecret`, `tls.caBundleSecret` by
  default).

The hub verifies the identity provider's certificate (`validate_server_cert`,
off only with `global.okdp.oidc.insecureSkipVerify`) against the platform's CA
bundle, mounted at `/cacerts` in the hub too (`SSL_CERT_FILE`), and the
image's CAs when it is unset.

The CA bundle is the platform's, `global.okdp.tls.caBundleSecret`: a Secret
with `ca.crt` and `bundle.p12`, as a trust-manager Bundle writes one,
mounted at `/cacerts` in the notebooks (`AWS_CA_BUNDLE`,
`REQUESTS_CA_BUNDLE`, `CURL_CA_BUNDLE`, `SSL_CERT_FILE`, the file browser)
and in the PySpark executors (`bundle.p12`, the JVMs' trust store). Unset,
none is mounted: the platform's certificates must chain to a public CA.

## Notebook security

A notebook runs as the image's user `jovyan` (uid 1000, group `users` 100),
never root (no `allow_root`, no `NB_USER`/`NB_UID`/`CHOWN_HOME`, which the
image's `start.sh` applies only when started as root), its home
`/home/jovyan` on a PVC of `global.okdp.storageClass.workspace`. It mounts no
ServiceAccount token: `singleuser.serviceAccountName` is unset (KubeSpawner
mounts a token only for a ServiceAccount it is given) and protected.

The PySpark profile alone runs as `<release>-spark` with its token: its
kernel is the Spark driver (client mode) and creates its executors. Its Role
`<release>-spark` (OKDP's objects) grants `pods`, `services`, `configmaps`
and `persistentvolumeclaims` (`get`, `list`, `watch`, `create`, `delete`,
`deletecollection`), `pods/log` and `events` (read), and no `secrets`, no
`patch` and no `services`: a client-mode driver creates and deletes its
executors' pods, their ConfigMaps and on-demand volumes and changes none of
them (a session that sets `spark.kubernetes.executor.deleteOnTermination=false`,
which Spark then labels with a patch, or submits in cluster mode, which applies
a driver Service and ConfigMaps, needs rights this Role does not give: a
SparkApplication of the Spark operator does it with its own). The executors run as the same ServiceAccount
(`spark.kubernetes.authenticate.executor.serviceAccountName`), which mounts
no token (`automountServiceAccountToken: false`): they reach the API server
for nothing. The chart spark-rbac renders the ServiceAccount only
(`rbac.create: false`): its RoleBinding grants its Role `spark-role`, which
reads the namespace's Secrets, to the group of every ServiceAccount of the
namespace.

The NetworkPolicies of the hub, the proxy and the notebooks are on, which
the CNI must enforce. A notebook takes connections from the hub and the
proxy, and from the PySpark executors of the namespace (label `spark-role:
executor`) to their driver; it reaches DNS, the hub, the internet and the
cluster's private networks (`egressAllowRules.privateIPs: true`: the API
server for the PySpark sessions, the object store, Trino, Polaris and the
identity provider behind the ingress), never the cloud metadata server
(`cloudMetadataServer: false`). `cloudMetadata.blockWithIptables` stays off:
z2jh blocks the metadata server with a privileged `NET_ADMIN` init
container, which the Pod Security level of a project's namespace (`enforce:
baseline`) refuses.

The PySpark profile's token creates pods in the namespace: a pod it
creates could run as another ServiceAccount of the namespace or mount one
of its Secrets, which no Role forbids (Pod Security does not either). The
package bounds them with the ConfigMap `<release>-spark-pod-bounds`
(label `okdp.io/pod-bounds: "true"`; `serviceAccount`, the sessions'
ServiceAccount, and `secrets`, the CA bundle, the S3 identity and those of
the PySpark connections, the Secrets the notebook reads already), the
param of the platform's ValidatingAdmissionPolicy `okdp-pod-bounds`: the
pods of `<release>-spark` run as `<release>-spark`, read no other Secret
(volumes, projected volumes, `secretKeyRef`, `envFrom`), mount no `csi`,
`flexVolume`, `azureFile`, `cephfs`, `rbd` or `iscsi` volume, mount no
PersistentVolumeClaim but those named by a prefix of `sparkPvcPrefixes`
(`pvcPrefixes`, none by default: the sessions mount none, and the
notebooks' homes, `<release>-claim-<user>`, are other pods'), update or
delete only the pods, Services, ConfigMaps and claims that carry Spark's
`spark-app-selector` label (the objects of its executors, not the hub's,
another user's or another instance's), never change the image of a pod, and
`<release>-spark` writes no ConfigMap of the bounds' label. The Role is the
first line, narrowed to what a client-mode driver does, the policy the
second: **its CEL is not exercised in a cluster yet** (the platform's
harness, `sandbox-charts` `scripts/check-bootstrap.sh`, and the kind run are
the checks to come). A policy and its
binding are cluster-scoped, which a project's deployer refuses (and
okdp-compile in the objects of a project that is not privileged): the
platform installs them. **Without them the ConfigMap is inert**, and the
PySpark profile reaches what the namespace's Secrets hold: the hub's
`allowed_groups` are the people who may.

## Generated secrets

The upstream chart generates the proxy token, the cookie secret and the
auth-state keys of its hub Secret with `lookup` + `randAlphaNum` (a new value
at every `helm template`: Argo CD would never settle). The mapping hands the
chart a placeholder for the three, and OKDP's objects generate the real ones
once with ESO (`Password` generators, ExternalSecret
`<release>-hub-generated`, all three keys declared up front since a
`refreshInterval: "0"` Secret is never rewritten; kept on uninstall,
`helm.sh/resource-policy: keep` and Argo `Delete=false`, and adopted again by
the next install, so that a kept or restored hub database still decrypts its
auth state). The hub reads the cookie secret and the auth-state keys from it
first (`hub.existingSecret`), and the proxy token too (Known gaps, closed).
The chart's own hub Secret `<release>-hub` stays
(Helm-managed: it carries the hub configuration, `values.yaml`, which follows
every upgrade); its three password keys hold the placeholder. The pre-pull
hook, whose checksum compares with the previous one (`lookup`), stays off and
protected (`prePuller.hook.enabled`).

## Upstream values

Any value of the `jupyterhub` and `spark-rbac` charts can be set per
instance, under `upstream.<chart>`, merged over the values computed from the
parameters (`upstream` in `okdp.yaml`):

```yaml
upstream:
  jupyterhub:
    hub: {image: {name: mirror.example.org/jupyterhub/k8s-hub}}
    cull: {enabled: true, timeout: 7200}
    singleuser:
      nodeSelector: {workload: notebooks}
      extraEnv: {EXTRA_FLAG: "1"}                   # merged into the chart's env map
      storage:
        extraVolumes: [{name: shared, persistentVolumeClaim: {claimName: shared}}]  # appended
        extraVolumeMounts: [{name: shared, mountPath: /shared}]
      profileList: [{display_name: Minimal, kubespawner_override: {image: quay.io/jupyter/minimal-notebook}}]  # appended
```

A workload identity on the PySpark sessions' ServiceAccount
(`upstream.spark-rbac.serviceAccount.annotations`, an IAM role) is the
platform's to give: the compiler denies the key `serviceAccount`, at any
depth, to the upstream values of a project that is not privileged
(`global.okdp.policy.upstream`), and so a pod's identity
(`serviceAccountName`, `securityContext`...).

The paths the platform relies on are refused (names, the ESO-generated hub
passwords, OIDC sign-in, the ingress host, the notebook env and files the
platform wires, the notebooks' identity and the PySpark sessions'
ServiceAccount), and the lists carrying the hub roles,
the CA bundle volume and the PySpark profile are appended to rather than
replaced: see `protect` and `append` in `okdp.yaml` (also listed in the
schema descriptions). The notebook variables that depend on the instance (the
`<PREFIX>_S3_ACCESS_KEY`/`<PREFIX>_S3_SECRET_KEY` of the PySpark connections
with an `s3SecretRef`, the platform proxy variables) are refused by the
`errors` section of `okdp.yaml`. The env maps (`hub.extraEnv`,
`singleuser.extraEnv`) and `hub.extraConfig` merge key by key. An upstream
value wins over the parameter it overlaps
(`hub.config.GenericOAuthenticator.allowed_groups` over `oidcRoleMapping`,
`singleuser.cpu` over `cpu`). No key or value may contain `{{` (the schema
and okdp-compile both refuse it). `oidc-dcr` takes no upstream values.

## Changes from the former package

- The notebooks run as uid 1000 (was root, `uid: 0` and `allow_root`), their
  home `/home/jovyan` (was `/home/<user>`, the image's user renamed by
  `start.sh`): an existing home PVC is mounted there, its files readable and
  writable through the group `users` (`fsGid: 100`). Its storage class is
  `global.okdp.storageClass.workspace` (was `standard`, hard-coded).
- No ServiceAccount token in a notebook (was the `spark` ServiceAccount's,
  whose Role reads the namespace's Secrets); the PySpark profile and its
  executors run as `<release>-spark` (was `spark`), whose Role has no
  secrets; `SPARK_USER` is `$(JUPYTERHUB_USER)` (was `$(NB_USER)`).
- The NetworkPolicies of the hub and the notebooks are on (were off).
- The pods of the PySpark sessions are bounded by the ConfigMap
  `<release>-spark-pod-bounds`, the param of the platform's policy
  `okdp-pod-bounds` (Notebook security).
- The Role `<release>-spark` has no `patch` and no `services` (it had
  `patch` on `pods`, `configmaps` and `persistentvolumeclaims`, and every
  verb listed on `services`), and the claims the sessions mount are those of
  `sparkPvcPrefixes`, none by default.
- The profiles' notebook images and the PySpark 3.5.6 executors' image
  run by digest (were tags only, pulled `Always`).

- The CA bundle is the platform's, `tls.caBundleSecret`, none when unset
  (was the Secret `certs-bundle`, always).
- The former OAuth client provisioning module is gone: the client is created
  in Keycloak beforehand or registered by DCR (identity is Keycloak only).
- `connections` (PySpark connections) is renamed `sparkConnections`.
- The hub passwords are ESO-generated: an upgrade from the KuboCD release gets
  a new cookie secret and new auth-state keys (users sign in again).
- The placeholders of the free-text parameters are written `${name}`
  (above): the `{{ ... }}` forms of Helm and KuboCD
  (`{{ .Values.global.okdp.ingress.suffix }}`, `{{ .Context.ingress.suffix }}`,
  `{{ storage.endpoints.apiUrl }}`, `{{ idp.endpoints.tokenUrl }}`...) are
  refused by okdp-compile, as any `{{` in an instance value.
- An instance's upstream values are its inline `upstream.<chart>` only: no
  `upstream.<chart>.yaml` file.

## Known gaps

What the okdp-lib-chart chart did to the rendered objects and values cannot
express (okdp-compiler gap list):

- **The proxy token is the generated one (closed).** z2jh 4.4.2 hard-wires
  the `CONFIGPROXY_AUTH_TOKEN` `secretKeyRef` of the hub and proxy Deployments
  to its own Secret `<release>-hub`, whose key holds the placeholder
  `generated-by-external-secrets` (the Deployments still carry that entry; it
  is inert, it is not the token). okdp-lib-chart re-pointed the entry; the chart
  is rendered as published here, so the two processes read the generated token
  (key `hub.config.ConfigurableHTTPProxy.auth_token` of
  `<release>-hub-generated`) another way: the hub through `hub.extraConfig`
  (`okdpProxyAuthToken`: `c.ConfigurableHTTPProxy.auth_token` set from
  `get_secret_value`, over the variable); the proxy through its own variable
  `OKDP_CONFIGPROXY_AUTH_TOKEN` and `NODE_OPTIONS=--import=data:text/javascript,...`,
  a module node preloads, which copies it over `CONFIGPROXY_AUTH_TOKEN` before
  configurable-http-proxy reads its environment. A second
  `CONFIGPROXY_AUTH_TOKEN` entry through `hub.extraEnv` / `proxy.chp.extraEnv`
  is not used: both orders would have the kubelet honour the later entry
  (it builds a map, last wins), but Argo CD applies the Deployment with
  `ServerSideApply=true` and `ServerSideDiff`, and server-side apply refuses a
  duplicate key of the `name`-keyed `env` list (`duplicate entries for key
  [name="CONFIGPROXY_AUTH_TOKEN"]`, structured-merge-diff `typed/validate.go`):
  it would never sync.
- the values ConfigMaps `<release>-<chart>-values` are gone: the compiled
  values replace them.

A fix comes with the standard render: z2jh derives its `component` labels
from `.Template.Name`, which okdp-lib-chart set to `vendor/jupyterhub/templates`
for every file, so the hub, the proxy, their Services and the proxy
NetworkPolicy were all `component: jupyterhub` (the Services selected both
pods). They are `hub`, `proxy`, `proxy-api`, `proxy-public`, `singleuser`,
`ingress` again. The `spec.selector` of the hub and proxy Deployments
changes with them, and a Deployment selector is immutable: an existing
release is migrated by deleting both Deployments before the first sync (the
hub database PVC and the Secrets stay). The hub pod's checksum annotations
change too (they cover the relabelled ConfigMap and Secret).

The PySpark 3.4.4 profile's executor image
`quay.io/okdp/spark-py:spark-3.4.4-python-3.11-scala-2.12-java-17`
(`SPARK_EXECUTOR_IMAGE`) is not published (quay.io/okdp/spark-py stops at
spark-3.4.2 on 3.4): its executors cannot pull it, and it is the one image
of the profiles left by tag.

## Tests

Its cases, `ci/*-values.yaml`, compile by hand as the repository's
[README](../../../README.md#working-on-a-package) shows.

`ci/named-claim-values.yaml` browses the project's bucket `lake`, a named
claim (`ci/named-claim-project.yaml`) bound to the repository's test
provider (`ci/` at its root), its identity the claim's;
`ci/workspace-class-values.yaml` the homes on another workspace class.
`ci/notebook-token-values.yaml` gives the notebooks a ServiceAccount, which
the package refuses. Every case trusts
the platform's CA bundle (`ci/platform.yaml`) but
`ci/existing-oidc-values.yaml`, without one
(`ci/existing-oidc-platform.yaml`).
