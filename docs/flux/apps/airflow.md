# Airflow Infrastructure

This infrastructure component will deploy Airflow into Kubernetes clusters. The structure of this component is described below:

```bash
airflow
├── README.md
├── charts
│   └── airflow
│       ├── <chart-version>
│       │   ├── kustomization.yaml
│       │   └── values.yaml.tpl
│       └── sync.yaml.tpl
├── kustomization.yaml
├── namespace.yaml.tpl
└── source.yaml.tpl
```

## Base resources

The base `Kustomization` creates the following resources:

1. `HelmRepository` - directs Flux to the Airflow helm repo
2. `Namespace` - creates the namespace for the helm chart to be deployed in

## Components

Airflow consists of one helm chart. The `infrastructure/airflow/charts` directory contains the resources required for deploying different versions of the Airflow helm chart. Each version directory contains a `values.yaml.tpl` which consists of a `Secret` for storing the helm values.

```yaml
# infrastructure/airflow/charts/airflow/<chart-version>/values.yaml.tpl
---
apiVersion: v1
kind: Secret
metadata:
  name: airflow-values
type: Opaque
stringData:
  values.yaml: |-
    ...
```

It also contains a `kustomization.yaml` which creates a `Component` that deploys the `Secret` in `values.yaml.tpl` and `HelmRelease` in the `sync.yaml.tpl` files. The `HelmRelease` chart version is patched in the component in order to avoid deploying the wrong chart version with the generated helm values.

```yaml
# infrastructure/airflow/charts/airflow/<chart-version>/kustomization.yaml
---
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
metadata:
  name: airflow-config
resources: [values.yaml.tpl, ../sync.yaml.tpl]
generatorOptions:
  disableNameSuffixHash: true
patches:
  - target:
      kind: HelmRelease
      name: airflow
    patch: |-
      apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      metadata:
        name: not-used
      spec:
        chart:
          spec:
            version: 1.15.0
```

## Deploying Airflow

Some secret values are required to be substituted into the airflow helm values secret. This Kubernetes secret should be created before the airflow Kustomization. First create a `secrets/airflow` directory in the `cluster/cml/dev/ue2/<cluster-name>` directory. Then create and encrypt secret files for the following secrets:

1. `dbPass` - the password to the airflow backend postgres database
2. `webserverSecretKey` - the secret key for the airflow web server
3. `gitSshKey` - the ssh deploy key for syncing dags to airflow

Now create a `kustomization.yaml` with the following configuration, this will trigger flux to create the secret required for variable substitution in the airflow Kustomization:

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
generatorOptions:
  disableNameSuffixHash: true
secretGenerator:
  - name: airflow-config-secrets
    namespace: flux-system
    files:
      - dbPass=./dbPass
      - webserverSecretKey=./webserverSecretKey
  - name: airflow-ssh-secret
    namespace: airflow
    files:
      - gitSshKey=./gitSshKey
```

To deploy Airflow, create a Flux `Kustomization` in the `acc-platform` repository and use the `airflow` component with the desired version path.

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: airflow
  namespace: flux-system
spec:
  targetNamespace: airflow
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: catalog
  serviceAccountName: kustomize-controller
  path: infrastructure/airflow
  prune: true
  decryption:
    provider: sops
  wait: true
  timeout: 5m
  components:
    - ./charts/airflow/1.15.0
  postBuild:
    substituteFrom:
      - kind: Secret
        name: airflow-config-secrets
    substitute: ...
```

## Okta Authentication

If Okta is required for authentication with Airflow, then the `airflow-okta-auth` component can be added to the airflow Kustomization:

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: airflow
  namespace: flux-system
spec:
  targetNamespace: airflow
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: catalog
  components:
    - ../../components/aws/cml/airflow-okta-auth
  ... # omitted for brevity
