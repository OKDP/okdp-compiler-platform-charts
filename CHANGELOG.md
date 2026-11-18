# Changelog

## [2.0.0](https://github.com/OKDP/platform-charts/compare/v1.0.0...v2.0.0) (2026-11-18)


### ⚠ BREAKING CHANGES

* **okdp-control-plane-ui:** 0.9.1-1.0.0, the console's pod spec by okdpPodSpec
* **okdp-control-plane-server:** 0.10.0-1.0.0, a ClusterRole of namespaces and the SparkApplication CRD only, gitops.topology required
* **okdp-examples:** trino and polaris referenced by the names of their outputs
* **superset:** a metadata database of its own by default, claimed, sign-in by the project's groups
* **jupyterhub:** notebooks without a ServiceAccount token, the PySpark sessions bounded, sign-in by the project's groups
* **airflow:** a metadata database of its own by default, claimed, sign-in by the project's groups
* **spark-history-server:** the event log store a connection or a claim, sign-in by the project's groups
* **spark-rbac:** the Role of the Spark jobs OKDP's, without secrets, their pods bounded by <release>-pod-bounds
* **trino:** catalogs on the namespace's outputs by name, OPA closed by a NetworkPolicy, tokens of the instance's audience alone
* **polaris:** a database of its own by default, claimed, its iceberg-catalog output published by name
* **hive-metastore:** a database of its own by default, claimed, its hive output published by name
* **contracts:** what a claim requests (x-okdp-claim), no naming conventions
* an okdp-compiler catalog: contracts, compile-charts.sh, CI and repository releases, nothing published

### Features

