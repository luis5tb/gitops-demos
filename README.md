# Repository for GitOps demos

## Apply a configuration

To apply a configuration to deploy an application (vllm + promptflow using it in
this case):

```bash
oc apply -f gitops/vllm
oc apply -f gitops/promptflow
```

## Notes
Secrets are not included so you need to create them manually. 

### For promptflow app

Define the secret:

```bash
export YOUR_KEY_VALUE=XXXX
```

And then define the secret:

```bash
cat << EOF >> secret.yaml
kind: Secret
apiVersion: v1
metadata:
  name: connection-keys
  namespace: rag-chatbot
type: Opaque
  data:
    YOUR_KEY: $(echo -n "$YOUR_KEY_VALUE" | base64)
EOF
```

And create it:
```bash
oc apply -f secret.yaml
```

### For vllm model serving

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
```

And apply it:
```bash
oc apply -f dataconnection.yaml
```