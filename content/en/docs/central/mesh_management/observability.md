---
title: Observability in Mesh
linkTitle: Observability in Mesh
weight: 160
date: 2021-03-03
description: Observe transactions in mesh.
---
{{< alert title="Public beta" color="warning" >}}This is a preview of new ALS Traceability Agent, which run separately from the current mesh governance agents and provide full governance of your hybrid environment. The new agent is deployed and configured from the Axway CLI, and it allows for observability in the mesh environment.{{< /alert >}}

## Before you begin

Before you start, see [Deploy your agents with the Amplify CLI](/docs/central/mesh_management/deploy-your-agents-with-the-amplify-cli) to learn how to use the CLI to install the mesh agents into your Kubernetes cluster.

This page will reference the resources created from the [Deploy your agents with the Amplify CLI](/docs/central/mesh_management/deploy-your-agents-with-the-amplify-cli) procedure.

## Prerequisites

These prerequisites are required by the Amplify Central CLI, which you will use to configure the Istio discovery agents.

* Node.js 8 LTS or later
* Minimum Amplify Central CLI version: 0.10.0

For more information, see [Install Amplify Central CLI](/docs/central/cli_central/cli_install/index.html).

## Overview

The ALS Traceability Agent gets installed into your Kubernetes cluster as part of deploying the `apicentral-hybrid` helm chart. The traceability agent (TA) sends metrics and logs for API activity back to AMPLIFY Central so that you can monitor service activity and troubleshoot your services. 
The agent publishes a summary of the transaction which can be seen in the API Observer. Once the transaction summary is expanded, we can see all the hops within a transactions including the request and response headers.

The ALS agent has two modes namely default and verbose. The default mode captures only the headers specified in the EnvoyFilter and the verbose mode captures all the headers in request and response flows. 

## Setup 

The ALS Traceability agent logs and publishes traffic within the Mesh. In order to generate traffic, we need to create certain custom resource definitions (crds) in our mesh. 

**Gateway**:

First, we will create a Gateway in the namespace in which we installed our Mesh agents. Please note if you already have a Gateway crd, you can skip this step and specify that Gateway in the Virtual Service. 

In the example below we spcify the selector as the "istio-apic-ingress" i.e., the ingress gateway that is installed during the istio installation step in [Deploy your agents with the Amplify CLI](/docs/central/mesh_management/deploy-your-agents-with-the-amplify-cli). If you have a separate ingress gateway that you would like to use, change the spec.selector.istio field to that ingress gateway instead. 

