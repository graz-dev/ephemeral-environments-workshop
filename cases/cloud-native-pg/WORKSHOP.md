## Prepare the workspace

1. Install **Kind** on your local machine (follow this [guide](https://kind.sigs.k8s.io/docs/user/quick-start/#installing-with-a-package-manager)for other install options):

```
brew install kind
```

2. Install **Helm** on your local machine (follow this [guide](https://helm.sh/docs/intro/install/) for other install options):

```
brew install helm
```

3. Install **kubectl** on your local machine (follow this [guide](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/)for other install options):

```
brew install kubectl
```
   
4. Create a cluster:

```
kind create cluster --name ephemeral-environments-demo
```

5. Check the cluster initial setup:

```
kubectl get ns
```

You should get this output:

```
NAME                 STATUS   AGE
default              Active   4m52s
kube-node-lease      Active   4m52s
kube-public          Active   4m52s
kube-system          Active   4m53s
local-path-storage   Active   4m47s
```

6. Add **Crossplane** *helm* repo:

```
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
```

7. Install **Crossplane**:

```
helm install crossplane --namespace crossplane-system --create-namespace crossplane-stable/crossplane
```

8. Check that *crossplane* is installed correctly:

```
kubectl get pods -n crossplane-system
```

You should get this output:

```
NAME                                       READY   STATUS    RESTARTS   AGE
crossplane-67b976bbf4-hn9kk                1/1     Running   0          81s
crossplane-rbac-manager-594757659d-dhr97   1/1     Running   0          81s
```

9. Install the *cert-manager*:

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml
```

10. Install *kube-green*:

```
kubectl apply -f https://github.com/kube-green/kube-green/releases/latest/download/kube-green.yaml
```

11. Check kube-green installation:

```
kubectl get pods -n kube-green
```

You should get this output:

```
NAME                                             READY   STATUS    RESTARTS   AGE
kube-green-controller-manager-6c677846bb-vxs64   1/1     Running   0          85s
```

## CASE 1: Handle infrastructure operators

1. Install the *Crossplane Kubernetes provider* creating the file `provider-kubernetes.yaml` with the following content:

```yaml
# provider-kubernetes.yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-kubernetes
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.18.0
```

2. Apply the provider manifest:

```
kubectl apply -f provider-kubernetes.yaml
```

You should get this output:

```
provider.pkg.crossplane.io/provider-kubernetes created
```

3. Create the `provider-kubernetes-config.yaml` file to configure the *kubernetes crossplane provider* with the following content:

```yaml
# provider-kubernetes-config.yaml
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: kubernetes-in-cluster
spec:
  credentials:
    source: InjectedIdentity
```

4. Apply the provider config manifest:

```
kubectl apply -f provider-kubernetes-config.yaml
```

You should get this output:

```
providerconfig.kubernetes.crossplane.io/kubernetes-in-cluster created
```

### Handle a PostgresSQL cluster with the CloudNativePG operator

1. Install the **CloudNativePG Operator**:

```
kubectl apply --server-side -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.26/releases/cnpg-1.26.0.yaml
```

2. Check the operator installation:

```
kubectl rollout status deployment -n cnpg-system cnpg-controller-manager
```

You should get this output:

```
deployment "cnpg-controller-manager" successfully rolled out
```

3. Create the **CompositeResourceDefinition (XRD)**
   Define a new API `XPostgresCluster` with an `instances` field. Create the `xrd-postgrescluster.yaml` file with the following content:

```yaml
# xrd-postgrescluster.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xpostgresclusters.demo.crossplane.io
spec:
  group: demo.crossplane.io
  names:
    kind: XPostgresCluster
    plural: xpostgresclusters
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              instances:
                type: integer
                description: "Number of PostgreSQL instances in the cluster."
                default: 1
              hibernation:
                type: string
                description: "Set to 'on' to hibernate the cluster, 'off' to wake it up."
                default: "off"
            required:
            - instances
```

4. Apply the *XRD*:

```
kubectl apply -f xrd-postgrescluster.yaml
```

You should get this output:

```
compositeresourcedefinition.apiextensions.crossplane.io/xpostgresclusters.demo.crossplane.io created
```

5. Create the **Composition**
   This `Composition` map the abstract API to an `Object` resource of the crossplane kubernetes-provider that defines a CloudNativePG `Cluster`. Create the file `composition-postgrescluster.yaml`

```yaml
# composition-postgrescluster.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: postgrescluster.demo.crossplane.io
spec:
  compositeTypeRef:
    apiVersion: demo.crossplane.io/v1alpha1
    kind: XPostgresCluster
  resources:
  - name: postgres-cluster-object
    base:
      apiVersion: kubernetes.crossplane.io/v1alpha2
      kind: Object
      spec:
        managementPolicies: ["*"]
        forProvider:
          manifest:
            apiVersion: postgresql.cnpg.io/v1
            kind: Cluster
            metadata:
              namespace: default
            spec:
              storage:
                size: 1Gi
              bootstrap:
                initdb:
                  database: appdb
                  owner: appuser
        providerConfigRef:
          name: kubernetes-in-cluster
    patches:
    - fromFieldPath: "metadata.name"
      toFieldPath: "spec.forProvider.manifest.metadata.name"
    - fromFieldPath: "spec.instances"
      toFieldPath: "spec.forProvider.manifest.spec.instances"
    - type: FromCompositeFieldPath
      fromFieldPath: spec.hibernation
      toFieldPath: spec.forProvider.manifest.metadata.annotations[cnpg.io/hibernation]
      policy:
        fromFieldPath: Optional
```

6. Apply the *Coposition* manifest:

```
kubectl apply -f composition-postgrescluster.yaml
```

You should get this output:

```
composition.apiextensions.crossplane.io/postgrescluster.demo.crossplane.io created
```

7.  Give the permission to the *kubernetes-provider* service account to act as a *cluster-admin*:

```
SA=$(kubectl -n crossplane-system get sa -o name | grep provider-kubernetes | sed -e 's|serviceaccount\/|crossplane-system:|g')
```

Then run:

```
kubectl create clusterrolebinding provider-kubernetes-admin-binding --clusterrole cluster-admin --serviceaccount="${SA}"
```

You should get:

```
clusterrolebinding.rbac.authorization.k8s.io/provider-kubernetes-admin-binding created
```

8. Create the instance of the database with this file:
   
```yaml
# my-postgres-db.yaml
apiVersion: demo.crossplane.io/v1alpha1
kind: XPostgresCluster
metadata:
  name: my-production-db
  namespace: default
spec:
  instances: 1
  hibernation: "off"
```

8. Apply the database manifest:

```
kubectl apply -f my-postgres-db.yaml
```

You should get the following output:

```
xpostgrescluster.demo.crossplane.io/my-production-db created
```

9. Verify the provisioning:

Check the *Crossplane XR*:

```
kubectl get xpostgrescluster my-production-db
```

You should get: 

```
NAME               SYNCED   READY   COMPOSITION                          AGE
my-production-db   True     False   postgrescluster.demo.crossplane.io   53s
```

Then, check the *CloudNativePG CR*:

```
kubectl get cluster -n default my-production-db
```

You should get:

```
NAME               AGE   INSTANCES   READY   STATUS                     PRIMARY
my-production-db   43s   1           1       Cluster in healthy state   my-production-db-1
```

Then, check the *database pods*:

```
kubectl get pods -n default -l cnpg.io/cluster=my-production-db
```

You should get this output:

```
NAME                 READY   STATUS    RESTARTS   AGE
my-production-db-1   1/1     Running   0          65s
```

10. Give the permission to kube-green to act on the `XPostgresCluster` resource. Create the file `kube-green-rbac-for-postgres.yaml`:

```yaml
# kube-green-rbac-for-postgres.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-green-xpostgrescluster-patcher
rules:
- apiGroups:
  - "demo.crossplane.io"
  resources:
  - "xpostgresclusters"
  verbs:
  - "get"
  - "list"
  - "watch"
  - "patch"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-green-patch-xpostgrescluster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-green-xpostgrescluster-patcher
subjects:
- kind: ServiceAccount
  name: kube-green-controller-manager
  namespace: kube-green
```

11. Apply the manifest:

```
kubectl apply -f kube-green-rbac-for-postgres.yaml
```

You should get:

```
clusterrole.rbac.authorization.k8s.io/kube-green-xpostgrescluster-patcher created
clusterrolebinding.rbac.authorization.k8s.io/kube-green-patch-xpostgrescluster created
```

12. Schedule the *database pods sleeping* with the file `schedule-sleep-postgres.yaml`:

```yaml
# schedule-sleep-postgres.yaml
apiVersion: kube-green.com/v1alpha1
kind: SleepInfo
metadata:
  name: sleep-schedule-for-postgres
  namespace: default
spec:
  weekdays: "*"
  sleepAt: "18:47"
  wakeUpAt: "18:49"
  patches:
  - target:
      group: demo.crossplane.io
      kind: XPostgresCluster
    patch: |
      - op: replace
        path: /spec/hibernation
        value: "on"
```

This `SleepInfo` resource will stop your db pods at every 5 minutes and restart at every 7 minutes of each hour in the day.

11. Apply the `SleepInfo`:

```
kubectl apply -f schedule-sleep-postgres.yaml
```

You should get:

```
sleepinfo.kube-green.com/sleep-schedule-for-postgres created
```

12. Check the result:

Check the *XRD* status:

```
watch "kubectl get xpostgrescluster my-production-db -n default -o yaml"
```

You should see the XRD initially with the property `hibernation` at `off`. When kube-green run it will modify this property to `on`

Then, check the `Cluster` Obecjet:

```
watch "kubectl get cluster my-production-db -n default -o yaml"
```

You should see the changes done by kube-green reflected here.

Last check if the database pod is up, it shouldn't.

```
watch kubectl get pods -n default -l cnpg.io/cluster=my-production-db
```
