# Repository for GitOps demos

## Ensure argoCD application controller has the needed rbac configuration

Apply rbac configuration, adapt the file as needed:

```bash
oc apply -f rbac_configuration/rbac.yaml
```

Ensure that the ArgoCD object have `default_policy: role:admin`, or configure the specific group policies as needed

And also ensure that, if some objects need to be skipped from the sync, to also include the next annotation in the ArgoCD object:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: openshift-gitops
  namespace: openshift-gitops
  annotations:
    argocd.argoproj.io/compare-options: IgnoreExtraneous
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource
spec:
  ...
```

### Resource tracking
When creating InferenceServices, there is a problem with couple of routes created on the istio-system namespace that do not have owner reference yet having the same label. For that reason we need to configure ArgoCD so that resource tracking method only considers annotations (instead of annotations+labels):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: openshift-gitops
  namespace: openshift-gitops
  annotations:
    argocd.argoproj.io/compare-options: IgnoreExtraneous
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource
spec:
  resourceTrackingMethod: annotation
  ...
```

## Apply a configuration to deploy a vLLM + promptflow App

To apply a configuration to deploy an application (vllm + promptflow using it in
this case):

```bash
oc apply -f gitops/vllm
oc apply -f gitops/promptflow
```

### Notes
Secrets are not included so you need to create them manually. 

#### For promptflow app

Define the secret:

```bash
export YOUR_KEY_VALUE=XXXX
export APP_NAMESPACE=promptflow-ns
```

And then define the secret:

```bash
cat << EOF >> secret.yaml
kind: Secret
apiVersion: v1
metadata:
  name: connection-keys
  namespace: $APP_NAMESPACE
type: Opaque
data:
  your-key: $(echo -n "$YOUR_KEY_VALUE" | base64)
EOF
```

And create it:
```bash
oc apply -f secret.yaml
```

#### For vllm model serving

Define the required env vars with proper values:

```bash
export AWS_ACCESS_KEY_ID="minio"
export AWS_DEFAULT_REGION="no-region"
export AWS_S3_BUCKET="models"
export AWS_S3_ENDPOINT="https://my_s3_endpoint_url"
export AWS_SECRET_ACCESS_KEY="my_secret_password"
```

And create the secret yaml for the dataconneciton:

```bash
cat << EOF >> dataconnection.yaml
kind: Secret
apiVersion: v1
metadata:
  annotations:
    opendatahub.io/connection-type: s3
    openshift.io/display-name: models
  labels:
    opendatahub.io/dashboard: "true"
    opendatahub.io/managed: "true"
  name: aws-connection-models
  namespace: vllm-ns
data:
  AWS_ACCESS_KEY_ID: $(echo -n "$AWS_ACCESS_KEY_ID" | base64)
  AWS_DEFAULT_REGION: $(echo -n "$AWS_DEFAULT_REGION" | base64)
  AWS_S3_BUCKET: $(echo -n "$AWS_S3_BUCKET" | base64)
  AWS_S3_ENDPOINT: $(echo -n "$AWS_S3_ENDPOINT" | base64)
  AWS_SECRET_ACCESS_KEY: $(echo -n "$AWS_SECRET_ACCESS_KEY" | base64)

type: Opaque
EOF
```

And apply it:
```bash
oc apply -f dataconnection.yaml
```

## Apply a configuration to deploy a tetkon CI pipeline: 

```bash
oc apply -f gitops/tekton_ci
```

Then, to execute it you can create and apply something like the next (either on the CLI or via console UI):

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: promptflow-pipeline-run
  namespace: promptflow-pipeline-ns
spec:
  pipelineRef:
    name: promptflow-pipeline
  taskRunSpecs:
    - pipelineTaskName: build-image
      serviceAccountName: tekton-sa
  params:
    - name: repo-url
      value: https://github.com/your-repo/your-project.git
    - name: branch-name
      value: your-branch
    - name: image-url
      value: your-registry/your-image:latest
    - name: username
      value: your-username
    - name: password
      value: your-password
    - name: registry
      value: your-registry
    - name: flow-name
      value: your-flow
    - name: api-base
      value: your-vllm-url
    - name: api-key
      value: your-vllm-token
  workspaces:
    - name: shared-workspace
      volumeClaimTemplate:
        spec:
          accessModes: [ "ReadWriteOnce" ]
          resources:
            requests:
              storage: 1Gi
```
