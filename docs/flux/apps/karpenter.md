# Karpenter

This infrastructure component will deploy Karpenter into Kubernetes clusters. The structure of this component is described below:

```bash
karpenter
├── README.md
├── charts
│   └── karpenter
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

1. `HelmRepository` - directs Flux to the Karpenter helm repo
2. `Namespace` - creates the namespace for the helm chart to be deployed in

## Components

Karpenter consists of one helm chart. The `apps/karpenter/charts` directory contains the resources required for deploying different versions of the Karpenter helm chart. Each version directory contains a `values.yaml.tpl` which consists of a `ConfigMap` for storing the helm values.

```yaml
# apps/karpenter/charts/karpenter/<chart-version>/values.yaml.tpl
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: karpenter-config
type: Opaque
data:
  values.yaml: |-
    ...
```

It also contains a `kustomization.yaml` which creates a `Component` that deploys the `ConfigMap` in `values.yaml.tpl` and `HelmRelease` in the `sync.yaml.tpl` files. The `HelmRelease` chart version is patched in the component in order to avoid deploying the wrong chart version with the generated helm values.

```yaml
# apps/karpenter/charts/karpenter/<chart-version>/kustomization.yaml
---
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
metadata:
  name: karpenter-config
resources: [values.yaml.tpl, ../sync.yaml.tpl]
generatorOptions:
  disableNameSuffixHash: true
patches:
  - target:
      kind: HelmRelease
      name: karpenter
    patch: |-
      apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      metadata:
        name: not-used
      spec:
        chart:
          spec:
            version: 1.2.1
```

## Deploying Karpenter

To deploy Karpenter, create a Flux `Kustomization` in your GitOps repository and use the `karpenter` component with the desired version path.

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: karpenter
  namespace: flux-system
spec:
  targetNamespace: karpenter
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: yggdrasil-catalog
  serviceAccountName: kustomize-controller
  path: apps/karpenter
  prune: true
  wait: true
  timeout: 5m
  components: [./charts/karpenter/1.2.1]
  postBuild:
    substitute:
      targetNamespace: karpenter
      tenant: <tenant> # i.e. titanbytedata
      ... # omitted for brevity
```

## Substitution Variables

### base

| Variable          | Required | Default         | Description                                     |
| ----------------- | -------- | --------------- | ----------------------------------------------- |
| `targetNamespace` |          | `karpenter`     | the namespace to create and deploy Karpenter in |
| `tenant`          |          | `titanbytedata` | the tenant that this service belongs to         |

### memgraph

| Variable                | Required | Default | Description                                                                      |
| ----------------------- | -------- | ------- | -------------------------------------------------------------------------------- |
| `coreNodeKey`           |          | `core`  | the taint key for core nodes so Karpenter doesn't get deployed to it's own nodes |
| `serviceAccountRoleArn` |          |         | the AWS role ARN for the Karpenter pods                                          |
| `clusterName`           |          |         | the EKS cluster name                                                             |
| `clusterEndpoint`       |          |         | the EKS cluster endpoint                                                         |
| `interruptionQueue`     |          |         | the interruption queue                                                           |
