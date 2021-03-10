---
title: Discover APIs and services
linkTitle: Discover APIs and services
weight: 30
date: 2020-11-19
description: Learn how to configure the Axway Istio agents to discover your APIs and services.
---
{{< alert title="Early Preview" color="warning" >}}This is a preview of new Istio discovery agents, which are deployed separately from the Istio POC agents that provide full governance of your hybrid environment. The new agents are deployed and configured from the Axway CLI, and they monitor Kubernetes resource discovery.{{< /alert >}}

## Before you begin

Before you start, see [Deploy your agents with the Amplify CLI](/docs/central/mesh_management/deploy-your-agents-with-the-amplify-cli) to learn how to use the CLI to install the mesh agents into your Kubernetes cluster and to create default resources to discover the demo service that gets deployed.

This page will reference the resources created from the [Deploy your agents with the Amplify CLI](/docs/central/mesh_management/deploy-your-agents-with-the-amplify-cli) procedure.

## Prerequisites

These prerequisites are required by the Amplify Central CLI, which you will use to configure the Istio discovery agents.

* Node.js 8 LTS or later
* Minimum Amplify Central CLI version: 0.13.0

For more information, see [Install Amplify Central CLI](/docs/central/cli_central/cli_install/index.html).

## Overview

Discovery agents are services that get installed into your Kubernetes cluster as part of deploying the `apicentral-hybrid` helm chart.

The API Discovery Agent (ADA) uses a resource called `SpecDiscovery` to find Swagger documentation exposed over an HTTP endpoint. The `SpecDiscovery` provides configuration details to the ADA to instruct it where to find documentation inside of a cluster.

You can configure the ADA with one or multiple SpecDiscoveries. As long as the ADA has at least one `SpecDiscovery` it will check all new events coming from Kubernetes to check if that event meets the match criteria defined by any of the known SpecDiscoveries. If the event is triggered by a pod, and the pod meets the match criteria, then the ADA will create an `APISpec` to capture the existence of documentation for that pod in the cluster, and save it in Amplify Central.

The Resource Discovery Agent (RDA) uses a resource called `ResourceDiscovery` to find pods and services running in a cluster. The `ResourceDiscovery` provides configuration details to the RDA to instruct it where to find pods and services inside of a cluster.

You can configure the RDA with one or multiple ResourceDiscoveries. As long as the RDA has at least one `ResourceDiscovery` it will check all new events coming from Kubernetes to see if that event meets the match criteria defined by any of the known ResourceDiscoveries. If the event is triggered by a pod or service, and if the pod or service meets the match criteria, then the RDA will create a K8SResource to capture the existence of the resource and save it in Amplify Central.

