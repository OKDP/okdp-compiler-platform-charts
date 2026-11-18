# okdp-control-plane-server

OKDP package of the control-plane API server. The server is a **Git writer**
for the desired state (instances, connections, projects, catalog in the
deployments repositories) and a **cluster reader** for the observed state
(workloads, HelmReleases or Applications). No database; its local clone (an
emptyDir) is only a cache.

A package in the okdp-compiler format, with no upstream chart: `Chart.yaml`
has no dependency and there is no values file. The server's ConfigMap,
Deployment, Service, ServiceAccount, ClusterRole, the engine's Role, their
bindings and the ValidatingAdmissionPolicy of its namespaces are OKDP's own
objects, written whole in the `objects` of `okdp.yaml`; the
compiled umbrella renders them (`okdp.objects.yaml`). They follow `chart/` of
the okdp-control-plane-server repository (the environment of
`internal/config/config.go`) and read `global.okdp`. What the instance
publishes (its usage) is `okdp.info.yaml`. Its project is one of the
platform's own, privileged (`global.okdp.policy.privileged` of
`platform.yaml`, `okdp-system` in the sandbox): the compile refuses its
cluster-scoped objects, and its engine Role of another namespace, in any
other.

The package takes no upstream values (no `upstream.<chart>`): every object is
computed from the parameters below (`config.schema.json`).

## Parameters

