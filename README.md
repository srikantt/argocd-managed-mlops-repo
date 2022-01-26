# ArgoCD & KubeFlow Repo

This is an MLOps repository based on Argo CD & KubeFlow

# Structure

Below is an explanation on how this repo is laid out. You'll notice
that I use [Kustomize](https://kustomize.io/) heavily. I do this since I
follow the [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)
principal when it comes to YAML files.

```shell
cluster-mlops/ # 1
├── bootstrap # 2
│   ├── base
│   │   ├── argocd-ns.yaml
│   │   └── kustomization.yaml
│   └── overlays
│       └── default
│           └── kustomization.yaml
├── components # 3
│   ├── applicationsets
│   │   ├── core-components-appset.yaml
│   │   ├── kustomization.yaml
│   │   └── tenants-appset.yaml
│   └── argocdproj
│       ├── kustomization.yaml
│       └── test-project.yaml
├── core # 4
│   ├── gitops-controller
│   │   └── kustomization.yaml
│   └── kubeflow
│       ├── kustomization.yaml
│       └── manifests-master
└── tenants # 5
    ├── 
    └── myapp
        ├── kustomization.yaml
        ├── myapp-deployment.yaml
        ├── myapp-ns.yaml
        └── myapp-service.yaml
```

|#|Directory Name|Description|
|---|----------------|-----------------|
| 1. |`cluster-mlops`| This is the cluster name. This name should be unique to the specific cluster you're targeting. If you're using CAPI, this should be the name of your cluster, the output of `kubectl get cluster`|
| 2. | `bootstrap` | This is where bootstrapping specifc configurations are stored. These are items that get the cluster/automation started. They are usually install manifests.<br /><br />`base` is where are the "common" YAML would live and `overlays` are configurations specific to the cluster.<br /><br />The `kustomization.yaml` file in `default` has `cluster-mlops/components/applicationsets/` and `cluster-mlops/components/argocdproj/` as a part of it's `bases` config.|
| 3. | `components` | This is where specific components for the GitOps Controller lives (in this case Argo CD).<br /><br />`applicationsets` is where all the ApplicationSets YAMLs live and `argocdproj` is where the ArgoAppProject YAMLs live.<br /><br />Other things that can live here are RBAC, Git repo, and other Argo CD specific configurations (each in their repsective directories).|
| 4. | `core` | This is where YAML for the core functionality of the cluster live. Here is where the Kubernetes administrator will put things that is necissary for the functionality of the cluster (like cluster configs or cluster workloads).<br /><br />Under `gitops-controller` is where you are using Argo CD to manage itself. The `kustomization.yaml` file uses `cluster-mlops/bootstrap/overlays/default` in it's `bases` configuration. This `core` directory gets deployed as an applicationset which can be found under `cluster-mlops/components/applicationsets/core-components-appset.yaml`.<br /><br />To add a new "core functionality" workoad, one needs to add a directory with some yaml in the `core` directory. 
| 5. | `tenants` | This is where the workloads for this cluster live.<br /><br />Similar to `core`, the `tenants` directory gets loaded as part of an ApplicationSet that is under `cluster-XXXX/components/applicationsets/tenants-appset.yaml`.<br /><br />This is where Devlopers/Release Engineers do the work. They just need to commit a directory with some YAML and the applicationset takes care of creating the workload.<br /><br />

# Testing

To see this in action, first get yourself a kubernetes cluster with 32 cpus and 64gb ram & 200 gb disk space

Then, just apply this repo.

```shell
until kubectl apply -k https://github.com/srikantt/argocd-managed-mlops-repo/cluster-mlops/bootstrap/overlays/default; do sleep 10; done
```

This should give you 4 applications

```shell
$ kubectl get applications -n argocd
NAME                          SYNC STATUS   HEALTH STATUS
kubeflow                      Synced        Healthy
gitops-controller             Synced        Healthy
```

Backed by 2 applicationsets

```shell
$ kubectl get appsets -n argocd
NAME      AGE
cluster   110s
tenants   110s
```

To see the Argo CD UI, you'll first need the password

```shell
kubectl get secret/argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d ; echo
```

Then port-forward to see it in your browser (using `admin` as the username).

```shell
kubectl -n argocd port-forward service/argocd-server 8080:443
```