```

A secret called `okta-oauth-secret` in the airflow namespace is required for this to work. The following secrets should be included:

1. `oktaClientId` - the okta client id
2. `oktaClientSecret` - the okta client secret

This secret can be added to the list of secret generators described in [Deploying Airflow](#deploying-airflow). Encrypt the `oktaClientId` and `oktaClientSecret` in two files in the secrets directory, then add this value to the `secretGenerator`:

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
generatorOptions:
  disableNameSuffixHash: true
secretGenerator:
  ...
  - name: okta-oauth-secret
    namespace: airflow
    files:
      - ./oktaClientId
      - ./oktaClientSecret
```

## Substitution Variables

### base

| Variable          | Required | Default    | Description                                   |
| ----------------- | -------- | ---------- | --------------------------------------------- |
| `targetNamespace` |          | `airflow`  | the namespace to create and deploy Airflow in |
| `tenant`          |          | `platform` | the tenant that this service belongs to       |

### airflow

| Variable                    | Required              | Default           | Description                                                                                        |
| --------------------------- | --------------------- | ----------------- | -------------------------------------------------------------------------------------------------- |
| `chartVersion`              |                       | `1.15.0`          | the airflow chart version, this is optional and should be patched by the versioned chart component |
| `airflowImageRepo`          |                       | `apache/airflow`  | the airflow image repository                                                                       |
| `airflowImageTag`           |                       | the chart default | the airflow image tag                                                                              |
| `airflowVersion`            |                       | the chart default | the airflow version                                                                                |
| `executor`                  |                       | `CeleryExecutor`  | which airflow executor to use                                                                      |
| `env`                       | ✅                    |                   | the environment, adds `ENVIRONMENT` env var to all pods                                            |
| `dbUser`                    |                       | `airflow`         | the database user                                                                                  |
| `dbPass`                    | ✅                    |                   | the password for the database, set this in the `airflow-config-secrets`                            |
| `dbHost`                    | ✅                    |                   | the host for the database                                                                          |
| `dbName`                    |                       | `airflow_db`      | the database name                                                                                  |
| `webserverSecretKey`        | ✅                    |                   | the webserver secret, set this in the `airflow-config-secrets`                                     |
| `workerRoleArn`             | ✅                    |                   | the iam role for the airflow worker service account                                                |
| `minWorkerReplicas`         |                       | `1`               | the minimum number of workers                                                                      |
| `maxWorkerReplicas`         |                       | `5`               | the maximum number of workers                                                                      |
| `workerLimitCpu`            |                       | `4`               | the worker cpu limit                                                                               |
| `workerLimitMemory`         |                       | `4096Mi`          | the worker memory limit                                                                            |
| `workerRequestCpu`          |                       | `2`               | the worker cpu request                                                                             |
| `workerRequestMemory`       |                       | `2048Mi`          | the worker memory request                                                                          |
| `schedulerReplicas`         |                       | `2`               | the number of scheduler replicas                                                                   |
| `schedulerLimitCpu`         |                       | `4`               | the scheduler cpu limit                                                                            |
| `schedulerLimitMemory`      |                       | `4096Mi`          | the scheduler memory limit                                                                         |
| `schedulerRequestCpu`       |                       | `2`               | the scheduler cpu request                                                                          |
| `schedulerRequestMemory`    |                       | `2048Mi`          | the scheduler memory request                                                                       |
| `webserverReplicas`         |                       | `1`               | the number of webserver replicas                                                                   |
| `webserverLimitCpu`         |                       | `1`               | the webserver cpu limit                                                                            |
| `webserverLimitMemory`      |                       | `4096Mi`          | the webserver memory limit                                                                         |
| `webserverRequestCpu`       |                       | `500m`            | the webserver cpu request                                                                          |
| `webserverRequestMemory`    |                       | `2048Mi`          | the webserver memory request                                                                       |
| `triggererEnabled`          |                       | `true`            | whether to enable the triggerer or not                                                             |
| `triggererReplicas`         |                       | `1`               | the number of triggerer replicas                                                                   |
| `triggererLimitCpu`         |                       | `2`               | the triggerer cpu limit                                                                            |
| `triggererLimitMemory`      |                       | `2048Mi`          | the triggerer memory limit                                                                         |
| `triggererRequestCpu`       |                       | `1`               | the triggerer cpu request                                                                          |
| `triggererRequestMemory`    |                       | `1024Mi`          | the triggerer memory request                                                                       |
| `dagProcessorEnabled`       |                       | `false`           | whether to enable the separate dag processor or not                                                |
| `dagProcessorLimitCpu`      |                       | `1`               | the dag processor cpu limit                                                                        |
| `dagProcessorLimitMemory`   |                       | `1024Mi`          | the dag processor memory limit                                                                     |
| `dagProcessorRequestCpu`    |                       | `500m`            | the dag processor cpu request                                                                      |
| `dagProcessorRequestMemory` |                       | `512Mi`           | the dag processor memory request                                                                   |
| `flowerEnabled`             |                       | `true`            | whether to enable celery flower or not                                                             |
| `flowerLimitCpu`            |                       | `500m`            | the flower cpu limit                                                                               |
| `flowerLimitMemory`         |                       | `512Mi`           | the flower memory limit                                                                            |
| `flowerRequestCpu`          |                       | `250m`            | the flower cpu request                                                                             |
| `flowerRequestMemory`       |                       | `256Mi`           | the flower memory request                                                                          |
| `statsdEnabled`             |                       | `false`           | whether to enable statsd or not                                                                    |
| `statsdLimitCpu`            |                       | `500m`            | the statsd cpu limit                                                                               |
| `statsdLimitMemory`         |                       | `512Mi`           | the statsd memory limit                                                                            |
| `statsdRequestCpu`          |                       | `250m`            | the statsd cpu request                                                                             |
| `statsdRequestMemory`       |                       | `256Mi`           | the statsd memory request                                                                          |
| `pgbouncerLimitCpu`         |                       | `500m`            | the pgbouncer cpu limit                                                                            |
| `pgbouncerLimitMemory`      |                       | `512Mi`           | the pgbouncer memory limit                                                                         |
| `pgbouncerRequestCpu`       |                       | `250m`            | the pgbouncer cpu request                                                                          |
| `pgbouncerRequestMemory`    |                       | `256Mi`           | the pgbouncer memory request                                                                       |
| `redisLimitCpu`             |                       | `500m`            | the redis cpu limit                                                                                |
| `redisLimitMemory`          |                       | `512Mi`           | the redis memory limit                                                                             |
| `redisRequestCpu`           |                       | `250m`            | the redis cpu request                                                                              |
| `redisRequestMemory`        |                       | `256Mi`           | the redis memory request                                                                           |
| `baseDomain`                | ✅                    |                   | the base domain for airflow                                                                        |
| `remoteLoggingEnabled`      |                       | `false`           | whether to enable remote logging or not                                                            |
| `s3LogBucketPath`           |                       | `""`              | the s3 log bucket path                                                                             |
| `dagRepoName`               | ✅                    |                   | the gitlab repository name to sync the dags from                                                   |
| `dagBranch`                 |                       | `main`            | the branch/tag to sync from                                                                        |
| `dagRev`                    |                       | `HEAD`            | the git revision to sync from                                                                      |
| `dagRef`                    |                       | `main`            | the branch/tag to sync from                                                                        |
| `dagPath`                   |                       | `dags`            | the subpath in the repository where dags are located                                               |
| `gitSyncLimitCpu`           |                       | `500m`            | the git sync cpu limit                                                                             |
| `gitSyncLimitMemory`        |                       | `512Mi`           | the git sync memory limit                                                                          |
| `gitSyncRequestCpu`         |                       | `250m`            | the git sync cpu request                                                                           |
| `gitSyncRequestMemory`      |                       | `256Mi`           | the git sync memory request                                                                        |
| `awsACMCertificateArn`      | ✅ if not using istio |                   | the aws acm certificate arn for ssl                                                                |