| Parameter | Default | Environment | Description |
|---|---|---|---|
| `gitops.topology` | required | `GITOPS_CONFIG` (mount) | The topology of the deployments repositories: `platform`, `projects` and `output`, each `{url, branch, path}` (`https://`, `ssh://`, `git@host:path`; the projects' `url` and `path` may hold `{project}`), the output in a repository of its own, and `forgejo` (`url`, `owner`), where the server creates a new project's repository in the split topology. Written to the ConfigMap `<release>-gitops`, mounted at `/etc/okdp/gitops/topology.yaml`; the package sets `forgejo.tokenFile` to `/etc/okdp/forgejo/token`. |
| `gitops.forgejoTokenSecret` | `""` | (mount) | Secret of the release namespace whose key `token` is the Forgejo API token, mounted at `/etc/okdp/forgejo/token`. Required with `gitops.topology.forgejo`. |
| `gitops.chartCacheDir` | `/var/cache/okdp/charts` | `GITOPS_CHART_CACHE_DIR` | Where the server pulls the upstream charts it checks the compiled values against, an emptyDir. Empty: the server's default, under its clone. |
| `gitops.engine` | `flux` | `GITOPS_ENGINE` | `flux` or `argocd`; also selects the engine objects the Role `<release>-engine` reads, and its namespace. |
| `gitops.releasesNamespace` | `okdp-releases` | `GITOPS_RELEASES_NAMESPACE` | The Flux HelmReleases, read by the Role `<release>-engine` there (`flux`). |
| `gitops.argocdNamespace` | `argocd` | `ARGOCD_NAMESPACE` | The Argo CD Applications, read by the Role `<release>-engine` there (`argocd`). |
| `gitops.credentialsSecret` | `""` | `GITOPS_CREDENTIALS_DIR` (mount) | Secret with write access to the repositories of the sources and read access to the output's, Flux GitRepository keys (`username`/`password`, `bearerToken`, `identity`/`known_hosts`). Mounted `0440` with the pod `fsGroup: 65534` (the image user), so the non-root server can read it; prefer it to credentials in the topology's URLs. |
| `gitops.sshInsecureIgnoreHostKey` | `false` | `GITOPS_SSH_INSECURE_IGNORE_HOST_KEY` | Sandboxes only. |
| `gitops.insecureHTTPHosts` | `[]` | `GITOPS_INSECURE_HTTP_HOSTS` (joined by `,`) | Hosts a credential (the repositories', the catalogs', the Forgejo token) is sent to over `http://`, in clear: host names or globs of them (`path.Match`), one per item. None by default: every credential for an `http://` URL is refused. Name the exact hosts, e.g. the sandbox's in-cluster Forgejo `forgejo-http.forgejo.svc.cluster.local`; a glob such as `*.svc.cluster.local` trusts every Service of the cluster. |
| `gitops.author.name` / `.email` | `OKDP control plane` / `okdp-control-plane@okdp.io` | `GITOPS_AUTHOR_NAME` / `_EMAIL` | Commit author. |
| `consoleHost` | `okdp-ui.<ingress suffix>` | `ALLOWED_ORIGINS` (`https://<host>`) | Single allowed origin. |
| `logLevel` | `info` | `LOG_LEVEL` | |
| `oidc.groupsClaim` | `groups` | `OIDC_GROUPS_CLAIM` | The claim of the console's tokens holding the caller's groups, a list of group names (Keycloak's groups mapper, *Full group path* off): the names `global.okdp.access.admins` and a project's `access` give. A dotted name is a nested claim. |
| `tls.caBundleSecret` | `""`: `global.okdp.tls.caBundleSecret` | `OKDP_CA_FILE` (mount `/etc/okdp/ca/<caBundleKey>`) | Secret of the release namespace with the CA bundle every TLS client of the server trusts beside the system's (OIDC, Keycloak, Trino, the SQL editor, Forgejo, Git). Empty: the platform's (a trust-manager Bundle's Secret in every namespace); none when that is unset too. |
| `tls.caBundleKey` | `ca.crt` | | Its key, a PEM bundle. |
| `sqlEditor.allowedNetworks` | `[]` | `SQL_ALLOWED_NETWORKS` (joined by `,`) | Networks (CIDRs or addresses) the SQL editor and the connection tests may reach although they are loopback, link-local or the Kubernetes API server, all refused otherwise. |
| `sqlEditor.statementTimeout` | `2m` | `SQL_STATEMENT_TIMEOUT` | One statement's timeout (a Go duration), PostgreSQL's `statement_timeout` too. |
| `sqlEditor.defaultRows` / `.maxRows` | `1000` / `10000` | `SQL_DEFAULT_ROWS` / `SQL_MAX_ROWS` | The row cap of a request that names none, and the largest it may ask. |
| `sqlEditor.maxResultBytes` | `16777216` | `SQL_MAX_RESULT_BYTES` | Past this size of a result's rows, the rest is dropped (truncated). |
| `sqlEditor.maxConcurrentPerUser` | `2` | `SQL_MAX_CONCURRENT_PER_USER` | Statements one user runs at once; past it, `429`. |
| `sqlEditor.tokenExchangeAudience` | `""` | `SQL_TOKEN_EXCHANGE_AUDIENCE` (unset when empty) | The audience of the token a statement on Trino runs with: the caller's token is exchanged at the issuer (RFC 8693, by the client of `keycloak.credentialsSecret`) and forwarded only as a token of this audience; Trino must accept it (the trino package's `oauth2AdditionalAudiences`). Empty: statements on Trino answer `400`, `sql-no-token-exchange`. |
| `secretStores.allowedServers` | `[]` | `SECRET_STORE_ALLOWED_SERVERS` (joined by `,`) | Vault (OpenBao) server URLs (`https://`) a SecretStore of the console may name beside the platform's. |
| `rbac.namespacePolicy` | `true` | | The ValidatingAdmissionPolicy of the server's namespaces (below, Kubernetes >= 1.30). |
| `insecureOciRegistries` | `""` | `INSECURE_OCI_REGISTRIES` | Plain-HTTP registries, sandboxes only. |
| `keycloak.credentialsSecret.name` | `""` | `KEYCLOAK_CLIENT_ID` / `KEYCLOAK_CLIENT_SECRET` (`secretKeyRef`) | Secret with the Keycloak service-account client credentials. Empty: user and group management is off, no env rendered. |
| `keycloak.credentialsSecret.clientIdKey` / `.clientSecretKey` | `client_id` / `client_secret` | | Keys of that Secret; the defaults match `global.okdp.identity.provisioning.keycloak.credentialsSecret`, so the same Secret can be reused. |
| `keycloak.url` / `keycloak.realm` | `""` | `KEYCLOAK_URL` / `KEYCLOAK_REALM` | Keycloak base URL (no `/realms/...`) and realm. Empty: derived by the server from the platform OIDC issuer `<url>/realms/<realm>`. |
| `keycloak.tlsInsecure` | `false` | `KEYCLOAK_TLS_INSECURE` | Skip the Keycloak certificate check. Sandboxes only (`global.okdp.oidc.insecureSkipVerify` also applies when the URL comes from the issuer). |
| `imageRepository` / `imageTag` | `quay.io/okdp/images/okdp-control-plane-server` / `0.10.0` | | An empty `imageTag` is the upstream half of the package version. |
| `resources` | `{}` | | |

Fixed: `PORT` 8093, `PLATFORM_NAMESPACE` = the release namespace,
`GITOPS_CLONE_DIR` (emptyDir). Platform values read from `global.okdp`:
`ingress.suffix` (default console host), `proxy` (`HTTP(S)_PROXY`,
`NO_PROXY`, both cases), `tls.caBundleSecret` (the default CA bundle). The
server reads the rest of the platform values (OIDC...) itself, from
`platform.yaml` in Git.

The Service `<release>` (port 8093) is what the console routes `/api` to.

## Topology

The io topology, the platform and the projects in one input repository,
the output in another:

```yaml
gitops:
  topology:
    platform: {url: https://git.example.com/okdp/okdp-sources.git}
    projects: {url: https://git.example.com/okdp/okdp-sources.git, path: "projects/{project}"}
    output: {url: https://git.example.com/okdp/okdp-output.git}
  credentialsSecret: okdp-gitops-credentials
```

The split topology, a repository per project, which the server creates in
Forgejo:

```yaml
gitops:
  topology:
    platform: {url: http://forgejo-http.forgejo.svc.cluster.local:3000/okdp/okdp-platform.git, branch: main}
    projects: {url: "http://forgejo-http.forgejo.svc.cluster.local:3000/okdp/okdp-project-{project}.git"}
    output: {url: http://forgejo-http.forgejo.svc.cluster.local:3000/okdp/okdp-output.git}
    forgejo: {url: http://forgejo-http.forgejo.svc.cluster.local:3000, owner: okdp}
  forgejoTokenSecret: okdp-forgejo-token
  credentialsSecret: okdp-gitops-credentials
  # Credentials in clear to the in-cluster Forgejo only.
  insecureHTTPHosts: [forgejo-http.forgejo.svc.cluster.local]
```

The file the server reads, `/etc/okdp/gitops/topology.yaml`, is the
`topology` with `forgejo.tokenFile: /etc/okdp/forgejo/token`. A change of the
topology or of the environment restarts the pod (`checksum/config`). The
compile fails when `topology` is not set or lacks one of `platform`,
`projects` and `output`, and when `topology.forgejo` is set without
`forgejoTokenSecret`. There is no single-repository mode: the server no
longer reads `GITOPS_REPO_URL`, `GITOPS_BRANCH` and `GITOPS_PATH` (it
refuses to start with `GITOPS_REPO_URL` set), and the package refuses
`gitops.repoURL`, `gitops.branch` and `gitops.path`: their repository is
the topology's `platform` (`path: <path>`) and `projects` (`path:
<path>/projects/{project}`), the output moved to a repository of its own.

## User and group management

The console's Identity pages (`/api/v1/identity`) manage users and groups of
the platform Keycloak realm through its Admin REST API. The client whose
credentials `keycloak.credentialsSecret` holds must be confidential, with
service accounts enabled, and its service account must hold the
`realm-management` client roles `view-users`, `query-users`, `manage-users`
and `query-groups` (no `manage-clients`: the server registers no client).
Without `keycloak.credentialsSecret.name` the identity routes answer `501`
and `/api/capabilities` reports `identity.userManagement: false`.

The `comment` and `uid` of a user are Keycloak user attributes: on Keycloak
24+ the realm user profile must allow them (unmanaged attributes enabled, or
both attributes declared), otherwise Keycloak silently drops them.

No RBAC change: Keycloak is reached over HTTPS, not through cluster objects.

## RBAC

The package grants the server only what lies outside the projects'
namespaces, as the server's chart does:

| Where | What |
|---|---|
| cluster (ClusterRole `<release>`) | `namespaces`: `get`, `create`, `delete` (a project's Namespace) |
| cluster | the CRD `sparkapplications.sparkoperator.k8s.io` alone: `get` (the SparkApplication form) |
| `gitops.releasesNamespace` (flux) or `gitops.argocdNamespace` (argocd), Role `<release>-engine` | `helmreleases` or `applications`: `get`, `list`, `watch` (the render/sync status) |

With `rbac.namespacePolicy` (the default), the ValidatingAdmissionPolicy
`<release>-namespaces` and its binding let the server's ServiceAccount
create only a Namespace labelled `okdp.io/project` with its name and a Pod
Security level that is not `privileged`, and delete only a Namespace so
labelled: never `kube-system`, a platform namespace or anyone else's.

In each project's namespace the rights are the compiled output's:
okdp-compiler writes there the Role `okdp-control-plane` and its
RoleBinding to the ServiceAccount `global.okdp.controlPlane.serviceAccount`
names (Secrets by name, never listed; pods, their logs and metrics;
events; volumes; SecretStores, ExternalSecrets, SparkApplications). The
ServiceAccount is the release's name, `<project>-<instance>`, in the
project's namespace: for the instance `okdp-control-plane-server` of the
project `okdp-system`, `okdp-system-okdp-control-plane-server`.
`platform.yaml` must name it, and the platform's administrators, whose
groups the server reads as such:

```yaml
# platform.yaml, for the instance okdp-control-plane-server of okdp-system
global:
  okdp:
    controlPlane:
      serviceAccount: {namespace: okdp-system, name: okdp-system-okdp-control-plane-server}
    access:
      admins: [platform_admin]
```

Without `controlPlane.serviceAccount`, the server reads and writes nothing
in the projects' namespaces (`403` from the Kubernetes API). A project the
console creates gets its Namespace at once, but the Role only when the
engine applies the project's compiled manifests.

## Security context

The pod meets Pod Security `restricted`: it runs as 65534:65534 (the image
user, `runAsNonRoot`, `fsGroup: 65534`), with `seccompProfile: RuntimeDefault`,
no privilege escalation, every capability dropped and a read-only root
filesystem. The server writes only its clone, its chart cache (emptyDirs)
and `HOME=/tmp` (emptyDir `tmp`). The pod spec is the compiler's hardened
pod, `okdpPodSpec` (the `podSpec` variable of `okdp.yaml` is its argument:
the containers, volumes, user and group; no `securityContext` is written
by hand), which sets the security contexts and mounts a ServiceAccount
token only when asked: the server's, which it needs to call the API,
`automountServiceAccountToken: true` (it was the default, unwritten). The
Deployment itself, its labels and selector stay the package's.

## Changes

- The pod spec by `okdpPodSpec`, the compiler's fragment of a hardened pod,
  where it was written by hand: the render differs by the explicit
  `automountServiceAccountToken: true` (the default) and the container's
  `runAsNonRoot` and `seccompProfile`, which the pod's already set; the
  Deployment, its selector and labels are unchanged, so the upgrade is in
  place.
- From the okdp-objects chart: the objects are the package's `objects`,
  rendered by the umbrella; their labels no longer carry `helm.sh/chart` and
  `app.kubernetes.io/version`. `checksum/config` hashes the environment and
  the topology.
- New parameters: `gitops.topology`, `gitops.forgejoTokenSecret`,
  `gitops.chartCacheDir`. `gitops.repoURL` is required only without a
  topology; an unknown `gitops.engine` is refused by the schema.
- From the KuboCD package: removed parameters `kubocdNamespace`,
  `releaseInterval`, `releaseTimeout`; no identity-management rules (users,
  groups and OIDC clients live in Keycloak); default image `0.9.0` (the
  Git-backed server).
- 0.10.0-1.0.0: default image `0.10.0` (the tags and schemas of packages in
  private registries); `gitops.topology` required, `gitops.repoURL`,
  `gitops.branch` and `gitops.path` removed, `GITOPS_REPO_URL`,
  `GITOPS_BRANCH` and `GITOPS_PATH` no longer rendered, `GITOPS_CONFIG`
  always (okdp-control-plane-server after 0.10.0 has no single-repository
  mode).
- 0.10.0-1.0.0, with okdp-control-plane-server after 0.10.0: the
  ClusterRole reduced to `namespaces` `get`, `create`, `delete` and the
  SparkApplication CRD by name (was Secrets, ConfigMaps, workloads, pods,
  volumes, events, ESO objects and SparkApplications cluster-wide, and every
  CRD): the projects' namespaces are the compiled Role's; the engine's
  objects read in their namespace only (Role `<release>-engine`); the
  ValidatingAdmissionPolicy `<release>-namespaces`
  (`rbac.namespacePolicy`). New parameters `gitops.insecureHTTPHosts`,
  `oidc.groupsClaim`, `tls.caBundleSecret` and `tls.caBundleKey` (the
  platform's CA bundle by default), `sqlEditor.*`,
  `secretStores.allowedServers`.

## Tests

Its cases, `ci/*-values.yaml`, compile by hand as the repository's
[README](../../../README.md#working-on-a-package) shows.

Every case is compiled in a privileged project (`ci/platform.yaml`).
`ci/split-topology-values.yaml` reaches the sandbox's Forgejo over plain
HTTP (`gitops.insecureHTTPHosts`), `ci/ca-bundle-sql-values.yaml` mounts the
platform's CA bundle and sets the SQL editor's bounds and token exchange,
`ci/flux-https-values.yaml` goes without the namespace policy, and
`ci/insecure-host-list-values.yaml` is refused (one host per item).
