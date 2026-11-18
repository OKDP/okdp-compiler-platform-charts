# trino

OKDP package of [Trino](https://trino.io): distributed SQL over the project's
Hive and Iceberg catalogs, OAuth2 sign-in against the platform OIDC provider,
optional OPA authorization. It **consumes** `hive`, `iceberg-catalog` and `s3`
connections and **provides** a `trino` connection.

A package in the okdp-compiler format: the former KuboCD modules are the
upstream charts of `Chart.yaml` (its dependencies), whose values
`values.cel.yaml` computes from the parameters (`config.schema.json`), over
OKDP's defaults (`<chart>.defaults.yaml`). `okdp.yaml` holds the platform
values required, what an instance may override, the catalogs computed once
(`vars`), the rules of the instance, the OAuth client, OKDP's own objects and
what the instance publishes:

| Former module | Now | Compiled when |
|---|---|---|
| `main` | chart `trino` (trinodb/charts 1.42.2) | always |
| `internal-secrets` | OKDP objects: ESO Password generator + ExternalSecret `<release>-internal` | always |
| `opa` | chart `opa-kube-mgmt` 11.0.12, and the OKDP object NetworkPolicy `<release>-opa` | `enableOPA` |
| `opal-secrets` | OKDP objects: the opal-secrets Job (idempotent pre-install/pre-upgrade hook) | `enableOPA` and `enableOPAL` |
| `opal` | chart `opal` 0.0.30 | `enableOPA` and `enableOPAL` |
| `oidc-dcr` | `oidcClients` of `okdp.yaml`: the compiler adds the chart `oidc-dcr` (the platform's oidc-dcr chart, 0.6.0 by default): Job `<release>-oidc-dcr` (pre-install/pre-upgrade hook) | `global.okdp.oidc.clientProvisioning: dcr` |

OKDP's objects are rendered by the instance's umbrella chart
(`okdp.objects.yaml`), with the upstream charts.

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `hiveCatalogs[]` | `[]` | `{name, metastore (hive connection), storage (s3 connection), s3SecretRef?}` |
| `icebergCatalogs[]` | `[]` | `{name, catalog (iceberg-catalog connection), storage (s3 connection), warehouse?, oidcSecretRef?, oauth2Scope?, s3SecretRef?}` |
| `s3SecretRef` | `""` | Default S3 identity (Secret with `accessKey`/`secretKey`) of the catalogs. |
| `oauth2AdditionalAudiences` | `[]` | Audiences of the tokens the OAuth2 authenticator accepts beside the instance's client: see Bearer tokens and the audience. |
| `numWorkers` | `1` | Workers. |
| `workerCpu` / `workerMemoryGi` | `0.5` / `2` | Worker requests; limits are twice the requests. |
| `coordinatorMemoryGi` | `8` | Coordinator memory limit (request 1 GiB). |
| `enableOPA` | `true` | OPA access control (`opaPolicyPath`, `enableOPADebugLogs`: OPA decision logs and Trino's `opa.log-requests`/`opa.log-responses`, off by default). |
| `enableOPAL` | `false` | Feed OPA from a policy repository (`policyRepoUrl`, `policyRepoMainBranch`, `OPAL_POLICY_SUBSCRIPTION_DIRS`, `OPAL_POLICY_REPO_MANIFEST_PATH`); off, kube-mgmt loads the instance's labelled ConfigMaps (see OPA's API). |

A connection reference names a connection (`connections.<name>` of the
platform, the project or the instance), or an output another instance of
the namespace publishes: a hive-metastore's or a polaris's, by its name, the
instance's (`hive`, `polaris`). An `s3` reference (`storage`)
may also name a named claim of the project (`claims.<name>` of
`project.yaml`): the bucket the metastore or the Polaris catalogs claim
too, each claimer with its copy of the claim's credentials; or claim a
bucket of its own (`{}`, `{class: <class>}`). A provider of another
namespace is a connection, or a claim's.

Free-text parameters (connection references, secret names, `warehouse`,
`oauth2Scope`, the OPA/OPAL settings) may not contain `{{`, `}}` or line
breaks: they end up in catalog properties that the upstream chart passes
through `tpl`. The fields of the connections they name (an s3 `apiUrl` or
`region`, a metastore's `thriftUri`, an Iceberg catalog's `uri` or `realm`)
are one line too: the contracts refuse a line break (cases `newline-*`).

S3 credentials of a catalog: its `s3SecretRef`, else the release
`s3SecretRef`, else the `secretRef` of its s3 connection. The OAuth client of
an Iceberg catalog: its `oidcSecretRef`, else `creds-<release>-oauth2`.
The warehouse of an Iceberg catalog: its `warehouse`, else the realm its
connection publishes (a polaris output's). A catalog without
S3 credentials or warehouse, or named twice (`tpch` and `tpcds` are built
in), fails the compile (`errors` of `okdp.yaml`).

Platform values read (`global.okdp`): `ingress.suffix`, `ingress.className`,
`tls.clusterIssuer`, `tls.caBundleSecret`, `oidc.{issuerUri,authUrl,tokenUrl,jwksUri,scope,clientProvisioning,dcr}`, `proxy`.

Secrets expected in the namespace: `creds-<release>-oauth2`
(`client_id`/`client_secret`, when `clientProvisioning` is `existing`; with `dcr`
the oidc-dcr Job `<release>-oidc-dcr` writes `<release>-<namespace>-dcr`, same keys), the S3
identities, and the platform's CA bundle, `tls.caBundleSecret`: a Secret
with `ca.crt` and `bundle.p12`, as a trust-manager Bundle writes one,
mounted at `/cacerts` in the coordinator and the workers, its `bundle.p12`
the JVMs' trust store. Unset, none is mounted: the platform's
certificates must chain to a public CA.

## Upstream values

Any value of the `trino` and `opa-kube-mgmt` charts can be set per instance,
under `upstream.<chart>`, merged over the values computed from the parameters
(`upstream` in `okdp.yaml`):

```yaml
upstream:
  trino:
    image: {repository: mirror.example.org/trinodb/trino}
    worker: {tolerations: [{key: dedicated, operator: Exists, effect: NoSchedule}]}
    env: [{name: EXTRA_FLAG, value: "1"}]          # appended to the computed env
    catalogs: {memory: "connector.name=memory\n"}  # added to the generated catalogs
  opa-kube-mgmt:
    resources: {limits: {memory: 256Mi}}
```

The paths the platform relies on are refused, and the lists carrying the
chart's Secrets are appended to rather than replaced: see `protect` and
`append` in `okdp.yaml` (also listed in the schema descriptions). The
switches of opa-kube-mgmt that would make Flux and Argo CD render different
objects (`admissionController`: genCA and genSignedCert; `prometheus`,
`serviceMonitor`: `.Capabilities`) are protected and stay off. An upstream
value wins over the parameter it overlaps (`server.workers` over
`numWorkers`). No key or value may contain `{{` (the schema and okdp-compile
both refuse it). `opal` and `oidc-dcr` take no upstream values.

## Provided connection

`<release>` (contract `trino`), in `okdp.info.yaml`:

```yaml
url: https://<instance>.<namespace>.<suffix>
uri: trino://trino@<instance>.<namespace>.<suffix>:443
internalUri: trino://trino@<release>-trino.<namespace>.svc:8080
catalogs: [tpch, tpcds, <catalog names>]
```

## OPA's API

OPA answers Trino's authorization requests on `<release>-opa:8181`, without
authentication (opa-kube-mgmt's `authz` is off and protected): whoever
reaches the port rewrites who may read what. Two measures:

- a NetworkPolicy `<release>-opa` lets in Trino's coordinator and workers
  and, with `enableOPAL`, the OPAL client, on 8181, and nobody else. It is
  enforced where the cluster's network plugin enforces NetworkPolicies, and
  the policies of a namespace are a union: with the platform's project
  policy (`global.okdp.network.policy`), which lets in the namespace's
  pods, those still reach OPA. The namespace is then the boundary, as for
  any workload of it; the NetworkPolicy closes OPA to the rest of the
  cluster, which the default (no project policy, egress open) leaves
  reaching it.
- without OPAL, kube-mgmt loads only the ConfigMaps of the instance:
  labelled `openpolicyagent.org/policy: <release>-opa` (policies) or
  `openpolicyagent.org/data: <release>-opa` (data), not the `rego` / `opa`
  values every OPA tool uses (the flags are protected). The label is a
  selector, not a boundary: whoever can create ConfigMaps in the namespace
  can set it, so keep that right to the people who may change Trino's
  authorization.

## OPAL's policy repository

`enableOPAL` is off. When it is on, the OPAL server polls a repository for
the policy that decides who reads what in Trino, and the defaults
(`policyRepoUrl: https://github.com/OKDP/okdp-examples.git`,
`policyRepoMainBranch: main`) are **an example to try OPAL with**: a mutable
branch of a repository the platform does not own, whose next commit is
Trino's next authorization. For anything else set `policyRepoUrl` to a
repository of yours, and `policyRepoMainBranch` to a branch (or tag) only
the people who may change that authorization can write.

## Changes from the KuboCD package

- The CA bundle is the platform's, `tls.caBundleSecret`, none when unset
  (was the Secret `certs-bundle`, always).
- Resources are named `<release>-trino-*` (was `<release>-main-trino-*`); the
  OPA service is `<release>-opa`, the OPAL secrets `<release>-opal-*` (were
  shared `opal-*-secret` names).
- The internal shared secret is `<release>-internal`, generated by ESO (was
  `creds-<release>-internal` from kubernetes-secret-generator).
- Connection references name a connection, an output (were
  `kcd-<release>-<output>` Connection objects) or a named claim.
- The opal-secrets Job runs under Pod Security's `restricted` level (uid
  1000, no privilege escalation, every capability dropped, the runtime's
  seccomp profile); its RSA key pair comes from the `ssh-keygen` of the
  OPAL server image (`permitio/opal-server` 0.9.4, by digest), not from
  `linuxserver/openssh-server`; its kubectl is `registry.k8s.io/kubectl`
  by digest. The `opal` chart, which takes no upstream values, runs its
  client, server and broadcast PostgreSQL (`postgres` 18.6-alpine, was
  the floating `alpine`) by digest (`opal.defaults.yaml`).
- The opal-secrets Job is the compiler's `okdpJob` (the `opalJob` variable
  of `okdp.yaml` is its argument; the ServiceAccount, Role and RoleBinding
  of the hook stay the package's), on its hardened pod, where the security
  contexts were written by hand. The render differs by the explicit
  `automountServiceAccountToken: true` (the default; kubectl needs the
  token), `readOnlyRootFilesystem: false` on its four containers (as they
  have run: the OPAL server image and kubectl write outside the volumes;
  not tried read-only in a cluster), the containers' own `runAsNonRoot` and
  `seccompProfile`, which the pod's already set, and
  `ttlSecondsAfterFinished: 300`: a failed Job is kept five minutes for its
  logs, then removed (it was kept until the next hook, a success being
  deleted at once, `hook-succeeded`).

## Known gaps

What the okdp-lib-chart chart did to the rendered objects and values cannot
express:

- the opa-kube-mgmt objects carry no `app.kubernetes.io/instance` label (the
  chart has no label value).

## Tests

Its cases, `ci/*-values.yaml`, compile by hand as the repository's
[README](../../../README.md#working-on-a-package) shows.

The instances a case references are in its tree
(`ci/<case>-instances/`): `internal-hive` the metastore `hive`,
`internal-catalogs` the metastore and the Polaris `polaris`, their
outputs referenced by name, their store the bucket `lake`, a named claim of
the project (`ci/<case>-project.yaml`) bound to the repository's test
provider (`ci/` at its root). `internal-catalogs` checks the polaris
instance against its charts too: polaris-console 0.3.0 must be published
(polaris' known gaps). `ci/catalog-errors-values.yaml` must fail with the
messages of the errors section. Every case trusts the platform's CA bundle
(`ci/platform.yaml`) but `opa-opal` (`ci/opa-opal-platform.yaml`).

## Access guide

This document provides instructions on how to authenticate and interact with the TrinoDB service using the Command Line Interface (CLI) and the HTTP API.

### Authentication with Authorization Flow for Human User

Trino is configured with OAuth2 authentication. Before performing any operations, you must obtain a valid access token.

#### Retrieving the Access Token

1. **Log in** to the Trino web interface.
2. **Retrieve the token** from your browser cookies:
   - Open Developer Tools.
   - Look for the cookie named `__Secure-Trino-Oauth2-Token`.
   - Copy the token value.

3. **Export the token** as an environment variable for use in commands:

   ```bash
   export TOKEN="<PASTE_YOUR_OAUTH2_TOKEN_HERE>"
   ```

   > **Note**: The token is a long JWT string. Ensure you copy the entire value.

### Connection Methods

#### 1. Trino CLI

You can use the native `trino` CLI to run interactive queries. The following command connects to the Trino server securely using the exported token.

**Command:**

```bash
trino \
  --server=https://localhost:8443 \
  --user=usera \
  --access-token=$TOKEN \
  --insecure
```

**Parameters:**
- `--server`: The URL of the Trino coordinator. Use `localhost` if properly forwarded or running locally.
- `--user`: The username to identify as (e.g., `usera`).
- `--access-token`: The OAuth2 bearer token.
- `--insecure`: Skips SSL certificate validation (useful for self-signed certificates in sandbox environments).

---

#### 2. HTTP API (cURL)

You can interact with Trino programmatically using `cURL`. This is useful for testing connectivity or automating queries.

**Example: Show Catalogs**

```bash
curl -sk \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Trino-User: usera" \
  --data-binary 'SHOW CATALOGS' \
  https://trino-<TRINO_SUFFIX>.okdp.sandbox/v1/statement
```

**Header Details:**
- `X-Trino-User`: Specifies the effective user for the transaction.
- `Authorization`: Passes the Bearer token for authentication.

### Bearer tokens and the audience

A token presented directly (`--access-token`, an `Authorization: Bearer`
header) passes only if its audience holds **this instance's OAuth client**
(`http-server.authentication.jwt.required-audience`, the `client_id` of
`creds-<release>-oauth2` or of the DCR Secret), not the realm-wide `account`:
a token minted for another client of the realm is refused. Keycloak puts a
client in the audience only through an **audience mapper**: add one to the
Trino client (or to a client scope of the client that requests the token,
`aud` set to the Trino client id), or bearer tokens are refused. The client
OKDP registers no longer has the `client_credentials` grant: a service
account authenticates with its own client, as below, with the mapper.

A token exchanged for another audience, as the control plane's SQL editor does
(its `sqlEditor.tokenExchangeAudience`), is a user's: it carries no `client_id`
claim, so the JWT authenticator (`principal-field=client_id`) passes it on to
the OAuth2 one, which accepts only the instance's client unless the audience is
listed in `oauth2AdditionalAudiences`
(`http-server.authentication.oauth2.additional-audiences`). With DCR, where the
client is per instance, the parameter is required; the principal is its
`preferred_username`.

### Authentication with Client Credentials Flow for Service Account

#### Keycloak Side Configuration

You must create a specific "Client" for your application or automated script.

- **Client ID**: `trino-m2m-app` (for example).
- **Access Type**: `confidential`.
- **Service Accounts Enabled**: `ON` (This is the crucial option for M2M).
- **Standard Flow Enabled**: `OFF` (No need for browser redirection).
- **Mappers**: Add an audience mapper naming the Trino client (see above), and if needed a "Protocol Mapper" to include a `preferred_username` or `sub` field in the token, as Trino uses it to identify the user executing the query.

> In the **Service Account Roles** tab of the client, you can assign specific roles to this application to manage its permissions in Trino via RBAC.

#### Verification Example

To test the applicative user (M2M), follow these steps:

1. **Obtain the Access Token**:

   ```bash
   export SECRET="<YOUR_CLIENT_SECRET>"
   export CLIENT_ID="trino-m2m-app"
   
   # Adjust the Keycloak URL if needed
   curl -ks -X POST "https://keycloak.okdp.sandbox/realms/okdp/protocol/openid-connect/token" \
    -d "grant_type=client_credentials" \
    -d "scope=openid" \
    -d "client_id=${CLIENT_ID}" \
    -d "client_secret=${SECRET}" | jq
   ```

2. **Export the Token**:

   ```bash
   export TOKEN="<ACCESS_TOKEN_FROM_RESPONSE>"

3. **Test with cURL**:

   ```bash
   # Adjust the Trino URL if needed
   curl -skL -X POST https://trino-<TRINO_SUFFIX>.okdp.sandbox/v1/statement \
     -H "Authorization: Bearer $TOKEN" \
     --data 'SHOW CATALOGS' | jq
   ```

4. **Test with Trino CLI**:

   ```bash
   # Ensure you have port-forwarded Trino to localhost:8443
   trino \
     --server=https://localhost:8443 \
     --user=service-account-${CLIENT_ID} \
     --access-token=$TOKEN \
     --insecure
   ```
