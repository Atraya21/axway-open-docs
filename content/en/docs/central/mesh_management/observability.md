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

The ALS Traceability Agent gets installed into your Kubernetes cluster as part of deploying the `apicentral-hybrid` helm chart. The agent logs and publishes transactions within the mesh. 
The agent publishes a summary of the transaction which can be seen in the API Observer. Once the transaction summary is expanded, we can see all the hops within a transactions including the request and response headers.

The ALS agent has two modes namely default and verbose. The default mode captures only the headers specified in the EnvoyFilter and the verbose mode captures all the headers in request and response flows. 

## Setup 

The ALS Traceability agent logs and publishes traffic within the Mesh. In order to generate traffic, we need to create certain resources in our mesh. 

First, we will create a Gateway in the namespace in which we installed our Mesh agents. Please note if you already have a Gateway resource, you can skip this step and specify that Gateway in the Virtual Service.  

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

If you have a Virtual Service resource already, simply add the name for the http route so that the API Service and the transactions can be linked in API Central:

```yaml
  http:
  - name: list 
```

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
 
 **From verbose to default **:
 
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