* **airflow:** a metadata database of its own by default, claimed, sign-in by the project's groups ([a825d4c](https://github.com/OKDP/platform-charts/commit/a825d4c231ee4e9516ff22d9df539dc5075ec83d))
* **airflow:** catalog/airflow/3.2.1-1.0.0, the bundled postgresql disabled and protected ([b659e92](https://github.com/OKDP/platform-charts/commit/b659e922de3701eb723cee671546bfdf81f6bc51))
* an okdp-compiler catalog: contracts, compile-charts.sh, CI and repository releases, nothing published ([82ddf8c](https://github.com/OKDP/platform-charts/commit/82ddf8ca4de199f57ac44abc47d7a82030ca4d6e))
* **contracts:** what a claim requests (x-okdp-claim), no naming conventions ([c63b0d5](https://github.com/OKDP/platform-charts/commit/c63b0d574b81f2f203dae68e604902e66cb25030))
* **hive-metastore:** a database of its own by default, claimed, its hive output published by name ([af2c0be](https://github.com/OKDP/platform-charts/commit/af2c0be85a95405c752908081a3e47d2d3cf3acd))
* **hive-metastore:** catalog/hive-metastore/4.0.1-1.0.0 ([dad17ba](https://github.com/OKDP/platform-charts/commit/dad17ba019b340442873c83defbe5381074cf234))
* **jupyterhub:** catalog/jupyterhub/4.4.2-1.0.0, ${suffix} placeholders ([071d642](https://github.com/OKDP/platform-charts/commit/071d6423672ac0cd69d3aae2d08cab8859780e67))
* **jupyterhub:** notebooks without a ServiceAccount token, the PySpark sessions bounded, sign-in by the project's groups ([d8fded8](https://github.com/OKDP/platform-charts/commit/d8fded8f2297bada8d01cb9c3cf7b734ee7ce209))
* **okdp-control-plane-server:** 0.10.0-1.0.0, a ClusterRole of namespaces and the SparkApplication CRD only, gitops.topology required ([0c261b4](https://github.com/OKDP/platform-charts/commit/0c261b4732d0bc044491bfd24524879c879c0ddb))
* **okdp-control-plane-server:** catalog/okdp-control-plane-server/0.9.0-1.0.0, its objects in okdp.yaml, the topology of the deployments repositories ([3204ba5](https://github.com/OKDP/platform-charts/commit/3204ba508d2da4c68fefd92b353d56d5375feab9))
* **okdp-control-plane-ui:** 0.9.1-1.0.0, the console's pod spec by okdpPodSpec ([ec08921](https://github.com/OKDP/platform-charts/commit/ec089212cba1f6d61b81e1cadc02539b11996e99))
* **okdp-control-plane-ui:** catalog/okdp-control-plane-ui/0.9.0-1.0.0, its objects in okdp.yaml ([dd742bb](https://github.com/OKDP/platform-charts/commit/dd742bb47e6f4904f42225526a7e2e3eedbb63c4))
* **okdp-examples:** catalog/okdp-examples/1.3.0-1.0.0 ([520d6fd](https://github.com/OKDP/platform-charts/commit/520d6fd0c9a2cc844a9ca8584355894ba7596d46))
* **okdp-examples:** trino and polaris referenced by the names of their outputs ([bd67aed](https://github.com/OKDP/platform-charts/commit/bd67aedc477449709c13b0775a1c544b7c3c0d55))
* **polaris:** a database of its own by default, claimed, its iceberg-catalog output published by name ([11a9cb7](https://github.com/OKDP/platform-charts/commit/11a9cb72fd2c63f2b9566a33f60a2c99e696b73f))
* **polaris:** catalog/polaris/1.3.0-incubating-1.0.0, the principals phase merges upstream.polaris-admin ([f3a5ec8](https://github.com/OKDP/platform-charts/commit/f3a5ec84a163d920c20bb38cb3740334e1d93d6d))
* **spark-defaults:** catalog/spark-defaults/1.0.0-1.0.0 ([8cecf6a](https://github.com/OKDP/platform-charts/commit/8cecf6a4e1d20c5b56b9f2bf5e37cba0a44e7734))
* **spark-history-server:** catalog/spark-history-server/3.5.1-1.0.0 ([8a2b96b](https://github.com/OKDP/platform-charts/commit/8a2b96baccc0003f9b64f9fa6e4bcf3ae959a539))
* **spark-history-server:** the event log store a connection or a claim, sign-in by the project's groups ([a1474a1](https://github.com/OKDP/platform-charts/commit/a1474a1ed492694f705b78dacfac1ededdfeebdc))
* **spark-operator:** catalog/spark-operator/2.5.2-1.0.0, its CRDs in crds/ ([2a2f4e3](https://github.com/OKDP/platform-charts/commit/2a2f4e38ccfbb8caf3bc93a5ebcef7abeb86a18d))
* **spark-rbac:** catalog/spark-rbac/1.0.1-1.0.0 ([4e24b68](https://github.com/OKDP/platform-charts/commit/4e24b680e435bacf348cc67731164e811ce5be1d))
* **spark-rbac:** the Role of the Spark jobs OKDP's, without secrets, their pods bounded by &lt;release&gt;-pod-bounds ([c11fda0](https://github.com/OKDP/platform-charts/commit/c11fda0d7dabade9e970ef918063bd7ff2a91cd5))
* **superset:** a metadata database of its own by default, claimed, sign-in by the project's groups ([66f07b9](https://github.com/OKDP/platform-charts/commit/66f07b94e763c46ba1f95aec52718239ff246ff3))
* **superset:** catalog/superset/6.0.0-1.0.0, the bundled postgresql and redis disabled and protected ([8f077ae](https://github.com/OKDP/platform-charts/commit/8f077ae332683fbd87e55f94a5f69a47d5344894))
* **trino:** catalog/trino/480.0.0-1.0.0 ([49d810b](https://github.com/OKDP/platform-charts/commit/49d810b3de39d5bbe53223913466870c8b97cd06))
* **trino:** catalogs on the namespace's outputs by name, OPA closed by a NetworkPolicy, tokens of the instance's audience alone ([e5e3b99](https://github.com/OKDP/platform-charts/commit/e5e3b996be72cb02402b3e29d540cc05ee245992))


### Documentation

* claims, platform.yaml and catalogs.yaml, a Migration from KuboCD section in the README ([bae66d5](https://github.com/OKDP/platform-charts/commit/bae66d56969d4e689cce30cc7a25563e4bad9f87))
* okdp.info.yaml, what an instance publishes ([16db638](https://github.com/OKDP/platform-charts/commit/16db6383e9d9d95286d21de2b9f307d769852ff8))
* outputs referenced by name in their namespace ([472636a](https://github.com/OKDP/platform-charts/commit/472636a12dcdbefbff868669b549975181c4780b))
