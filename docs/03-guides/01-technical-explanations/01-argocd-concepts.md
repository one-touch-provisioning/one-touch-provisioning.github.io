# ArgoCD concepts

For a complete introduction to ArgoCD concepts, visit [ArgoCD documentation website](https://argo-cd.readthedocs.io/en/stable/), and [OpenShift GitOps documentation](https://docs.openshift.com/container-platform/4.12/cicd/gitops/gitops-release-notes.html) for product-specific information. This page shows the relevant concepts to the pattern.
## Application sync waves

ArgoCD Application [sync waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/) concept is an annotation that specifies the order of deployment of resources. Sync waves are arbitrary numbers. Lower numbered resources get deployed first. ArgoCD sync waves only work within a single ArgoCD application context. To order the deployments of multiple ArgoCD applications, an overarching Application must be used to form a larger application context. This is the basis for the [App-of-Apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/).

```yaml
apiVersion: argoproj.io/v1alpha1
// highlight-next-line
kind: Application
metadata:
  name: cluster-pool
  // highlight-start
  annotations:
    argocd.argoproj.io/sync-wave: "310"
  // highlight-end
  labels:
    gitops.tier.layer: clusters
spec:
  ...
```

In this pattern, we have an overarching `bootstrap` app, that deploys multiple ArgoCD applications: `infra`, `services`, `policies`, `clusters`, and `apps`.

## Application template source

An ArgoCD Application can specify where its source templates come from by specifying the `spec.source.repoURL` and `spec.source.targetRevision` fields. 

In the following example, an ArgoCD application called `cluster-claim-argocd-app` deploys a Helm template in the `otp-gitops-clusters` [repository](https://github.com/one-touch-provisioning/otp-gitops-clusters). The path `clusterclaims` points to a [folder](https://github.com/one-touch-provisioning/otp-gitops-clusters/tree/master/clusterclaims) within the `otp-gitops-clusters` repository.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cluster-claim-argocd-app
  annotations:
    argocd.argoproj.io/sync-wave: "460"
  labels:
    gitops.tier.layer: clusters
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  ignoreDifferences:
    - group: hive.openshift.io
      kind: ClusterClaim
      jqPathExpressions:
        - .metadata.annotations."cluster.open-cluster-management.io/createmanagedcluster"
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
    syncOptions:
    - RespectIgnoreDifferences=true
  destination:
    namespace: openshift-gitops
    server: https://kubernetes.default.svc
  project: clusters
  // highlight-start
  source:
    path: clusterclaims
    repoURL: 'https://github.com/one-touch-provisioning/otp-gitops-clusters.git'
  // highlight-end
    helm:
      values: |
        clusterClaimName: cluster-claim-name
        clusterPoolName: cluster-pool-name
```

## AppProject

[An ArgoCD AppProject](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/) provides a logical grouping of applications. It restricts what may be deployed, where the apps could be deployed, what kinds of objects may or may not be deployed. By default, an ArgoCD application belongs to a `default` AppProject, which is created automatically and it permits deployment from any source repos, to any clusters, and all resource kinds.

An ArgoCD application is assigned to an AppProject in the `spec.project` field.

In this pattern, there are 5 AppProjects, each map to a corresponding ArgoCD application: [`infra`](https://github.com/one-touch-provisioning/otp-gitops/blob/master/0-bootstrap/hub/1-infra/1-infra.yaml), [`services`](https://github.com/one-touch-provisioning/otp-gitops/blob/master/0-bootstrap/hub/2-services/2-services.yaml), [`policies`](https://github.com/one-touch-provisioning/otp-gitops/blob/master/0-bootstrap/hub/3-policies/3-policies.yaml), [`clusters`](https://github.com/one-touch-provisioning/otp-gitops/blob/master/0-bootstrap/hub/4-clusters/4-clusters.yaml), and [`apps`](https://github.com/one-touch-provisioning/otp-gitops/blob/master/0-bootstrap/hub/5-apps/5-apps.yaml).

When a new resource is added to a layer, its kind, apiGroup and the intended namespace must be added to the the corresponding `spec.clusterResourceWhitelist` field:

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  # Metadata fields
spec:
  # Other spec fields
  // highlight-start
  destinations:
  - namespace: applicable-namespaces
    server: https://kubernetes.default.svc
  // highlight-end
  # Other spec fields
  // highlight-start
  clusterResourceWhitelist:
  - group: apiGroup
    kind: crd-kind
  // highlight-end
  # Other allowed cluster resources
```

## ApplicationSet

[ArgoCD ApplicationSet](https://argocd-applicationset.readthedocs.io/en/stable/) is a separate controller in the ArgoCD project. It is deployed by default with OpenShift GitOps. `ApplicationSet` is similar to Red Had Advanced Cluster Management, in that it allows a single application to be deployed to multiple clusters. `ApplicationSet`, however, also allows you to define deployment targets based on [generators](https://argocd-applicationset.readthedocs.io/en/stable/Generators/) instead of just using cluster selectors like RHACM's [PlacementRule](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.7/html/applications/managing-applications#placement-rule-yaml-structure).

In OTP, `ApplicationSet`s are used to dynamically provision machine pools in different cloud vendors while re-using the existing templates provided in `otp-infra`. In the example below, we're using the [cluster generator](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Cluster/) to select managed clusters on Azure, and dynamically creating the ArgoCD applications using the cluster generator's outputs.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: azure-machinepools
spec:
  // highlight-start
  generators:
  - clusters:
      selector:
        matchLabels:
          apps.open-cluster-management.io/acm-cluster: "true"
          hive.openshift.io/cluster-platform: azure
  // highlight-end
  template:
    metadata:
      name: '{{name}}-storage'
    spec:
      project: clusters
      source:
        # infra repo must be added to the sourceRepos in ArgoCD AppProject
        repoURL: https://github.com/otp-itz/otp-gitops-infra.git
        path: machinepools
        helm:
          values: |
            cloudProvider:
              name: "azure" # aws, azure, ibmcloud or vsphere
              managed: false # for roks, aro, rosa set to true
            cloud:
              clusterName: {{ name }}
      destination:
        server: 'https://kubernetes.default.svc'
        namespace: '{{ name }}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```
