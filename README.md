# Azure Container Instances Connector for Kubernetes (experimental)

The Azure Container Instances Connector for Kubernetes allows Kubernetes clusters to deploy Azure Container Instances.

This enables on-demand and nearly instantaneous container compute, orchestrated by Kubernetes, without having VM infrastructure to manage and while still leveraging the portable Kubernetes API. This will allow you to utilize both VMs and container instances simultaneously in the same Kubernetes cluster, giving you the best of both worlds.

Please note this software is experimental and should not be used for anything resembling a production workload.

## How does it Work

The ACI Connector roughly mimics the [Kubelet](https://kubernetes.io/docs/admin/kubelet/) interface by:

- Registering into the Kubernetes data plane as a `Node` with unlimited capacity
- Dispatching scheduled `Pods` to Azure Container Instances instead of a VM-based container engine

Once the connector is registered as a node named `aci-connector`, you can use `nodeName: aci-connector` in your Pod spec to run the Pod via Azure Container Instances.  Pods without this node name will continue to be scheduled normally.  See below for instructions on how to use use the ACI Connector with the Kubernetes scheduler [via taints and tolerations](#using-the-kubernetes-scheduler).

![ACI Connector for Kubernetes GIF](https://github.com/Azure/aci-connector-k8s/blob/master/gifs/aci-connector-k8s.gif)

## Requirements

 1. A working `az` command-line client - [Install azure cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) 
 2. A Kubernetes cluster with a working `kubectl` - [Set up a Kubernetes cluster on Azure](https://docs.microsoft.com/en-us/azure/container-service/kubernetes/container-service-kubernetes-walkthrough)

## Current Features
In addition to the provided examples directory, the following Kubernetes features are currently supported when defined within a Kubernetes Pod manifest. This list is subject to change as we improve the aci-connector.
* Environment Variables
* Commands

## Limitations
The following Kubernetes features are not currently supported as part of the aci-connector.
* ConfigMaps
* Secrets
* ImagePullSecrets
* ServiceAccounts
* Volumes
* kubectl logs
* kubectl exec

## Quickstart

1. Edit `examples/aci-connector.yaml` and supply environment variables
2. Run the ACI Connector with `kubectl create -f examples/aci-connector.yaml`
3. Wait for `kubectl get nodes` to display the `aci-connector` node
4. Run an NGINX pod via ACI using `kubectl create -f examples/nginx-pod.yaml`
5. Access the NGINX pod via its public address

## Usage

### Create a Resource Group

The ACI Connector will create each container instance in a specified resource group.  You can create a new resource group with:

```console
$ az group create -n aci-test -l westus
{
  "id": "/subscriptions/<subscriptionId>/resourceGroups/aci-test",
  "location": "westus",
  "managedBy": null,
  "name": "aci-test",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null
}
```

Edit the `examples/aci-connector.yaml` and put the name of the resource group into the `ACI_RESOURCE_GROUP` environment variable.

### Create a Service Principal

A service principal is required to allow the ACI Connector to create resources in your Azure subscription.  You can create one using the `az` CLI using the instructions below.

Find your `subscriptionId` with the `az` CLI:

```console
$ az account list -o table
Name                                             CloudName    SubscriptionId                        State    IsDefault
-----------------------------------------------  -----------  ------------------------------------  -------  -----------
Pay-As-You-Go                                    AzureCloud   12345678-9012-3456-7890-123456789012  Enabled  True
```

Use `az` to create a Service Principal that can perform operations on your subscription:

```console
$ az ad sp create-for-rbac --role=Contributor --scopes /subscriptions/<subscriptionId>/
{
  "appId": "<redacted>",
  "displayName": "azure-cli-2017-07-19-19-13-19",
  "name": "http://azure-cli-2017-07-19-19-13-19",
  "password": "<redacted>",
  "tenant": "<redacted>"
}
```

Edit the `examples/aci-connector.yaml` and input environment variables using the values above:

- AZURE_CLIENT_ID: insert `appId`
- AZURE_CLIENT_KEY: insert `password`
- AZURE_TENANT_ID: insert `tenant`
- AZURE_SUBSCRIPTION_ID: insert `subscriptionId`

### Confirm Microsoft.ContainerInstance provider is registered

```console
$ az provider list -o table | grep ContainerInstance
Microsoft.ContainerInstance             NotRegistered
```
If it is not registered, register it by running the following command.
```console
$ az provider register -n Microsoft.ContainerInstance
$ az provider list -o table | grep ContainerInstance
Microsoft.ContainerInstance             Registered
```


### Install the ACI Connector

```console
$ kubectl create -f examples/aci-connector.yaml 
deployment "aci-connector" created

$ kubectl get nodes -w
NAME                        STATUS                     AGE       VERSION
aci-connector               Ready                      3s        1.6.6
k8s-agentpool1-31868821-0   Ready                      5d        v1.7.0
k8s-agentpool1-31868821-1   Ready                      5d        v1.7.0
k8s-agentpool1-31868821-2   Ready                      5d        v1.7.0
k8s-master-31868821-0       Ready,SchedulingDisabled   5d        v1.7.0
```

### Install the ACI Connector with Helm (optional)

First, make sure to fill the values in the values.yaml file located in the /charts/aci-connector directory.

You can then install the chart:

```console
$ helm install --name my-release ./charts/aci-connector
```

There's also the option to set the values from the command line. This will override any values in the values.yaml file:

```console
$ helm install --name my-release --set env.azureClientId=YOUR-AZURECLIENTID,env.azureClientKey=YOUR-AZURECLIENTKEY,env.azureTenantId=YOUR-AZURETENANTID,env.azureSubscriptionId=YOUR-AZURESUBSCRIPTIONID,env.aciResourceGroup=YOUR-ACIRESOURCEGROUP,env.aciRegion=YOUR-ACI-REGION ./charts/aci-connector
```

### Install the NGINX example

```console
$ kubectl create -f examples/nginx-pod.yaml 
pod "nginx" created

$ kubectl get po -w -o wide
NAME          READY     STATUS    RESTARTS   AGE       IP             NODE
aci-connector-3396840456-v75q2  1/1       Running   0          44s       10.244.2.21    k8s-agentpool1-31868821-2
nginx         1/1       Running   0          31s       13.88.27.150   aci-connector
```

Note the pod is scheduled on the `aci-connector` node.  It should now be accessible at the public IP listed.


### Using the Kubernetes scheduler

The example in [nginx-pod](examples/nginx-pod.yaml) hard codes the node name, but you can also use the Kubernetes scheduler.

The virtual `aci` node, has a taint (`azure.com/aci`) with a default effect
of `NoSchedule`. This means that by default Pods will not schedule onto
the `aci` node unless they are explicitly placed there.

However, if you create a Pod that _tolerates_ this taint, it can be scheduled
to the `aci` node by the Kubernetes scheduler.

Here is an [example](examples/nginx-pod-tolerations.yaml) of Pod with this
toleration.

To use this Pod, you can simply:

```sh
$ kubectl create -f examples/nginx-pod-tolerations.yaml
```

Note that if you have other nodes in your cluster then this Pod may not
necessarily schedule onto the Azure Container Instances.

To force a Pod onto Azure Container Instances, you can either explicitly specify the NodeName as in the first example, or you can delete all of the other nodes in your cluster using `kubectl delete nodes <node-name>`. A third option is to fill your cluster with other workloads, then the scheduler will be obligated to schedule work to the Azure Container Instance API.

## Using Canary builds

"Canary" builds are versions of the connector that are built periodically from the latest master branch. They are not official releases, and may not be stable. However, they offer the opportunity to test the cutting edge features.

To use the latest canary release you can patch the aci-connector deployment to update the container tag using the following command:
```console
$ kubectl set image deploy/aci-connector aci-connector=microsoft/aci-connector-k8s:canary
```

## Development Instructions

### Local Development

```console
<edit source>
$ make clean
$ make build
$ node connector.js
```

### Docker Development

```console
make docker-build
docker tag <local-image> <remote-image>
docker push <remote-image>
```

Then edit `examples/aci-connector.yaml` to point to the `remote-image`.

# Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.microsoft.com.

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.
