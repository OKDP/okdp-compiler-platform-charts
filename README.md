[![Release](https://img.shields.io/github/v/release/okdp/platform-charts)](https://github.com/okdp/platform-charts/releases/latest)&ensp;&ensp;
[![Helm](https://img.shields.io/badge/helm-3%20%7C%204-blue.svg)](https://helm.sh/)&ensp;&ensp;
[![Kubernetes](https://img.shields.io/badge/kubernetes-1.30+-blue.svg)](https://kubernetes.io/)&ensp;&ensp;
[![License Apache2](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](http://www.apache.org/licenses/LICENSE-2.0)
<a href="https://okdp.io">
<img src="https://okdp.io/logos/okdp-notext.svg" height="20px" style="margin: 0 2px;" />
</a>

## Overview

This repository is a **catalog** of OKDP platform services: one package per
service version, in the format of
[`okdp-compile`](https://github.com/OKDP/okdp-compiler), which compiles the
values of an instance into the values of each upstream chart the service is
made of, OKDP's own objects and what the instance publishes. The upstream
charts are rendered as published, from their own repositories: nothing of
this repository is published as a chart, everything OKDP stays in Git.

A platform reads the catalog from Git: the `catalogs.yaml` of its deployments
repository, beside `platform.yaml`, lists the catalog repositories
(`repositories: [{name: platform-charts, url: <git url>, ref:
<branch|tag|commit>}]`), and the versions of a service are the
`catalog/<service>/<version>/` directories found at `ref`. Deploying the
instances is done from the deployments repositories, by Flux or Argo CD,
directly or through the OKDP console, which commits there. The reference
layout lives in [`OKDP/okdp-sandbox`](https://github.com/OKDP/okdp-sandbox)
(`gitops/`).

## Structure

```
catalog/<service>/<version>/   one package version, immutable once released
contracts/                     the contracts the packages provide or consume (<contract>.schema.json)
scripts/check-shared.sh        diffs the files the catalogs share with other catalogs checked out beside this one
ci/                            what every test case is compiled with: platform values (ci/platform.yaml),
                               test providers of the claims (ci/catalog/, ci/projects/), not packages
```

| Service | Version | Upstream charts |
|---|---|---|
| [airflow](catalog/airflow/3.2.1-1.0.0) | `3.2.1-1.0.0` | `airflow` 1.22.0 (airflow.apache.org) |
| [hive-metastore](catalog/hive-metastore/4.0.1-1.0.0) | `4.0.1-1.0.0` | `hive-metastore` 1.4.0 (quay.io/okdp/charts) |
| [jupyterhub](catalog/jupyterhub/4.4.2-1.0.0) | `4.4.2-1.0.0` | `jupyterhub` 4.4.2 (hub.jupyter.org), `spark-rbac` 1.0.1 |
| [okdp-examples](catalog/okdp-examples/1.3.0-1.0.0) | `1.3.0-1.0.0` | `okdp-examples` 1.3.0 (quay.io/okdp/charts) |
| [polaris](catalog/polaris/1.3.0-incubating-1.0.0) | `1.3.0-incubating-1.0.0` | `polaris` 1.3.0-incubating (downloads.apache.org), `polaris-admin` 1.0.0 (twice: `polaris-admin`, `polaris-admin-principals`), `polaris-console` 0.3.0 |
| [spark-defaults](catalog/spark-defaults/1.0.0-1.0.0) | `1.0.0-1.0.0` | `spark-defaults` 1.0.0 (quay.io/okdp/charts) |
| [spark-history-server](catalog/spark-history-server/3.5.1-1.0.0) | `3.5.1-1.0.0` | `spark-history-server` 1.0.0, `spark-web-proxy` 0.1.0 (quay.io/okdp/charts) |
| [spark-operator](catalog/spark-operator/2.5.2-1.0.0) | `2.5.2-1.0.0` | `spark-operator` 2.5.2 (kubeflow.github.io) |
| [spark-rbac](catalog/spark-rbac/1.0.1-1.0.0) | `1.0.1-1.0.0` | `spark-rbac` 1.0.1 (quay.io/okdp/charts) |
| [superset](catalog/superset/6.0.0-1.0.0) | `6.0.0-1.0.0` | `superset` 0.22.8 (apache.github.io) |
| [trino](catalog/trino/480.0.0-1.0.0) | `480.0.0-1.0.0` | `trino` 1.42.2, `opa-kube-mgmt` 11.0.12, `opal` 0.0.30 |
| [okdp-control-plane-server](catalog/okdp-control-plane-server/0.10.0-1.0.0) | `0.10.0-1.0.0` | none: OKDP's own objects |
| [okdp-control-plane-ui](catalog/okdp-control-plane-ui/0.9.1-1.0.0) | `0.9.1-1.0.0` | none: OKDP's own objects |

A package (`catalog/<service>/<version>/`, laid out as a Helm umbrella chart
over its upstream charts, never packaged):

```
Chart.yaml                name: <service>, version: <version> (<upstream>-<okdp semver>),
                          dependencies: the upstream charts, each pinned in its own repository
okdp.yaml                 OKDP's part: roles and dependsOn (catalog metadata), require (platform
                          values), defaults, upstream (what an instance may set: protect, append),
                          then vars, errors, objects (OKDP's own objects), instance (URL, usage),
                          provides (connections published), oidcClients (OAuth clients)
values.cel.yaml           the values of each upstream chart, by dependency key, YAML + CEL
<key>.defaults.yaml       OKDP's defaults of the chart <key>, plain YAML
config.schema.json        the parameters: types, defaults, x-ui-* hints, x-okdp-connection-ref
README.md                 parameters, platform values read, charts, known gaps
ci/<case>-values.yaml     test instances (a first line "# error: <message>": a case that must fail)
ci/platform.yaml          the platform values of the cases (global.okdp), when they share them
ci/<case>-platform.yaml   a case's own platform values
ci/<case>-project.yaml    a case's project file: its named claims, classes, connections
ci/<case>-connections.yaml a case's own connections
ci/<case>-instances/      other instances of a case's project, whose outputs it references
ci/<case>-projects/       other projects of a case's tree (project.yaml, instances/): the claimers of
                          other namespaces of a provider (the cases of a package that provisions claims)
```

The format is described in
[`okdp-compiler`](https://github.com/OKDP/okdp-compiler):
[writing a package](https://github.com/OKDP/okdp-compiler/blob/main/docs/02_writing_package.md),
[the service definition](https://github.com/OKDP/okdp-compiler/blob/main/docs/specs/02_service_definition.md),
[inputs and outputs](https://github.com/OKDP/okdp-compiler/blob/main/docs/specs/01_inputs_outputs.md).

## Values

A compile reads the context, by level: the platform's values (`global.okdp`:
ingress, OIDC, proxy, TLS, storage classes, the classes of the claims) and
connections, a project's (its connections, its named claims,
its classes), an instance's own connections; then the instance's values, its
parameters and its upstream values (`upstream.<key>`).

A connection reference parameter (`x-okdp-connection-ref`) names a
connection of the context, a named claim of the project, or an output
another instance of the namespace publishes, by its name: a provider
publishes its connections (`provides`), the instance's name by default
(`hive` for the `hive` metastore of the project `demo`, the release
`demo-hive`), a name of their own for several (a claim's ID). No connection is computed from a
name: the contracts have no naming convention. The compiler resolves the
reference against its contract (`contracts/`) and what the package requires
of the connection (`require`), before the mapping runs, and prints the
instances it depends on (on stderr), whose releases this one waits for.

A contract's text fields (URLs, hosts, realms, regions, names, the items of
its lists) refuse a carriage return and a line feed (`pattern:
^[^\r\n]*$`): packages write a connection's values into configuration files
(Trino's catalog properties, the metastore's), where a second line would be
a setting of its own. The compiler refuses such a connection whoever
declared it, a context file or a claim's provider; a new text field of a
contract takes the same pattern. Secrets (`x-okdp-secret`) are not in a
connection, and a field with an `enum` or its own `pattern` needs none. The
CI cases `newline-*` of trino, hive-metastore and superset show the
refusals.

A `database-server` or `s3` reference may claim its connection instead: a
map in place of a name, `{}` for one of the default class, `{class:
<class>}`, `{provider: <namespace>/<instance>}`, with a `request` (the
contract's `x-okdp-claim`: a database's extensions, a bucket's paths). The
instance that provisions the contract (`provisions`, cnpg-postgresql for
databases, seaweedfs for buckets, in
[sandbox-charts](https://github.com/OKDP/sandbox-charts)) creates a database
or a bucket for each claim bound to it, and publishes its connection, under
the claim's ID, its credentials in a Secret. The classes are the platform's
and the projects' (`global.okdp.provisioning.<contract>`: a default and the
classes by name, each its provider). A service that needs a database of
its own claims it by default, its package's claim template (see its
README); a name instead references an existing connection, an output or a
named claim, `claims.<name>` of the project, one connection for every
instance that names it. No package claims a bucket by default: an `s3`
reference names a connection, the platform's S3 store in practice (a
connection of `platform.yaml`, a store OKDP neither deploys nor manages)
or a project's, or a claim the instance writes. A provider of another
namespace is a connection of the context, or a claim's, its credentials
crossing through the platform's secret store (`global.okdp.secrets.store`).

Each upstream chart then receives, merged in this order: its own defaults
(Helm applies them), OKDP's defaults (`<key>.defaults.yaml`), the values the
mapping computes (`values.cel.yaml`), and the instance's upstream values,
except the paths the package protects (`okdp.yaml`). The compiled values
hold no chart default.

Identity is Keycloak only: `global.okdp.oidc.clientProvisioning` is
`existing` (the OAuth client is created in Keycloak beforehand, its
credentials in a Secret) or `dcr` (dynamic client registration: the compiler
adds one oidc-dcr chart per client of the package's `oidcClients`, rendered
by the platform's oidc-dcr chart, `oci://quay.io/adaltas/oidc-dcr` 0.6.0 by
default).

Ingress and TLS come from the platform's settings alone, whichever ingress
controller and cert-manager the cluster runs (the sandbox's, installed by
its bootstrap, or the cluster's own): `global.okdp.ingress` (`suffix`, `className`),
`global.okdp.tls.clusterIssuer`, the cert-manager ClusterIssuer of the
services' certificates, and `global.okdp.tls.caBundleSecret`, the CA bundle
the services trust: a Secret of each namespace with `ca.crt` (PEM) and
`bundle.p12` (PKCS#12, no password), as a trust-manager Bundle writes one,
mounted at `/cacerts`; the oidc-dcr Jobs trust it by default
(`oidc.dcr.caSecret`). Unset, no bundle is mounted: the platform's
certificates must chain to a public CA. No package names a release, a
namespace or an object of the platform's own services.

OKDP's own objects (generated Secrets, hooks, custom resources, the control
plane's workloads) are the package's `objects`: the compiled umbrella renders
them from `okdp.objects.yaml`, as written. What an instance publishes (URL,
usage, provided connections), which the console lists, is
`okdp.info.yaml`, beside the compiled umbrella in the deployments
repository. Generated passwords come from External Secrets Operator
generators: a compile is deterministic, it never produces a secret.

Each upstream chart is rendered as a subchart of the instance's umbrella,
as published: under Helm 4, a null default nested in a map of a chart's
`values.yaml`, which the values leave unset, stays null in a subchart,
where the chart alone drops it. A template reads both alike, except with
`hasKey`, `toYaml` and `range` over the map.

## Migration from KuboCD

The packages of this catalog replace the KuboCD packages of the same
services. What moved, for someone coming from the KuboCD modules:

| KuboCD | Here |
|---|---|
| A package built and pushed to a registry, one chart per module | A directory `catalog/<service>/<version>/`, read from Git through `catalogs.yaml`: nothing is built, pushed or pulled but the upstream charts |
| The modules of a package: `main` (the service's chart), `opa` and `opal` (trino), `bootstrap`, `principals` and `console` (polaris), `proxy` (spark-history-server), `spark-rbac` (jupyterhub), `noname` (spark-operator) | The charts of `Chart.yaml`, one dependency key each (`polaris-admin` and `polaris-admin-principals` for the two polaris-admin phases), rendered as published; the values of each are computed in `values.cel.yaml`, over OKDP's defaults (`<key>.defaults.yaml`) |
| The module `oidc-dcr` | `oidcClients` of `okdp.yaml`: the compiler adds the oidc-dcr chart in `dcr` mode (`global.okdp.oidc.clientProvisioning`), none in `existing` mode |
| The module `internal-secrets` (a StringSecret), wrapper charts (superset's), secrets drawn with `lookup` and `randAlphaNum` | OKDP's own objects, the `objects` of `okdp.yaml`: External Secrets Operator generators and ExternalSecrets (`okdpGeneratedSecret`), so Flux and Argo CD render the same; a wrapper's settings are folded into the values |
| `{{ .Parameters }}` and `{{ .Context }}` templates, the endpoints of a role (`{{ storage.endpoints.apiUrl }}`, `{{ idp.endpoints.tokenUrl }}`) | `${{ ... }}` CEL over `values`, `global.okdp` (the platform: Keycloak is the only identity provider) and connections of `contracts/` (`x-okdp-connection-ref`): a database, a bucket, a metastore are references to a connection, or claims. The free-text parameters take `${name}` placeholders, a `{{` in an instance value is refused |
| A release of a package, in a namespace | An instance of a project: the release `<project>-<instance>`, in the project's namespace; objects carry no module suffix (`<release>-spark-operator-controller`) |
| The values ConfigMap `<release>-<chart>-values`, the descriptor ConfigMap `<release>-okdp` | The compiled values of the instance, and `okdp.info.yaml` beside them, which the console lists |
| The CA bundle Secret `certs-bundle`, always mounted | `global.okdp.tls.caBundleSecret`, none when unset |
| Sign-in by the realm's roles | The project's groups (`platform.access`) in airflow, with `oidcRoleMapping` over them; each package's README says its own |

Each service's README says what changed from its KuboCD package ("Changes
from the former package", "Known gaps"). `roles` and `dependsOn` of
`okdp.yaml` are kept as catalog metadata: the compiler does not read them, and
the order a deployer applies the instances in comes from the references
between them. `roles` names what a service gives the platform:
`interactive-query` (trino), `data-catalog` (polaris, hive-metastore),
`iceberg-catalog` (polaris), `metastore-catalog` (hive-metastore), `dataviz`
(superset), `airflow`, `notebooks` (jupyterhub), `spark` (spark-history-server),
`sparkOperator`, `spark-defaults`, `spark-rbac`, `examples`, `okdp-server` and
`okdp-ui`. A role no package of this catalog provides is the platform's own or
another catalog's: `ingress` (the ingress controller) and `identity`
(Keycloak) are installed by the platform's bootstrap or the cluster;
`database-server` and `storage` are provided by
[sandbox-charts](https://github.com/OKDP/sandbox-charts) (cnpg-postgresql;
seaweedfs, or an external S3 store, a connection of `platform.yaml`);
`external-secrets` is External Secrets Operator.

## Working on a package

`okdp-compile` compiles trees: the catalog, a platform's values, projects
and their instances, every instance of the tree together
(`okdp-compile --repository <dir>...`, each catalog a `--catalog <dir>`).
A test case is such a tree: this repository, a catalog (`catalog/`,
`contracts/` only: `okdp-compile` refuses a catalog that holds
`platform.yaml` or `projects/`) and a test tree of the case,
`platform.yaml`, the project `demo` and its instance `<service>` (the
release `demo-<service>`), `config.yaml` the case's values, beside the
providers of its claims (`ci/` of the repository).

```bash
export OKDP_CHARTS=~/.cache/okdp/charts     # the chart cache of the checks (helm pull --untar)
pkg=catalog/trino/480.0.0-1.0.0

# a test case, compiled by hand: the case opa-opal of trino
t=$(mktemp -d) && mkdir -p $t/projects/demo/instances/trino
yq eval-all '. as $l ireduce ({}; . * $l)' ci/platform.yaml $pkg/ci/platform.yaml \
  $pkg/ci/opa-opal-platform.yaml > $t/platform.yaml
cp -R ci/catalog ci/projects $t/
printf 'service: trino\nversion: 480.0.0-1.0.0\n' > $t/projects/demo/instances/trino/instance.yaml
cp $pkg/ci/opa-opal-values.yaml $t/projects/demo/instances/trino/config.yaml
okdp-compile --catalog . --repository $t -o /tmp/tree           # every instance of the tree
okdp-compile --catalog . --repository $t --check --charts $OKDP_CHARTS   # against the upstream charts

# render the case's instance as a deployment does
i=/tmp/tree/projects/demo/instances/demo-trino
chart=$(mktemp -d) && cp -R $i/. "$chart" && helm dependency build "$chart"
helm template demo-trino "$chart" -n demo -f $i/instance.values.yaml
```

The cases compile by hand, with okdp-gitops' `okdp-compile`, `helm` and
`yq`; no workflow compiles them. A case's render may hold no object that
the deployer of a project, confined to its namespace, refuses at sync (of
a cluster-scoped kind, ClusterRole, CustomResourceDefinition, a webhook
configuration..., or of another namespace), unless its platform values
make its project, `demo`, privileged: an operator's package, or the
control plane's, gives `global.okdp.policy.privileged: [demo]` in its
`ci/platform.yaml`.

The files the OKDP catalog repositories share are byte-identical in each:
`scripts/check-shared.sh`, `LICENSE`, the workflows `release-please.yml`
and `conventional-commits.yml` of `.github/workflows/`, and the contracts
several catalogs hold (a tree reads the contracts of all its catalogs, and
the compiler refuses two of one name that differ). This repository holds
the canonical copies: a change is made here and copied to every catalog in
the same pass. CI cannot check it, no catalog reading another:
`scripts/check-shared.sh ../community-charts ../sandbox-charts`, the
catalogs checked out side by side, diffs them.

A released version directory is immutable: a change of a service is a new
version, a copy of the directory under its new version (`Chart.yaml`
`version`). The upstream half of the version follows the main upstream
chart (`480.0.0` for Trino 480), the OKDP half is SemVer of the package.
A superseded version whose package is less safe than its successor is deleted,
not kept (okdp-control-plane-server 0.9.0 and okdp-control-plane-ui 0.9.0 were).

## CI and releases

No workflow compiles the packages: their test cases compile by hand
([Working on a package](#working-on-a-package)). Nothing is published.

[`release-please.yml`](.github/workflows/release-please.yml) releases the
repository as a whole: merging its release pull request tags `vX.Y.Z` and
updates `CHANGELOG.md`, a ref a platform's `catalogs.yaml` can pin.
