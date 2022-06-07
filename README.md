# Use Case

Manually adding instrumentation to existing codebases can be a burden for teams. Pixie (a CNCF Sandbox Project) provides immediate, significant baseline visibility into the target system. Once deployed, it automatically collects full-body application requests from a variety of protocols, system metrics, and network-level data. Pixie's auto-instrumentation is powered by eBPF, a Linux kernel technology. Pixie agent also enables users to send data formatted in the OpenTelemetry format to any OpenTelemetry collector. 

AWS Distro for OpenTelemetry Collector (ADOT Collector) is an AWS supported version of the upstream OpenTelemetry Collector and is distributed by Amazon. It enables users to send telemetry data to AWS CloudWatch Metrics. ADOT also enables users to configure a local OTLP(OpenTelemetry Protocol) endpoint as a push based receiver, to send data hence collected to AWS CloudWatch/AMP using IAM Roles for Service Account (IRSA). OTLP specification describes the encoding, transport, and delivery mechanism of telemetry data between telemetry sources (eBPF) , intermediate nodes such as collectors and telemetry backends (AWS CloudWatch/AMP).

In this proof of concept, we will walk through the reference implementation and step by step on how to auto instrument EBPF data collection on a EKS Cluster using Pixie agent and then use ADOT with a local OTLP endpoint to send the data to AWS CloudWatch/AMP/AMG for querying, visualization and alerting .


# Enabling ADOT EKS AddOn

* Grant permissions to Amazon EKS add-ons to install ADOT:

```
kubectl apply -f https://amazon-eks.s3.amazonaws.com/docs/addons-otel-permissions.yaml
```

* Install cert manager

```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.0/cert-manager.yaml
```

* Create IRSA

**NOTE** - REPLACE_ME with clustername

```
eksctl create iamserviceaccount \
    --name adot-collector \
    --namespace default \
    --cluster REPLACE_ME \
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonPrometheusRemoteWriteAccess \
    --attach-policy-arn arn:aws:iam::aws:policy/AWSXrayWriteOnlyAccess \
    --attach-policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
    --approve \
    --override-existing-serviceaccounts
```

* Enable ADOT 

**NOTE** - REPLACE_ME with clustername

```
aws eks create-addon --addon-name adot --cluster-name REPLACE_ME
```

* Create CRD for `OpenTelemetryCollector` which creates a GRPC OTLP receiver and CloudWatch as exporter

**NOTE** - REPLACE_ME with AWS region id where CloudWatch metrics are sent.

```
kubectl apply -f otel-crd.yaml
```
 
* (OPTION) Test data flowing in CloudWatch by deploying sample apps under `sample-app` folder


# Install Pixie agent

* Create Pixie account at `https://work.withpixie.ai/` using email and password
* Create a deploy key as mentioned [here](https://docs.pixielabs.ai/reference/admin/deploy-keys/#create-a-deploy-key)
* Deploy Pixie

**NOTE** - REPLACE_ME with clustername and deploykey 

```
helm repo add pixie-operator https://pixie-operator-charts.storage.googleapis.com

# Get latest information about Pixie chart.
helm repo update

# Install the Pixie chart (No OLM present on cluster).
helm install pixie pixie-operator/pixie-operator-chart --set deployKey=REPLACE_ME --set clusterName=REPLACE_ME --namespace pl --create-namespace
```

* Test by executing `pxl` in scratchpad (Select the `Scratch Pad` script from the `script` drop-down menu in the top left at Pixie UI .). You should see metrics flowing in CloudWatch

**NOTE** : Replace the OTEL URL as appropriate in the pxl

* Enable `OpenTelemetry` Plugin from https://work.withpixie.ai/admin/plugins . Ensure to first enable the plugin with the OTEL URL , then disable `Secure connections with TLS` and dont forget to hit `Save` . Refresh the UI to confirm the settings persist. Now within 10 seconds, you should see data flowing in (with default PXL scripts) in CloudWatch.

 