
# TrustyAI Model Drift Demo

## TrustyAI Operator Installation

To install the TrustyAI Operator using helm:

`helm install --create-namespace --namespace trustyai-service-operator trustyai-service-operator helm/trustyai-service-operator`

To install the TrustyAI Service into a datascience project:

`helm install --namespace my-datascience-project trustyai-service helm/trustyai-service`

## Demo Installation

### Install Minio

We will use Minio as an object store to store models and data.

First, install Minio:

`helm install --namespace my-datascience-project minio oci://registry-1.docker.io/bitnamicharts/minio`

Follow the instructions in the helm chart output to set the ROOT_USER and ROOT_PASSWORD environment variables on your local system:
```
export ROOT_USER=$(kubectl get secret --namespace my-datascience-project minio -o jsonpath="{.data.root-user}" | base64 -d)
export ROOT_PASSWORD=$(kubectl get secret --namespace my-datascience-project minio -o jsonpath="{.data.root-password}" | base64 -d)
```

Finally, make the Minio webUI available using the following command:

`oc port-forward --namespace my-datascience-project svc/minio 9001:9001 &`

Verify that you can login to the Minio UI at http://127.0.0.1:9001/minio

Create a bucket called 'my-datascience-project'

### Setup the Demo Environment

In your datascience project, add your newly created Minio bucket as a Data Connection as follows:
* Name: `minio`
* Access key: contents of `$ROOT_USER`
* Secret key: contents of `$ROOT_PASSWORD`
* Endpoint: `minio.my-datascience-project.svc.cluster.local:9000`
* Region: `any`
* Bucket: `my-datascience-project`

Next, 










