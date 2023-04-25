# Git repositories

The One Touch Provisioning pattern is consisted of 6 repositories: 1 bootstrap repository `otp-gitops`, containing the ArgoCD Applications, and 5 template repositories: `otp-gitops-infra`, `otp-gitops-services`, `otp-gitops-clusters`, `otp-gitops-apps` and `otp-gitops-policies`. These tepmlate repositories source code are pulled into `otp-gitops` bootstrap repository.

Rather than relying on a single mono repository, we use multiple repositories to reflect the ownership and contributions of various personas. Refer to [Motivation](/#motivation) for details.

## Code organisation

As the source repositories are constant across multiple ArgoCD applications, OTP uses Kustomize to patch each ArgoCD applications with the approriate repository URL.

As an example, the `infra` layer is responsible for setting up an organisation's infrastructure components: machinesets, infraconfig, storage, etc. In `otp-gitops`, the `infra` layer is contained in an overarching `infra` ArgoCD Application, and an `infra` ArgoCD AppProject. The `infra` application helps provide a sync context to deploy infrastructure applications (refer to [ArgoCD concepts](/guides/technical-explanations/argocd-concepts) for more details), while the `infra` AppProject restricts what resources, where the source templates come from, and which clusters and namespaces the resources are allowed to be deployed into (refer to [ArgoCD concepts](/guides/technical-explanations/argocd-concepts) for more details).

In `otp-gitops`, [`0-bootstrap/hub/1-infra/1-infra.yaml`](https://github.com/one-touch-provisioning/otp-gitops/blob/146ddce502/0-bootstrap/hub/1-infra/1-infra.yaml) contains the source code to the `infra` ArgoCD `app` and the associated `AppProject`. Note that each layer can be selected with a label. In the `infra` layer example, the `infra` AppProject and Application has labels `gitops.tier.layer=infra`.


```yaml title="0-bootstrap/hub/1-infra/1-infra.yaml"
---
apiVersion: argoproj.io/v1alpha1
// highlight-next-line
kind: AppProject
metadata:
  name: infra
  // highlight-start
  labels:
    gitops.tier.layer: infra
  // highlight-end
spec:
  sourceRepos: [] # Populated by kustomize patches in 1-infra/kustomization.yaml
  destinations:
  - namespace: namespace
    server: k8s-server
  clusterResourceWhitelist:
  - group: allowed-apigroup
    kind: allowed-kind
  roles:
  # A role which provides read-only access to all applications in the project
  - name: read-only
    description: Read-only privileges to my-project
    policies:
    - p, proj:my-project:read-only, applications, get, my-project/*, allow
    groups:
    - argocd-admins
---
apiVersion: argoproj.io/v1alpha1
// highlight-next-line
kind: Application
metadata:
  name: infra
  annotations:
    argocd.argoproj.io/sync-wave: "100"
  // highlight-start
  labels:
    gitops.tier.layer: gitops
  // highlight-end
spec:
  destination:
    namespace: openshift-gitops
    server: https://kubernetes.default.svc
  project: infra
  source: # repoURL  and targetRevision populated by kustomize patches in 1-infra/kustomization.yaml
    path: 0-bootstrap/hub/1-infra
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

The `repoURL` and `targetRevision`, used to determine where `infra` apps can pull its templates from, is provided in [`0-bootstrap/hub/1-infra/kustomization.yaml`](https://github.com/one-touch-provisioning/otp-gitops/blob/146ddce502/0-bootstrap/hub/1-infra/kustomization.yaml#L76).

```yaml title="0-bootstrap/hub/1-infra/kustomization.yaml"
resources:
 # Infrastructure applications to be patched

patches:
- target:
    group: argoproj.io
    kind: Application
    // highlight-start
    labelSelector: "gitops.tier.layer=infra"
    // highlight-end
  patch: |-
    - op: add
    // highlight-start
      path: /spec/source/repoURL
    // highlight-end
      value: ${GIT_BASEURL}/${GIT_ORG}/${GIT_GITOPS_INFRA}
    - op: add
    // highlight-start
      path: /spec/source/targetRevision
    // highlight-end
      value: ${GIT_GITOPS_BRANCH}
```

The allowed source repositories for the `infra` AppProject is provided in [`0-bootstrap/hub/kustomization.yaml`](https://github.com/one-touch-provisioning/otp-gitops/blob/146ddce502/0-bootstrap/hub/kustomization.yaml#L26).


```yaml title="0-bootstrap/hub/kustomization.yaml"
resources:
 - 1-infra/1-infra.yaml
 # Other YAML files containing the ArgoCD AppProjects and Applications
patches:
  # Other patch targets
  - target:
      group: argoproj.io
      kind: AppProject
      labelSelector: "gitops.tier.layer=infra"
    patch: |-
      - op: add
        path: /spec/sourceRepos/-
        value: ${GIT_BASEURL}/${GIT_ORG}/${GIT_GITOPS}
      - op: add
        path: /spec/sourceRepos/-
        value: ${GIT_BASEURL}/${GIT_ORG}/${GIT_GITOPS_INFRA}
```

Similar patterns exist for other layers in the pattern.

As the use cases evolve, OTP will choose to adopt and innovate on the code organisation and deployment methodologies. Check back on this page for future updates.