For more information about the discovery agents, see [Amplify Central resources for discovering APIs and services](#Amplify-Central-resources-for-discovering-APIs-and-services).

## Start the discovery agents

After deploying the `apicentral-hybrid` helm chart to your Kubernetes cluster, you can see two discovery agents running. The services are called `apic-hybrid-ada` and `apic-hybrid-rda`. These two agents are responsible for discovering pods, services, and documentation inside of your Kubernetes cluster. After they discover a service, they report back to Amplify Central where the service will be represented as an API service in your environment.

## How the discovery agents report events to Amplify Central

Kubernetes triggers the following events: add, update, and delete. The discovery agents use these events to report back to Amplify Central what has been discovered in your cluster, based on the match configuration for each agent.

### Event: add

An add event occurs when a new pod or service is created in Kubernetes and it meets the match criteria defined by the configuration of the agent, then the agent reports back to Amplify Central that a resource has been discovered.

### Event: update

An update event occurs when an existing pod or service is in the cluster and some change takes place, such as changing the labels for that resource.

When a pod or service is updated in Kubernetes:

* If it did not previously meet the match criteria defined by the configuration for the agent, but after the update event it *does* meet the match criteria, it will be reported back to Amplify Central that a new resource has been discovered.
* If it *did* previously meet the match criteria defined by the configuration for the agent, but after the update event it does *not* meet the match criteria, it will be reported back to Amplify Central that the resource that previously matched has been deleted.
* If it *did* previously meet the match criteria defined by the configuration for the agent, and after the update event it *continues* to meet the match criteria, it will be reported back to Amplify Central that the resource has been updated.

### Event: delete

A delete event occurs when an existing pod or service is removed from the cluster.

When a pod or service is deleted in Kubernetes, the agents check if the deleted resource was previously matched. If the resource was matched, then the agents will report back to Amplify Central that the resource it currently matches has been deleted.

## Amplify Central resources for discovering APIs and services

The discovery agents use following resources:

* **Environment** - Represents an Amplify Central environment.
* **Mesh** - Represents a kubernetes service mesh, and has a reference to an environment. Multiple K8SClusters can reference a mesh.
* **MeshDiscovery** - Used to promote services discovered in a kubernetes cluster to an environment.
* **K8SCluster** - Represents a Kubernetes cluster. It has a reference to a single mesh resource.
* **ResourceDiscovery** - Configuration for the RDA for discovering pods and services in a Kubernetes cluster. It is scoped to a K8SCluster.
* **SpecDiscovery** - Configuration for the ADA to discover API documentation exposed over an HTTP endpoint. It is scoped to a K8SCluster
* **APISpec** - A resource created by the ADA to represent unique API documentation found in a cluster. The same documentation may be found across multiple pods, especially if you are running multiple instances of one pod. One APISpec is created for one unique documentation found in a cluster. If multiple pods expose the same documentation, they will be grouped together in one APISpec. APISpecs are scoped to a K8SCluster and have a reference to a `SpecDiscovery`. APISpecs are created by the ADA as a response to finding a pod that exposes documentation as described by the `SpecDiscovery`.
* **K8SResource** - A resource created by the RDA that represents a pod or service discovered in Kubernetes. The K8SResource is scoped to a K8SCluster, and it has a reference to a ResourceDiscovery. K8SResources are created by the RDA as a response to find a pod or service that meets the match criteria described by the `ResourceDiscovery`.

## Log in to the Amplify Central CLI

Run the following command to log into the Central CLI with your Amplify Platform credentials:

```shell
amplify auth login --client-id apicentral
```

A dialog box is shown. Enter your valid credentials (email address and password), and after the authorization successful message is displayed go back to the Amplify CLI.

If you are a member of multiple Amplify organizations, select an organization and continue.

## Create a K8SCluster

The K8SCluster resource is a representation of your Kubernetes cluster. All of the other resources previously mentioned will be scoped to a K8SCluster.

If you previously followed [Deploy the service mesh and Axway Istio agents](/docs/central/mesh_management/add_env/#deploy-the-service-mesh-and-axway-mesh-agents), then the ADA and RDA will already be configured with your selected K8SCluster, and you can skip to [Configure the API Discovery Agent](#Configure-the-API-discovery-agent).

If you do not have the ADA and RDA configured, or if you would like the agents to use a different K8SCluster, follow these steps:

1. To create a K8SCluster on your local environment, create a file called `k8scluster.yaml` with the following content.

   ```yaml
   apiVersion: v1alpha1
   group: management
   kind: K8SCluster
   name: mesh-env
   title: mesh-env
   spec: {}
   ```
2. Change the name and the title of the K8SCluster to represent your cluster:

   ```bash
   amplify central create -f ./k8scluster.yaml
   ```

3. If the command was successful, you should see a message indicating that the resource was created:

   ```bash
   ~ » amplify central create -f ./k8scluster.yaml
   ✔ "k8scluster/mesh-env" has successfully been created.
   ```

4. Run the following command to retrieve the K8SCluster and view the YAML content:

   ```bash
   ~ » amplify central get k8sclusters mesh-env -o yaml
   group: management
   apiVersion: v1alpha1
   kind: K8SCluster
   name: mesh-env
   title: mesh-env
   metadata:
     id: 8ac992d875ac0c170175b3bd93fd0000
     audit:
       createTimestamp: '2020-11-10T19:59:07.261+0000'
       createUserId: 31c9538c-32ad-4db7-a5e4-020470accd0c
       modifyTimestamp: '2020-11-10T19:59:07.261+0000'
       modifyUserId: 31c9538c-32ad-4db7-a5e4-020470accd0c
     resourceVersion: '309'
     references: []
     selfLink: "/management/v1alpha1/k8sclusters/mesh-env"
   attributes: {}
   finalizers: []
   tags: []
   spec: {}
   ```

5. After the K8SCluster is created, you must update the `apic-hybrid-ada` and `apic-hybrid-rda` pods to connect to the new K8SCluster. Run the following commands to update each pod with the new environment variable:

   ```bash
   ~ » kubectl set env deployment/apic-hybrid-ada CLUSTERNAME=<YOUR-K8SCLUSTER-NAME> -n apic-control
   deployment.extensions/apic-hybrid-ada env updated
   ```

   ```bash
   ~ » kubectl set env deployment/apic-hybrid-rda CLUSTERNAME=<YOUR-K8SCLUSTER-NAME> -n apic-control
   deployment.extensions/apic-hybrid-ada env updated
   ```

## Deploy the Sunset app

To demonstrate how the discovery agents work, we will deploy a new service called `sunset` into a new namespace called `sunset-demo`. Then, we will create a new `SpecDiscovery` and ResourceDiscoveries that will be responsible for discovering the `sunset` pod, service, and documentation.

The steps are as follows:

1. Copy the following content into a yaml file named `sunset.yaml` on your machine.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: sunset
        apic-managed: "true"
      creationTimestamp: null
      name: sunset
      namespace: sunset-demo
    spec:
      ports:
      - port: 8080
        protocol: TCP
        targetPort: http
        name: http
      selector:
        run: sunset
    status:
      loadBalancer: {}
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      creationTimestamp: null
      labels:
        run: sunset
      name: sunset
      namespace: sunset-demo
    spec:
      replicas: 1
      selector:
        matchLabels:
          run: sunset
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: sunset
            run: sunset
        spec:
          containers:
          - image: axway.jfrog.io/ampc-public-docker-release/sunset:latest
            imagePullPolicy: Always
            name: sunset
            ports:
            - containerPort: 8080
              name: http
            resources: {}
    status: {}
    ```
2. Create a new namespace named `sunset-demo`:

   ```bash
   ~ » kubectl create namespace sunset-demo
   namespace/sunset-demo created
   ```
3. Deploy the `sunset` app:

   ```bash
   ~ » kubectl apply -f sunset.yaml
   service/sunset created
   deployment.apps/sunset created
   ```
4. Verify that the deployment was successful and that the `sunset` app is in a running status:

   ```bash
   ~ » kubectl get pods -n sunset-demo
   NAME                                READY   STATUS    RESTARTS   AGE
   sunset-749ddd444-ncllr              1/1     Running   0          22m
   ```

The following sections will cover how to configure the ADA and the RDA to discover the `sunset` app.

### Configure the API discovery agent

The `SpecDiscovery` is a resource that configures how the ADA identifies which pods to examine for documentation.

To create a `SpecDiscovery`, follow these steps:

1. Create a new file locally named `spec-discovery.yaml`, with the following content:

   ```yaml
   apiVersion: v1alpha1
   kind: SpecDiscovery
   group: management
   name: sunset-discovery
   metadata:
     scope:
       kind: K8SCluster
       name: mesh-env
   spec:
     namespaceFilter:
       names:
       - sunset-demo
     resourceFilter:
       matchLabels:
         discover: 'true'
     targets:
       exactPaths:
       - path: "/apidocs"
         headers:
           Content-Type: application/json
       fromAnnotation:
       - template: docpath.apicentral.io/{{.Port.Name}}
   ```
2. Update the `metadata.scope.name` field to reflect the same name given to the K8SCluster. This links the `SpecDiscovery` to the K8SCluster. To verify the K8SCluster name that is being used for the ADA, run:

   ```bash
   ~ » kubectl set env deployment/apic-hybrid-ada --list -n apic-control | grep CLUSTERNAME
   CLUSTERNAME=mesh-env
   ```

   Take the name from the output of the command above and place it in the `metadata.scope.name` field.

You can configure the `SpecDiscovery` in many ways so that the ADA can identify which pods are exposing documentation. The ADA uses the `SpecDiscovery` to identify which namespace to search to find pods. Once the agent knows the namespace to search, it will then identify pods to match based on the labels of the pod, or by the name of the pod. The matching of the pod is done by comparing the pod labels to the `spec.resourceFilter.matchLabels` or `spec.resourceFilter.names` fields.

Read through the descriptions of the fields below, and update the `SpecDiscovery` to make the documentation correctly discovered in your cluster.

* `spec.namespaceFilter.names`: A list of namespaces to follow. If the value is not defined, or it is empty, the ADA will look in all namespaces.
* `spec.resourceFilter`: This object has two keys, `names` and `matchLabels`. Both keys are optional, and they provide two different ways to find a pod. This object is used to configure how the agent will identify a pod to search for documentation. If this value is an empty object, or it is not set, the ADA will look for documentation in all pods found in the provided namespaces.
* `spec.resourceFilter.matchLabels`: An object containing labels that a pod must have in order for the ADA to discover the pod. If this object is empty or not provided, then matching will be done based on the `names` list.
* `spec.resourceFilter.names`: A list of pod names to find in the provided namespaces. If the list is empty or not provided, then matching will be done based on the `matchLabels`.
* `spec.targets`: An object that identifies where to find documentation on an exposed endpoint after a pod has been matched.
* `spec.targets.exactPaths`: A list of objects that identifies where to find the documentation and the headers to send with the request to retrieve the documentation.
* `spec.targets.exactPaths.path`: The endpoint where the documentation can be found. The full host name is not needed because the discovery agent will have already matched the pod, and it will use the pod's IP address and this path to find the documentation. The ADA will make one request to this endpoint for every exposed port found in the pod.
* `spec.targets.exactPaths.headers`: The headers to send with the request.
* `spec.targets.exactPaths.fromAnnotations`: A list of annotations that the `SpecDiscovery` might use to make a request to a pod to fetch documentation. If this field is provided, and if the ADA finds a pod with a matching annotation found in this list, then the ADA will make a request to the pod based on the value of the pod annotation.
* `spec.targets.exactPaths.fromAnnotations.template`: A string template that matches an annotation found on a pod. To learn more about this field, see [Defining a pod annotation](#Defining-a-pod-annotation)

### Discover your APIs

Now that the details of the `SpecDiscovery` are explained, let's see an example of how to get the ADA to discover the `sunset` pod running in the `sunset-demo` namespace. The configuration will be created to match the `sunset` pod based on the labels of the pod.

Follow these steps to discover the pod:

1. confirm that the `sunset` app is running.

   ```bash
   ~ » kubectl get pods -n sunset-demo
   NAME READY STATUS RESTARTS AGE
   sunset-749ddd444-ncllr              2/2     Running   0          98m
   ```
2. If there is a pod that begins with `sunset` and it is in a running status, then you are ready to continue. To confirm that the ADA is working run the command below:

   ```bash
   ~ » kubectl get pods -n apic-control
   NAME                               READY   STATUS    RESTARTS   AGE
   apic-hybrid-ada-6cdc497bdf-r2zht   1/1     Running   0          50m
   apic-hybrid-rda-69ddfbd88c-9pws2   1/1     Running   0          50m
   ```
3. In order to match the `sunset` pod, you must add a label to the pod so that it can be picked up by the discovery agent. This label also needs to be defined in the `spec.resourceFilter.matchLabels` field of the `SpecDiscovery` in order for the agent to find it. The label can be anything you want. In this example, we will add `discover=true` to the pod labels as follows:

   ```bash
   ~ » kubectl label pod -n sunset-demo sunset-749ddd444-ncllr discover=true
   pod/sunset-749ddd444-ncllr labeled
   ```
4. After labeling the pod, you must update three fields on the `SpecDiscovery` so that the ADA can find the `sunset` pod running in the `sunset-demo` namespace:

   4.1. Update the `spec.namespaceFilter.names` array and provide the namespace the agent should watch when looking for new pods:

   ```yaml
     names:
     - sunset-demo
   ```

   4.2. Update `spec.resourceFilter.matchLabels` and provide the label to be used so that the ADA will know which pods to search for documentation.

   ```yaml
     matchLabels:
       discover: "true"
   ```

   4.3. Update the `spec.targets.exactPaths` array and provide an object with a `path` field and a `headers` field so that the ADA knows where to make a request to the pod to collect the documentation.

   ```yaml
     exactPaths:
     - path: /apidocs
       headers:
         Content-Type: application/json
   ```
5. After the `SpecDiscovery` has been updated, you can create the resource and the ADA will look for the `sunset` pod based on the new configuration.

   ```bash
   ~ » amplify central create -f ./spec-discovery.yaml
   ✔ "specdiscovery/sunset-discovery" has successfully been created.
   ```
6. After creating the new `SpecDiscovery`, run the following command to see two SpecDiscoveries. The `sunset-discovery` is the resource that will be used to find the documentation from the `sunset` pod.

   ```bash
   ~ » amplify central get specdiscoveries -s mesh-env
   ✔ Resource(s) has successfully been retrieved

   NAME                 AGE             TITLE                SCOPE KIND  SCOPE NAME
   sunset-discovery     18 minutes ago  apic-demo-discovery  K8SCluster  mesh-env
   cli-1605551801337    2 hours ago     cli-1605551801337    K8SCluster  mesh-env
   ```
7. The `apic-hybrid-ada` pod will see the new `SpecDiscovery` configuration and will start looking for pods that match the criteria it has specified. Run the following command to retrieve your APISpecs. APISpecs are created from the ADA in response to finding a pod that matches, based on the `SpecDiscovery` match criteria.

   ```bash
   amplify central get apispecs -s <YOUR-K8SCLUSTER-NAME> -o yaml
   ```

   Returns the APISpec that was created in response to the pod that matched the `SpecDiscovery`.

   ```bash
   ~ » amplify central get apispecs -s mesh-env
   ✔ Resource(s) has successfully been retrieved

   NAME              AGE         TITLE   SCOPE KIND  SCOPE NAME
   mylist100swagger  1 hour ago  mylist  K8SCluster  mesh-env
   sunsetapp100swagger  a few seconds ago  Sunset App  K8SCluster  mesh-env
   ```

If you see an APISpec named `sunsetapp100swagger` scoped to your K8SCluster, then you have successfully configured the ADA to search your Kubernetes cluster for pods exposing documentation. The `sunset` app documentation will now be visible in Amplify Central.

### Defining a pod annotation

The `SpecDiscovery` might contain a list of annotation templates defined at `spec.targets.exactPaths.fromAnnotations`. The objects in this list contain a single key, called `template`, which takes a string value. For example:

```yaml
fromAnnotations:
- template: docpath.apicentral.io/{.Port.Name}
```

`{.Port.Name}` is the template. There are two ways to define the template, you can use the name of the port, `{.Port.Name}`, or the actual port value `{.Port.Port}`. The rest of the example uses the `{.Port.Name}` template.

The ADA uses the Go library template to parse the `SpecDiscovery` template based on the name of the ports found on the pod that have been matched. A pod has at least one container, and a container might have an exposed port. In Kubernetes, these ports have a `name` and a `containerPort` key. When a pod has been discovered, the ADA will find all of the exposed ports on the pod, take the names of the ports, and parse the template with the given name.

If the ADA finds a port named `sunset-port`, it will parses the template as `docpath.apicentral.io/sunset-port`, and it will look for an annotation on the pod that matches this newly parsed template. If one template is found, the expected value of the annotation is a JSON object with a `path` and a `headers` field, exactly like the `spec.targets.exactPaths` field. If the pod does not contain an annotation that matches the parsed template, it will have no impact on the ADA.

To illustrate, let's use an example of a pod and assume it has been matched by the ADA.

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    docpath.apicentral.io/sunset-http-port: '{"path": "/swagger.yaml","headers": {"Content-Type": "application/yaml"}}'
  labels:
    discover: 'true'
  name: sunset-749ddd444-ld7rf
  namespace: sunset-demo
spec:
  containers:
  - image: axway-docker-public-registry.bintray.io/sunset
    imagePullPolicy: Always
    name: sunset
    ports:
    - containerPort: 8080
      name: sunset-http-port
      protocol: TCP
    - containerPort: 9090
      name: no-match
      protocol: TCP
```

Take a look at the array of ports first. The first port is named `sunset-http-port`. The name of the annotation is `docpath.apicentral.io/sunset-http-port`. By providing a template to the SpecDiscovey, such as `docpath.apicentral.io/{.Port.Name}`, the ADA will take the names of all ports and parse the template to create the final string. If the final parsed string matches any of the templates in the pod, then the JSON object that is provided in the annotation will be used to make the request to the pod.

There is another port in the example named `no-match`. The ADA will parse the template against all ports found in a pod. This will result in two parsed template: `docpath.apicentral.io/sunset-http-port` and `docpath.apicentral.io/no-match`. The ADA will look for each of these templates in the pod. In this example, there is only one annotation. There is no annotation for `docpath.apicentral.io/no-match`, meaning that the ADA will not perform any action with this parsed template.

The JSON object found describing a request from an annotation might be used alongside the `spec.targets.exactPaths` field. An exact path might be defined as part of the `SpecDiscovery`, and an annotation might be defined as well. The ADA always makes one request to each port found in a pod. In this case, the ADA will make four requests because the pod has two ports, and there are two request objects. One request object was defined in the `exactPaths` field and another request object was found in the pod annotation.

Here is an example. Let's assume that the ADA has stored an array of requests in memory:

```json
[
  {
    "path": "/apidocs",
    "headers": {
      "Content-Type": "application/json"
    }
  },
  {
    "path": "/swagger.yaml",
    "headers": {
      "Content-Type": "application/yaml"
    }
  }
]
```

The pod has two exposed ports, 8080 and 9090. The host IP for the pod is 172.0.44.37. The ADA will make the following requests:

```bash
curl http://172.0.44.37:8080/apidocs -H "Content-Type: application/json"
curl http://172.0.44.37:8080/swagger.yaml -H "Content-Type: application/yaml"
curl http://172.0.44.37:9090/apidocs -H "Content-Type: application/json"
curl http://172.0.44.37:9090/swagger.yaml -H "Content-Type: application/yaml"
```

This is the behavior of the ADA whether you decide to use an annotation template or not. You can pass more than one object to `exactPaths`. The ADA will make a request to each exposed port for each request object found, either in `exactPaths` or from an annotation found in a pod.

### Configure the resource discovery agent

The `ResourceDiscovery` is a resource that configures how the RDA identifies which pods and services to discover. The `ResourceDiscovery` can discover pods and services by specifying the `spec.kind` field to either `Pod` or `Service`. Multiple ResourceDiscoveries can be created. We will create one `ResourceDiscovery` to discover pods and another to discover services. The following example contains two YAML objects. The first configures the discovery of pods. The second configures the discovery of services.

Follow these steps to configure a resource discovery agent:

1. Create a new file locally named `resource-discoveries.yaml`, and copy the YAML data below into it.

    ```yaml
    ---
    apiVersion: v1alpha1
    group: management
    kind: ResourceDiscovery
    name: sunset-pod-discovery
    title: sunset-pod-discovery
    metadata:
      scope:
        kind: K8SCluster
        name: mesh-env
    spec:
      kind: Pod
      group: ''
      version: v1
      ignoreLabels:
      - originalLabel1
      namespaceFilter:
        names:
        - sunset-demo
      resourceFilter:
        matchLabels:
          discover: 'true'
    ---
    apiVersion: v1alpha1
    group: management
    kind: ResourceDiscovery
    name: sunset-service-discovery
    title: sunset-service-discovery
    metadata:
      scope:
        kind: K8SCluster
        name: mesh-env
    spec:
      kind: Service
      group: ''
      version: v1
      ignoreLabels:
      - originalLabel1
      namespaceFilter:
        names:
        - sunset-demo
      resourceFilter:
        matchLabels:
          discover: 'true'
    ```

2. Update the `metadata.scope.name` field to reflect the same name given to the K8SCluster. This links the ResourceDiscoveries to the K8SCluster. To verify the K8SCluster name that is being used for the ADA, run:

   ```bash
   ~ » kubectl set env deployment/apic-hybrid-rda --list -n apic-control | grep CLUSTERNAME
   CLUSTERNAME=mesh-env
   ```

   Take the name from the output of the command, and place it in the `metadata.scope.name` field.

The `ResourceDiscovery` can be configured in many ways so that the RDA can identify which pods and services to match so that you can have visibility on what is running inside of your cluster within Amplify Central. The RDA uses the `ResourceDiscovery` to identify which namespace to search and find pods and services. Once the agent knows the namespace to search, it will identify pods and services to match based on the labels of the pod, or by the name of the pod. The matching of the pod is done by comparing the pod labels to the `spec.resourceFilter.matchLabels` field or the `spec.resourceFilter.names` field.

Read through the description of the fields and update the `ResourceDiscovery` to make your pods and services correctly discovered in your cluster.

* `spec.kind`: The kind of Kubernetes resource to discover. Either Pod or Service.
* `spec.group`: A collection of Kinds that are logically related for the selected `kind`.
* `spec.version`: The version of the selected `kind`.
* `spec.namespaceFilter.names`: A list of namespaces to follow. If the value is not defined, or it is empty, the RDA will look in all namespaces.
* `spec.resourceFilter`: This object has two keys, `names` and `matchLabels`. Both keys are optional and they provide two ways to find a pod or service. This object is used to configure how the agent will identify a pod or service for matching. If this value is an empty object, or it is not set, the RDA will look for all pods or services found in the provided namespaces.
* `spec.resourceFilter.matchLabels`: An object containing labels that a pod must have in order for the RDA to discover the pod. If this object is empty or not provided, the matching will be done based on the `names` list.
* `spec.ignoreLabels`: An array of labels to ignore from the matching pod or service. Any labels in this array will not be copied over to the K8SResource that the RDA will create when a match is found.

### Discover your resources

This section shows an example of how to get the RDA to discover the `sunset` pod running in the `sunset-demo` namespace. We are going to create a configuration to match the `sunset` pod and the `sunset` service based on the labels of the pod and service.

To discover pods and services, follow these steps:

1. Verify that the `sunset` pod is running:

   ```bash
   ~ » kubectl get pods -n sunset-demo
   NAME READY STATUS RESTARTS AGE
   sunset-749ddd444-ncllr              2/2     Running   0          98m
   ```
2. Verify that the RDA is working:

   ```bash
   ~ » kubectl get pods -n apic-control
   NAME                               READY   STATUS    RESTARTS   AGE
   apic-hybrid-ada-6cdc497bdf-r2zht   1/1     Running   0          50m
   apic-hybrid-rda-69ddfbd88c-9pws2   1/1     Running   0          50m
   ```

   You should see a pod named `apic-hybrid-rda` in running status.
3. To match the pod, add a label to the pod to make it discoverable by the discovery agent. This label must be defined in the `spec.resourceFilter.matchLabels` field so that the `ResourceDiscovery` agent can find it. In this example, we will add `discover=true` to the pod labels:

   ```bash
   ~ » kubectl label pod -n sunset-demo sunset-749ddd444-ncllr discover=true
   pod/sunset-749ddd444-ncllr labeled
   ```
4. You must label the service as well:

   ```bash
   ~ » kubectl label service -n sunset-demo sunset discover=true
   service/sunset labeled
   ```

   After labeling the pod and the service, you must update two fields on each of the ResourceDiscoveries so that the RDA can find the `sunset` pod and the `sunset` service running in the `sunset-demo` namespace based on the newly added label.

5. Update the `spec.namespaceFilter.names` array and provide the namespace, which the agent should watch when looking for new pods.

   ```yaml
   names:
   - sunset-demo
   ```
6. Update `spec.resourceFilter.matchLabels` and provide the label, which the RDA will use to match pods or services.

   ```yaml
   matchLabels:
     discover: "true"
   ```

7. After the `ResourceDiscovery` is updated, you must create the resource to enable the RDA to look for the `sunset` pod and service based on the new configuration.

   ```bash
   amplify central get k8sresources -s <YOUR-K8SCLUSTER-NAME> -o yaml
   ~ » amplify central create -f ./resource-discoveries.yaml
   ✔ "resourcediscovery/sunset-pod-discovery" has successfully been created.
   ✔ "resourcediscovery/sunset-service-discovery" has successfully been created.
   ```

8. Finally, run the following command to return the K8SResources that were created in response to the pod that matched the `SpecDiscovery`.

   ```bash
   amplify central get k8sresources -s mesh-env
   ✔ Resource(s) has successfully been retrieved
   
   NAME                                             AGE         TITLE                      SCOPE KIND  SCOPE NAME
   pod.apic-demo.apic-hybrid-list-598f8f9b4b-jb7ds  an hour ago        pod-cli-1605565746103      K8SCluster  mesh-env
   pod.sunset-demo.sunset-749ddd444-ld7rf           a few seconds ago  sunset-pod-discovery       K8SCluster  mesh-env
   service.sunset-demo.sunset                       a few seconds ago  sunset-service-discovery   K8SCluster  mesh-env
   service.apic-demo.apic-hybrid-list               an hour ago        service-cli-1605565746103  K8SCluster  mesh-env
   ```

If you see two K8SResources that include the name `sunset-demo` that are scoped to your K8SCluster, then you have successfully configured the RDA to search your Kubernetes cluster for pods and services based on your own configuration in your ResourceDiscoveries.
