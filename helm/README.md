
# TrustyAI Operator Helm Installation

To install the TrustyAI Operator using helm:

`helm install --create-namespace --namespace trustyai-service-operator trustyai-service-operator helm/trustyai-service-operator`

To install the TrustyAI Service into a datascience project:

`helm install --namespace my-datascience-project trustyai-service helm/trustyai-service`