**Note** For more information about Gateways crd please refer to [Istio documentation] (https://istio.io/latest/docs/reference/config/networking/gateway/).

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: gateway-ingress
  namespace: <<apic-control>>
spec:
  selector:
    istio: istio-apic-ingress
  servers:
  - hosts:
    - <<host>>.hybrid.sandbox.axwaytest.net
    port:
      name: <<port-HTTP>>
      number: <<8080>>
      protocol: <<HTTP>>
 ```
 
Once configured, create the resource using the command:

 ```bash
 kubectl apply -f <fileName>.yaml
 ```
**Virtual Service**:

Next, we will create the Virtual Service for our services within the mesh. 

The example below applies to the "list" demo service that comes with the Hybrid installation. If you already have a Virtual Service, please skip this step and continue

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: route-to-list
  namespace: apic-control
spec:
  hosts:
  - "*"
  gateways:
  - gateway-ingress
  http:
  - name: list
    match:
    - uri:
        prefix: /mylist
    rewrite:
      uri: /api
    route:
    - destination:
        host: apic-hybrid-list.apic-demo.svc.cluster.local
        port:
          number: 8080
 ```
 
Once configured, create the resource using the command:

 ```bash
 kubectl apply -f <fileName>.yaml
 ```

**Pre-existing Virtual Service**:

If you have a Virtual Service resource already, simply add the name for the http route so that the API Service and the transactions can be linked in API Central:

```yaml
  http:
  - name: list 
```
**Note** The name specified under http.name field should be the same as the API service name

**Service Entry**:

If you have an egress hop within the service, then we need to create a service entry. See the example below:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: httpbin.org
  namespace: apic-control
spec:
  hosts:
  - httpbin.org
  ports:
  - name: http-80
    number: 80
    protocol: HTTP
  resolution: DNS
```

**API Central Resources**:

In order to link the transactions with the API Services in API central, certain resources need to be created with the "externalAPIID" specified in the attributes. 

**Note** This step is NOT required to just view transactions in API Observer. The step is only required in the case of linking those transactions to a specific API Service

```yaml
kind: APIService
name: <<serviceName>>
metadata:
  scope:
    kind: Environment
    name: <<environmentName>>
attributes:
  externalAPIID: <<clustername-http.name>> # http.name from VS
spec: {}  
---
kind: APIServiceRevision
name: <<revisionName>>
metadata:
  scope:
    kind: Environment
    name: <<environmentName>>
attributes:
  externalAPIID: <<clustername-http.name>> # http.name from VS
spec:
  apiService: <<serviceName>>
  definition:
    type: "oas2"
    value: <<base64 encoded spec>>
---
kind: APIServiceInstance
name: <<serviceName>>
metadata:
  scope:
    kind: Environment
    name:  <<environmentName>>
spec:
  apiServiceRevision: <<revisionName>>
  endpoint:
    - host: "apicentral.axway.com"
      port: 8080
      protocol: http
      routing:
        basePath: <<basePath>>
```

Here in an example for the Hybrid List demo service: 

```yaml
kind: APIService
name: mylist
metadata:
  scope:
    kind: Environment
    name: meshone
attributes:
  externalAPIID: clusterone-mylist
spec: {}  
---
kind: APIServiceRevision
name: list-v1
metadata:
  scope:
    kind: Environment
    name: meshone
attributes:
  externalAPIID: clusterone-mylist
spec:
  apiService: mylist
  definition:
    type: "oas2"
    value: eyJzd2FnZ2VyIjoiMi4wIiwiaW5mbyI6eyJ0aXRsZSI6Im15bGlzdCIsImRlc2NyaXB0aW9uIjoiQW4gQVBJIEJ1aWxkZXIgc2VydmljZSIsInZlcnNpb24iOiIxLjAuMCJ9LCJob3N0IjoiMTAwLjk5LjIyMi4xMDI6ODA4MCIsImJhc2VQYXRoIjoiL2FwaSIsInBhdGhzIjp7Ii9saXN0Ijp7ImdldCI6eyJyZXNwb25zZXMiOnsiMjAwIjp7ImRlc2NyaXB0aW9uIjoiT0siLCJzY2hlbWEiOnsidHlwZSI6ImFycmF5IiwiaXRlbXMiOnsiJHJlZiI6IiMvZGVmaW5pdGlvbnMvbGlzdCJ9fX19fSwicG9zdCI6eyJyZXNwb25zZXMiOnsiMjAwIjp7ImRlc2NyaXB0aW9uIjoiT0siLCJzY2hlbWEiOnsidHlwZSI6Im9iamVjdCIsInByb3BlcnRpZXMiOnt9fX19LCJwYXJhbWV0ZXJzIjpbeyJpbiI6ImJvZHkiLCJuYW1lIjoiYm9keSIsInNjaGVtYSI6eyIkcmVmIjoiIy9kZWZpbml0aW9ucy9saXN0In19XX19LCIvbGlzdC97aWR9Ijp7ImdldCI6eyJyZXNwb25zZXMiOnsiMjAwIjp7ImRlc2NyaXB0aW9uIjoiT0siLCJzY2hlbWEiOnsiJHJlZiI6IiMvZGVmaW5pdGlvbnMvbGlzdCJ9fX0sInBhcmFtZXRlcnMiOlt7ImluIjoicGF0aCIsIm5hbWUiOiJpZCIsInR5cGUiOiJzdHJpbmciLCJyZXF1aXJlZCI6dHJ1ZX1dfSwicGFyYW1ldGVycyI6W3sibmFtZSI6ImlkIiwiaW4iOiJwYXRoIiwidHlwZSI6InN0cmluZyIsInJlcXVpcmVkIjp0cnVlfV0sImRlbGV0ZSI6eyJyZXNwb25zZXMiOnsiMjAwIjp7ImRlc2NyaXB0aW9uIjoiT0siLCJzY2hlbWEiOnsidHlwZSI6Im9iamVjdCIsInByb3BlcnRpZXMiOnt9fX19LCJwYXJhbWV0ZXJzIjpbeyJpbiI6InBhdGgiLCJuYW1lIjoiaWQiLCJ0eXBlIjoic3RyaW5nIiwicmVxdWlyZWQiOnRydWV9XX19fSwiZGVmaW5pdGlvbnMiOnsibGlzdCI6eyJ0eXBlIjoib2JqZWN0IiwidGl0bGUiOiJsaXN0IiwicHJvcGVydGllcyI6eyJuYW1lIjp7InR5cGUiOiJzdHJpbmcifSwicHJpY2UiOnsidHlwZSI6InN0cmluZyJ9LCJzdG9yZSI6eyJ0eXBlIjoic3RyaW5nIn19fSwiUmVzcG9uc2VNb2RlbCI6eyJ0eXBlIjoib2JqZWN0IiwicmVxdWlyZWQiOlsic3VjY2VzcyIsInJlcXVlc3QtaWQiXSwiYWRkaXRpb25hbFByb3BlcnRpZXMiOmZhbHNlLCJwcm9wZXJ0aWVzIjp7ImNvZGUiOnsidHlwZSI6ImludGVnZXIiLCJmb3JtYXQiOiJpbnQzMiJ9LCJzdWNjZXNzIjp7InR5cGUiOiJib29sZWFuIiwiZGVmYXVsdCI6ZmFsc2V9LCJyZXF1ZXN0LWlkIjp7InR5cGUiOiJzdHJpbmcifSwibWVzc2FnZSI6eyJ0eXBlIjoic3RyaW5nIn0sInVybCI6eyJ0eXBlIjoic3RyaW5nIn19fSwiRXJyb3JNb2RlbCI6eyJ0eXBlIjoib2JqZWN0IiwicmVxdWlyZWQiOlsibWVzc2FnZSIsImNvZGUiLCJzdWNjZXNzIiwicmVxdWVzdC1pZCJdLCJwcm9wZXJ0aWVzIjp7ImNvZGUiOnsidHlwZSI6ImludGVnZXIiLCJmb3JtYXQiOiJpbnQzMiJ9LCJzdWNjZXNzIjp7InR5cGUiOiJib29sZWFuIiwiZGVmYXVsdCI6ZmFsc2V9LCJyZXF1ZXN0LWlkIjp7InR5cGUiOiJzdHJpbmcifX19fX0=
---
kind: APIServiceInstance
name: mylist
metadata:
  scope:
    kind: Environment
    name: mesh
spec:
  apiServiceRevision: list-v1
  endpoint:
    - host: "apicentral.axway.com"
      port: 8080
      protocol: http
      routing:
        basePath: "/mylist"
```

Once configured, please use the following command to populate the resources in API Central:

 ```bash
amplify central apply -f <fileName>.yaml 
 ```

The set up is complete for observability in the mesh. In order to verify view transactions in API Observer, generate some traffic for the service:

```bash
curl -v http://demo.sandbox.axwaytest.net:8080/mylist/list
 ```
 ![AMPLIFY Central control plane](/Images/central/Transactions.png)
 

## Toggling the traceability agent

After deploying the `apicentral-hybrid` helm chart to your Kubernetes cluster, you can see ALS Traceability agent running. The service is called `apic-hybrid-als`. During the step [Deploy your agents with the Amplify CLI](/docs/central/mesh_management/deploy-your-agents-with-the-amplify-cli), we were able to select which the mode for the ALS agent. If you want to switch the mode please follow the procedure below. 

**From default to verbose**:

Edit the Istio-override.yaml file's configuration under the meshConfig section to include 
```yaml
spec:
  meshConfig:
    enableTracing: true
    enableEnvoyAccessLogService: true
    defaultConfig:
      envoyAccessLogService:
        address: apic-hybrid-als.apic-control.svc.cluster.local:9000
 ```

After the change, please re-install Istio again:
 
 ```bash
 istioctl install --set profile=demo -f istio-override.yaml
 ```
 After the Istio re-installation, run the following command to set the ALS agent's mode to "verbose"
 
  ```bash
helm repo update
helm upgrade --install --namespace apic-control apic-hybrid axway/apicentral-hybrid -f hybrid-override.yaml --set als.mode="verbose"
 ```
 
 **From verbose to default**:
 
 Edit the Istio-override.yaml file's configuration under the meshConfig section to set enableEnvoyAccessLogService as false as shown below
 
```yaml
spec:
  meshConfig:
    enableTracing: true
    enableEnvoyAccessLogService: false
    defaultConfig:
      envoyAccessLogService:
        address: apic-hybrid-als.apic-control.svc.cluster.local:9000
 ```
 
 After the change, please re-install Istio again:
 
 ```bash
 istioctl install --set profile=demo -f istio-override.yaml
 ```
 
 After the Istio re-installation, run the following command to set the ALS agent's mode to "default"
 
  ```bash
helm repo update
helm upgrade --install --namespace apic-control apic-hybrid axway/apicentral-hybrid -f hybrid-override.yaml --set als.mode="default"
 ```
 
 Aditionally, in default mode the agent can be confgiured to only capture certain request and response headers. By default, we capture all the headers specified in the EnvoyFilter configuration below. See "additional_request_headers_to_log" and "additional_response_headers_to_log" section. 
 
 ```yaml
 apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: patch-gateway-and-sidecars-with-als
  namespace: <<envoyFilterNamespace>>
spec:
  configPatches:
    - applyTo: NETWORK_FILTER
      match:
        context: ANY
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
      patch:
        operation: MERGE
        value:
          name: "envoy.filters.network.http_connection_manager"
          typed_config:
            "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager"
            access_log:
              - name: envoy.access_loggers.http_grpc
                typed_config:
                  "@type": "type.googleapis.com/envoy.extensions.access_loggers.grpc.v3.HttpGrpcAccessLogConfig"
                  additional_request_headers_to_log: ["accept","user-agent","x-envoy-decorator-operation","x-envoy-external-address","x-forwarded-client-cert","x-forwarded-for","x-forwarded-proto","x-request-id","x-b3-parentspanid","x-b3-spanid","x-istio-attributes"]
                  additional_response_headers_to_log: ["connection","content-length","content-md5","content-type","date","etag","date","request-id","response-time","server","start-time","vary"]
                  common_config:
                    filter_state_objects_to_log: ["wasm.upstream_peer","wasm.upstream_peer_id","wasm.downstream_peer","wasm.downstream_peer_id"]
                    log_name: mesh
                    grpc_service:
                      google_grpc:
                        target_uri: apic-hybrid-als.apic-control.svc.cluster.local:9000
                        stat_prefix: apic-hybrid-als
 ```
 
 To exclude any headers, remove them from "additional_request_headers_to_log" and "additional_response_headers_to_log". Please note unless otherwise specified <<envoyFilterNamespace>> is "istio-system". Once the configuration is changed, run the following command:
 
  ```bash
 kubectl apply -f <fileName>.yaml
 ```
