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

## Handle AWS S3 buckets

1. Install the *AWS S3 Provider* creating the file `provider-aws-s3.yaml` with the following content:

```yaml
# provider-aws-s3.yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v1.23.1
```

2. Apply the provider manifest:

```
kubectl apply -f provider-aws-s3.yaml
```

You should get this output:

```
provider.pkg.crossplane.io/provider-aws-s3 created
```

3. Create a *secret* with your AWS credentials. You should have a file `aws-cretentials.txt` with your credentials, like:

```txt
aws-credentials.txt

[default]
aws_access_key_id = <your-access-key-id>
aws_secret_access_key = <your-secret-access-key>
```

Then run this command:

```
kubectl create secret generic aws-secret -n crossplane-system --from-file=creds=./aws-credentials.txt
```

4. Create the `provider-aws-config.yaml` file to configure the *AWS S3 Provider* with the following content:

```yaml
# provider-aws-config.yaml
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-secret
      key: creds
```

5. Apply the provider config manifest:

```
kubectl apply -f provider-aws-config.yaml
```

You should get this output:

```
providerconfig.aws.upbound.io/default created
```

6. We're going to create a *Crossplane XRD* to handle three resources available with the *AWS S3 Provider*: **Bucket**, **BucketPolicy** and **BucketPublicAccessBlock**. We first need to create our **CompositeResourceDefinition (XRD)**
Define a new API `XSecureBucket` with the following `parameters`: `bucketName`, `region`, `policy` and `blockPublicPolicy`. Create the `xrd-securebucket.yaml` file with the following content:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xsecurebuckets.platform.example.org
spec:
  group: platform.example.org
  names:
    kind: XSecureBucket
    plural: xsecurebuckets
  claimNames:
    kind: SecureBucket
    plural: securebuckets
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
              parameters:
                type: object
                properties:
                  bucketName:
                    description: "Il nome univoco del bucket S3."
                    type: string
                  region:
                    description: "La regione AWS dove creare il bucket."
                    type: string
                  policy:
                    description: "La policy JSON completa del bucket in formato stringa."
                    type: string
                  blockPublicPolicy:
                    description: "Se impostato a true, blocca le policy pubbliche."
                    type: boolean
                    default: true
                required:
                - bucketName
                - region
                - policy
```

7. Apply the *XRD*:

```
kubectl apply -f xrd-securebucket.yaml
```

You should get:

```
compositeresourcedefinition.apiextensions.crossplane.io/xsecurebuckets.platform.example.org created
```

8. Create the **Composition**
   This `Composition` map the abstract API to an `Bucket`, `BucketPolicy` and `BucketPublicAccessBlock` resource of the crossplane *AWS S3 Provider*. Create the file `composition-securebucket.yaml`

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: s3.securebucket.platform.example.org
  labels:
    type: secure-s3-from-manifests
spec:
  compositeTypeRef:
    apiVersion: platform.example.org/v1alpha1
    kind: XSecureBucket
  resources:
    - name: bucket
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider: {}
          providerConfigRef:
            name: default
      patches:
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "metadata.name"
        - fromFieldPath: "spec.parameters.region"
          toFieldPath: "spec.forProvider.region"
    - name: bucket-policy
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: BucketPolicy
        spec:
          forProvider: {}
      patches:
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "metadata.name"
          transforms:
            - type: string
              string:
                fmt: "%s-policy"
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "spec.forProvider.bucket"
        - fromFieldPath: "spec.parameters.region"
          toFieldPath: "spec.forProvider.region"
        - fromFieldPath: "spec.parameters.policy"
          toFieldPath: "spec.forProvider.policy"
    - name: bucket-public-access-block
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: BucketPublicAccessBlock
        spec:
          forProvider:
            blockPublicAcls: false
            restrictPublicBuckets: false
      patches:
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "metadata.name"
          transforms:
            - type: string
              string:
                fmt: "%s-public-access-block"
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "spec.forProvider.bucket"
        - fromFieldPath: "spec.parameters.region"
          toFieldPath: "spec.forProvider.region"
        - fromFieldPath: "spec.parameters.blockPublicPolicy"
          toFieldPath: "spec.forProvider.blockPublicPolicy"
```

9. Apply the **Composition**:

```
kubectl apply -f composition-securebucket.yaml
```

You should get:

```
composition.apiextensions.crossplane.io/s3.securebucket.platform.example.org created
```

10. Create the **Claim** to provision your bucket on AWS S3.
Create a file `my-new-bucket-claim.yaml` with this content:

```yaml
apiVersion: platform.example.org/v1alpha1
kind: SecureBucket
metadata:
  name: my-final-bucket-for-production
spec:
  compositionSelector:
    matchLabels:
      type: secure-s3-from-manifests
  parameters:
    bucketName: "my-final-bucket-for-production-2025"
    region: "us-east-2"
    blockPublicPolicy: false
    policy: |
      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Sid": "AllowAdminAccess",
            "Effect": "Allow",
            "Principal": {
              "AWS": "arn:aws:iam::<your-aws-account-id>:user/<your-aws-username>"
            },
            "Action": "s3:*",
            "Resource": [
              "arn:aws:s3:::my-final-bucket-for-production-2025",
              "arn:aws:s3:::my-final-bucket-for-production-2025/*"
            ]
          }
        ]
      }
```

11. Apply the *Claim* and check the S3 bucket is correctly provisioned.
This claim will provision a publicly available S3 bucket. With kube-green, we will schedule a patch to change the policy and prevent the bucket from being accessed at certain times, except for the user used by Crossplane.

```
kubectl apply -f my-new-bucket-claim.yaml
```

You should get:

```
securebucket.platform.example.org/my-final-bucket-for-production created
```

12. Check the provisioning of your bucket:

```
kubectl get securebuckets
```

You should get:

```
NAME                             SYNCED   READY   CONNECTION-SECRET   AGE
my-final-bucket-for-production   True     True                        38s
```

And on the AWS console you should see your bucket available:

![](/cases/aws-s3/imgs/s3-1.png)

13. 
