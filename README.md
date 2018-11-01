ACA on K8S with Spinnaker
=========================

The instructions below are based on the Google tutorial which uses GKE and Stackdriver[1] and the
instructions, pipelines and other details are modified in order to run the tutorial on a local
K8S cluster with Prometheus.

Simple Deploy Pipeline
----------------------

- Create an application named "sampleapp" on Spinnaker

- Download the description for Simple Deploy pipeline.
wget https://raw.githubusercontent.com/spinnaker/spinnaker/master/solutions/kayenta/pipelines/simple-deploy.json

- Update the simple-deploy.json replace "my-kubernetes-account" with the account you have on local K8S cluster.
Diff pipelines/simple-deploy-gke.json and pipelines/simple-deploy-k8s.json to see what exactly is updated.

- Create the pipeline on Spinnaker. Please ensure you use the right url for gate.

curl -d@simple-deploy-k8s.json -X POST -H "Content-Type: application/json" -H "Accept: /" http://10.233.21.135:8084/pipelines

- Start manual execution of the Simple Deploy Pipeline to deploy "production" with a success rate of "70"

- Start "injector" to create traffic towards the application

kubectl -n default run injector --image=alpine -- \
    /bin/sh -c "apk add --no-cache --yes curl; \
    while true; do curl -sS --max-time 3 \
    http://sampleapp:8080/; done"

- Check the logs of "injector" to see high number of server error messages

kubectl -n default logs -f \
    $(kubectl -n default get pods -l run=injector \
    -o=jsonpath='{.items[0].metadata.name}')

- Check the health of the app on Prometheus
sum(requests) by(http_code)

Create Canary Deploy Pipeline
-----------------------------

- Download the description of the Canary Deploy pipeline.
wget https://raw.githubusercontent.com/spinnaker/spinnaker/master/solutions/kayenta/pipelines/canary-deploy.json

- Get pipeline id for Simple Deploy so the Canary Deploy pipeline can use it to fetch the baseline version.
export PIPELINE_ID=$(curl http://10.233.21.135:8084/applications/sampleapp/pipelineConfigs/Simple%20deploy | jq -r '.id')

- Inject the ID of Simple Deploy pipeline into Canary Deploy pipeline

jq '(.stages[] | select(.refId == "9") | .pipeline) |= env.PIPELINE_ID | (.stages[] | select(.refId == "8") | .pipeline) |= env.PIPELINE_ID' canary-deploy-k8s.json | \
    curl -d@- -X POST \
    -H "Content-Type: application/json" -H "Accept: /" \
    http://10.233.21.135:8084/pipelines




References
==========

[1] https://cloud.google.com/solutions/automated-canary-analysis-kubernetes-engine-spinnaker
[2] https://github.com/spinnaker/spinnaker/tree/master/solutions/kayenta
