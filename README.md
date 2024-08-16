
# TrustyAI Model Drift Demo

## TrustyAI Operator Installation

To install the TrustyAI Operator using helm:

`helm install --create-namespace --namespace trustyai-service-operator trustyai-service-operator helm/trustyai-service-operator`

To install the TrustyAI Service into a datascience project:

`helm install --namespace my-datascience-project trustyai-service helm/trustyai-service`

The TrustyAI Service is authenticated by bearer token to grab a token:

`export TOKEN=$(oc whoami -t)`

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
* Endpoint: `http://minio.my-datascience-project.svc.cluster.local:9000`
* Region: `any`
* Bucket: `my-datascience-project`

The demo model uses an XGBoost model, which the default serving runtimes in RHOAI do not support. We therefore needs to create a new serving runtime that uses Seldon:

`helm install --namespace my-datascience-project mlserver-runtime helm/mlserver-runtime`


Finally, we need to manually upload our model into our Minio storage, to make it available to the runtime. Upload the `data/gaussian_credit_model.json` file to the bucket you defined in Minio earlier.

## Run the Demo

We are finally ready to run our demo!

### Deploy the Credit Scoring Model

First, deploy the model into the runtime, on the "Models" pane of your datascience project, select "Deploy Model" next to the mlserver-1.x server and fill in the following:

Model name: `gaussian-credit-model`
Model framework: `XGBoost`
Existing data connection name: `minio`
Path: `gaussian_credit_model.json`

The model will take a while to start up so be patient.

Verify that the model has registered with TrustyAI by selecting one of the modelmesh serving pods. In the `Environment` pane you should see the field `MM_PAYLOAD_PROCESSORS` is set.

### Upload Reference Data

First, we are going to upload our reference data to TrustyAI - this is a labelled dataset that we will measure inference data and predictions against to determine if we have data or model drift.

First, determine the route to our TrustyAI service:

`TRUSTY_ROUTE=https://$(oc -n my-datascience-project get route/trustyai-service --template={{.spec.host}})`

Next, we'll upload our reference data to TrustyAI:

```
curl -sk -H "Authorization: Bearer ${TOKEN}" $TRUSTY_ROUTE/data/upload  \
  --header 'Content-Type: application/json' \
  -d @data/training_data.json
```

You should see a message that 1000 datapoints have been uploaded to the TrustyAI service.

Let's query the reference dataset via the `/info` endpoint:

`curl -sk -H "Authorization: Bearer ${TOKEN}" --header 'Content-Type: application/json' $TRUSTY_ROUTE/info | jq`

We can see here that the input and output parameters are not descriptively labelled, which will make metrics hard to interpret. We can relabel it using a name mapping:

```
curl -sk -H "Authorization: Bearer ${TOKEN}" -X POST --location $TRUSTY_ROUTE/info/names \
  -H "Content-Type: application/json"   \
  -d "{
    \"modelId\": \"gaussian-credit-model\",
    \"inputMapping\":
      {
        \"credit_inputs-0\": \"Age\",
        \"credit_inputs-1\": \"Credit Score\",
        \"credit_inputs-2\": \"Years of Education\",
        \"credit_inputs-3\": \"Years of Employment\"
      },
    \"outputMapping\": {
      \"predict-0\": \"Acceptance Probability\"
    }
  }"
```

Now if we re-run the query of `/info` we will see that the input and output parameters have useful names.

### Schedule a Metric

By default, the TrustyAI service does not generate any metrics - we have to tell it what metrics to generate. Here, we are going to tell it to generate the meanshift metric, with respect to the dataset tagged TRAINING that we uploaded to the service (you can look at the reference data in `data/training_data.json` to see how this tag is set):

```
curl -k -H "Authorization: Bearer ${TOKEN}" -X POST --location $TRUSTY_ROUTE/metrics/drift/meanshift/request -H "Content-Type: application/json" \
  -d "{
        \"modelId\": \"gaussian-credit-model\",
        \"referenceTag\": \"TRAINING\"
      }"
```

We are now ready to run inference with our model and observe drift!

### Observing Drift

Let's start doing some inference!

First, get the route to the model inference endpoint:

`MODEL_ROUTE=https://$(oc -n my-datascience-project get route/gaussian-credit-model --template={{.spec.host}})`

Now start running inference:

```
for batch in {0..595..5}; do
  curl -k -H "Authorization: Bearer ${TOKEN}" $MODEL_ROUTE/v2/models/gaussian-credit-model/infer -d @data/data_batches/$batch.json
  sleep 1
done
```

You can now pick up the original demo instructions from here: https://github.com/trustyai-explainability/odh-trustyai-demos/blob/main/3-DataDrift/README.md#check-the-metrics

















