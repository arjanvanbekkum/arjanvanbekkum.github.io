---
layout: post
title: "Updating Azure Kubernetes Cluster failed because of a scaling exception."
date: 2020-11-19
summary: "When updating an Azure Kubernetes Cluster, the update failed because of a ScaleVMASAgentPoolFailed exception. What had happened and how can you fix it."
minute: 5
tags: [azure, kubernetes]
---

The other day we were trying to update our Azure Kubernetes Service (AKS) cluster with some new settings. We noticed during the AKS update, one of the nodes became in a `not ready` state. Pods were unable to start, and part of the application became unavailable. 

The error we got was `ScaleVMASAgentPoolFailed`. This seems like an error for scaling an agent failed, strange enough because we were not scaling any nodes. The First thing we did was check the number of CPU's in our Azure subscription. When you update your cluster too, for example, a new version, AKS creates a new Node next to the old ones. After creating these nodes, pods are moved from one note to the new one. Then it created a new node and removed the empty one. If you do not have enough CPU's in you Azure Subscription available, the update will fail because a new node cannot be created

To verify the number of CPU's available, go to the <a href ="https://portal.azure.com">Azure portal</a>, select `Subscriptions`, click on the subscription, in the navigation pane click on `Usage + quotas`, select `Microsoft.Compute` from the second dropdown and the number of CPU's will appear. 

So, unfortunately, this was not our problem. We had plenty of CPU's left in our subscription. 

## Support ticket

We were unable to find what was going on. In the end, we contacted Microsoft support. They came up with the following result

```json
{
  "statusMessage": 
    "{
        "status":"Failed",
        "error":
            { 
              "code":"ResourceOperationFailure",
              "message":"The resource operation completed with terminal provisioning state 'Failed'.",
              "details":[
                  {
                      "code":"ScaleVMASAgentPoolFailed",
                      "message":"We are unable to serve this request due to an internal error, Correlation ID: <GUID>, Operation ID: <GUID, Timestamp: <Timestamp> "
                  }]
            }
    }",
    "eventCategory": "Administrative"
}
```
After restarting the API server and controller manager and using `kubectl describe node <node>` for a more detailed analysis of the failing node they came with the following

|Type          | Status   |  Reason             | Message|
|---|---|---|---|---|---|
|MemoryPressure|Unknown   |  NodeStatusUnknown | Kubelet stopped posting node status.|
|DiskPressure  | Unknown  |  NodeStatusUnknown  | Kubelet stopped posting node status.|
|PIDPressure   | Unknown  |  NodeStatusUnknown  | Kubelet stopped posting node status.|
|Ready         | Unknown  |  NodeStatusUnknown  | Kubelet stopped posting node status.|

Consistent with this, they came up with the following error: `CustomerLinuxNodesNotReady`

>AKS monitoring has detected an issue with a node reporting NotReady in your cluster. We attempt to restart Docker first. If the issue
>persists we send a restart operation to the guest. And finally we restart the VM. Each with a 6 hour window between attempts.

AKS uses a docker container engine on the Virtual Machine to host the, when looking into the docker status and further checking in the `/var/log/messages` and `/var/log/syslog` the following error repeatedly shows:
 
>aks-agentpool-3 health-monitor.sh[8179]: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?<br/>
>aks-agentpool-3 health-monitor.sh[8179]: Container runtime docker failed!<br/>
>aks-agentpool-3 health-monitor.sh[8179]: Failed to kill unit docker.service: No main process to kill

The container runtime docker failed, and it cannot be restarted, leaving the node in a NotReady State. Thus the cluster is stuck in a Failed state.

After some failing attempts to connect to the cluster, we ended up removing the invalid node in the Azure Portal. 
Find the `MC_`  group and locate the VM corresponding to the failed node. Select it in the portal and remove it from the subscription, including the disks and network interface. 

We are now running on one node less, so get the cluster in the correct state with all the nodes we need. We need to perform an update. There is, however, a small problem, it is not possible to update the cluster in the Azure Portal because the state is `Not Ready`.

In this case, we can use the CLI and update the cluster to the same version as we were already running. This will update the cluster and also check the number of nodes. Because we removed one node, AKS will add a new healthy node making sure we are running on all nodes again.
 
```cli
az aks upgrade -n Subscription -g clustername -k 1.16.7
```
After updating the cluster, all nodes and pods are back online.