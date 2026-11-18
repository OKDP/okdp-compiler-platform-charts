# spark-rbac

OKDP package of the ServiceAccount and RBAC (Role and RoleBinding
`<release>`) the Spark jobs of a namespace run with. The RoleBinding binds
the ServiceAccount `serviceAccountName` alone; the Role grants what Spark on
Kubernetes needs of its driver (pods, services, configmaps and persistent
volume claims of its executors, the pods' logs and the events), and the
SparkApplications (to read; to write only with `submitApplications`), but no
right on Secrets. That is not "no Secrets": a pod
the ServiceAccount creates mounts any Secret of the namespace and runs as any
of its ServiceAccounts, which needs no right on them (the kubelet reads
them). The ConfigMap `<release>-pod-bounds` bounds them: it is the param of
the platform's ValidatingAdmissionPolicy `okdp-pod-bounds`
(sandbox-charts' `bootstrap/protection`, cluster-scoped, no project's), which
refuses a pod of the ServiceAccount that runs as another ServiceAccount or
reads a Secret not in `podSecrets`, and any write of the labelled ConfigMaps
by the ServiceAccount. It is inert on a cluster without that policy, as
jupyterhub's `<release>-spark-pod-bounds`.

The Role and RoleBinding are OKDP's (`okdp.yaml` `objects`), not the upstream
chart's: its RoleBinding binds `spark-role`, which reads the namespace's
Secrets, to the group `system:serviceaccounts:<namespace>`, every
ServiceAccount of the namespace. The chart's `rbac.create` is always false.

A package in the okdp-compiler format: `values.cel.yaml` computes the values
of the upstream chart `oci://quay.io/okdp/charts/spark-rbac` 1.0.1, the
dependency of `Chart.yaml`, from the parameters below
(`config.schema.json`). The console parameters are flat (`rbacCreate`) while
the upstream keys are nested (`rbac.create`): the mapping renames them.

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `rbacCreate` | `true` | Create the Role and its RoleBinding to the ServiceAccount. |
| `serviceAccountCreate` | `true` | Create the ServiceAccount. |
| `serviceAccountName` | `spark` | ServiceAccount name (created or existing). |
| `automountServiceAccountToken` | `true` | Mount the ServiceAccount token. |
| `submitApplications` | `false` | Let the ServiceAccount create, update and delete SparkApplications, for a task that submits them as this ServiceAccount. Off, it reads them only. |
| `podSecrets` | `[]` | The Secrets the pods the Spark jobs create may mount or read (volumes, `secretKeyRef`, `envFrom`): the executors' S3 identities, for example. Names only, no comma. Without the policy, no effect. |

No platform value is read. No provided connection, no UI.

## Upstream values

Any value of the `spark-rbac` chart can be set per instance, under
`upstream.spark-rbac`, merged over the values computed from the parameters
(`upstream` in `okdp.yaml`):

```yaml
upstream:
  spark-rbac:
    serviceAccount:
      annotations: {eks.amazonaws.com/role-arn: arn:aws:iam::111122223333:role/spark}
```

`serviceAccount.name` is refused (`protect`): it is the `serviceAccountName`
parameter, which `okdp.info.yaml` publishes to the Spark jobs. `rbac` is
refused too: the chart's Role and RoleBinding stay off (`rbacCreate` is
OKDP's own Role). The rest of
`serviceAccount` (a workload identity, an IAM role) is the platform's to
give: the compiler denies the key `serviceAccount`, at any depth, to the
upstream values of a project that is not privileged
(`global.okdp.policy.upstream`). An
upstream value wins over the parameter it overlaps
(`serviceAccount.automount` over `automountServiceAccountToken`). No key or
value may contain `{{` (the schema and okdp-compile both refuse it).

## Notes

- The Role and RoleBinding are named after the release (`<release>`, as
  `okdpFullname`), no longer `spark-role` and `spark-rolebinding`; an
  instance upgraded from them loses them, the chart no longer rendering
  them. Two instances of a namespace still need two `serviceAccountName`.
- No other ServiceAccount of the namespace holds the Role. A workload that
  submits SparkApplications (an Airflow task, with the KubernetesExecutor:
  its `executor_config` `pod_override`, `serviceAccountName`) runs as the
  ServiceAccount `serviceAccountName` with `submitApplications: true`, or is
  granted its own rights. A driver needs neither: the default is read only. The
  Spark operator, the Spark History Server and JupyterHub's PySpark
  sessions have rights of their own and need none of this binding.

## Tests

Its cases, `ci/*-values.yaml`, compile by hand as the repository's
[README](../../../README.md#working-on-a-package) shows.

`submit-applications` renders the Role with the write verbs on
SparkApplications; the others have the read verbs only.

`upstream-rbac` is a case that must fail: upstream values that turn the
chart's Role and RoleBinding back on. `pod-secrets` renders the bounds with
two Secrets, `pod-secrets-comma` must fail (a name with a comma would add an
entry to the list); the bounds are in every case with the Role, and absent
from `no-rbac`.
