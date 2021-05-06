# Persistent Volumes, Claims, and Storage Classes

## Introduction
TODO: Brief introduction to the module here....

This module will guide you through the tutorials below.
- Pod storage
- Node storage (static)
- File share storage (static)
- File share storage (dynamic)
- File share volume expansion

## Pre-requisites
 
The following are required to successfully complete this module.

- Azure Subscription (commercial or government)
- An AKS instance resulting from the [Azure Kubernetes Service Workshop](https://docs.microsoft.com/en-us/learn/modules/aks-workshop/).  

## Tutorial: Pod storage
_(10 minutes)_

In this tutorial, you will explore volume storage that is local to a pod.  This kind of volume storage is created when a pod is scheduled to a node and is removed when a pod is removed from the node (for any reason).  If a pod is re-created, even on the same node, any data that was written to the volume previously is lost.  In Kubernetes, this kind of storage/persistence is known as the [`emptyDir` volume](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir).

The following diagram illustrates this kind of storage/persistence in a pod.

TODO: Insert diagram here

This tutorial will use an nginx container running on your cluster to demonstrate the learning objectives.  Before proceeding, review [`01-pod-storage.yaml`](./01-pod-storage.yaml) to familiarize yourself with what it does.


```bash
# Apply deployment to the cluster
kubectl apply -f ./01-pod-storage.yaml

# List the pod that was created
kubectl get pods

# Setup port forwarding so you can browse to the running container.
# Replace '[pod-name]' with the name of your pod.
kubectl port-forward [pod-name] 8080:80

# Open browser to localhost:8080.
# Observe that the response is the standard 'Welcome to nginx!' response.

# Press Ctrl-c to terminate the port forwarding.

# Make a change in the container's file system
# Specifically, change the contents of the 'index.html' file
# that nginx returns when you browse to it.
kubectl exec --stdin --tty [pod-name] -- /bin/bash

# Now you are in the pod's file system.  Perform the following commands:

#   cd /usr/share/nginx/html
#   cat index.html    // Notice it's the HTML you observered previously.
#   echo "Hi there!" > index.html
#   cat index.html    // Notice the new contents of index.html.
#   exit

# Setup port forwarding again
kubectl port-forward [pod-name] 8080:80

# Open browser to localhost:8080.
# Observe the response contains the new text that was added.

# Press Ctrl-c to terminate the port forwarding.

# Delete the pod - kubernetes will re-create it
kubectl delete pod [pod-name]

# List the new pod that was created
kubectl get pods

# Setup port forwarding to the new pod
kubectl port-forward [pod-name] 8080:80

# Open browser to localhost:8080

# Observe the response is the original nginx response, meaning, our change to
# ths filesystem was *not* persisted when the pod was deleted and then re-created.

# Press Ctrl-c to terminate the port forwarding.

# Delete the deployment
kubectl delete -f ./01-pod-storage.yaml
```

### Summary

In this tutorial, you observed that the lifecycle of default storage for a pod follows the lifecycle of the pod.  If a pod is deleted and re-created for any reason, any data written to the filesystem in the pod is lost.

You also observed that you didn't have to add any volume configuration to the configuration for this to work.  However, if you wanted to be explicit about it, you could add the `spec.volumes` and `spec.containers.volumeMounts` as shown in `01-pod-storage-explicit.yaml`.  It would be a good idea to review `01-pod-storage-explicit.yaml` and understand the configuration because it will be used in subsequent tutorials.

## Tutorial: Node Storage (static)
_(15 minutes)_

In this tutorial, you will explore volume storage that is attached to a node.  This kind of volume storage is attached to a node as a data disk.  Because the volume is attached to the node, multiple pods are able to persist and share data.  This also allows a pod to retrieve that data if it gets deleted and re-created on the node.

The following diagram illustrates this kind of storage/persistence on the node.

TODO: Insert diagram here

This tutorial uses the same nginx container you used previously to demonstrate the learning objectives.  Before proceeding, review [`02-node-storage.yaml`](./02-node-storage.yaml) to familiarize yourself with what it does.  In particular, notice the following changes:

- A public load balancer is added.  This is added simply to make it easier for you to interact with the nginx container(s) withouth having to do port-forwarding like you did in the previous tutorial.
- An `azureDisk` volume called "html" was added to `spec.volumes`.
  - Notice the `diskURI` property. You will replace this value with a resource you will create shortly.
- The "html" volume is mounted in the container via the configuration at `spec.containers.volumeMounts`.

### Preliminary cluster coniguration

Before you apply this deployment to your cluster, there are a few items that need to be done first as described here:

- Disable the cluster auto-scaler.  This is so you can manually scale the nodes.  At the end of the module, this will be re-enabled so your cluster is put back to it's original state.
- Scale the cluster to two (2) nodes.
- Create a disk that can be attached to a node in the cluster.

```bash
# Define some environment variables
AKS_WORKSHOP_RG="[your aks-workhsop resource group name]"
AKS_WORKSHOP_CLUSTER="[your aks-workhsop cluster name]"
AKS_WORKSHOP_NODE_RG=$(az aks show --resource-group $AKS_WORKSHOP_RG --name $AKS_WORKSHOP_CLUSTER --query nodeResourceGroup -o tsv)

# Disable the cluster autoscaler.
# This should already be enabled as a result of working through the AKS Workshop (see pre-requisites above).
az aks update --resource-group $AKS_WORKSHOP_RG --name $AKS_WORKSHOP_CLUSTER --disable-cluster-autoscaler

# Scale cluster to two nodes
az aks scale --resource-group $AKS_WORKSHOP_RG --name $AKS_WORKSHOP_CLUSTER --node-count 2 --nodepool-name nodepool1

# Create a disk in the node resource group of your cluster
DISK_RESOURCE_ID=$(az disk create --resource-group $AKS_WORKSHOP_NODE_RG --name pv-static-disk --size-gb 20 --query id --output tsv)

# (optional) Use the Azure portal to see the disk created in the MC_xyz... resource group.
```

### Update the deployment template

In this section, you will update [02-node-storage.yaml](./02-node-storage.yaml) to include the the resource ID of the disk you created in the previous section.

Print out the the resource ID of the disk you created.

```bash
# Print out the resouce ID for the disk created in the previous step.
echo $DISK_RESOURCE_ID
```

Open `02-node-storage.yaml` in an editor and replace "<your disk URI>" with the value from above, and then save your change.

### Work through the tutorial

Now that the changes above are in place, you're ready to start the tutorial.


```bash
# Apply deployment to the cluster
kubectl apply -f ./02-node-storage.yaml

# Wait for the 'volumes-lb' service to get a public IP
# When you see a value for the 'EXTERNAL_IP', press Ctrl-c
kubectl get svc -w

# Browse to the public IP address of the 'volumes-lb' service - observe error 403
# This is because there is not an index.html file for nginx to return.
# You will create one next.

# List the pod that was created
kubectl get pods

# Make a change in the container's file system
# Specifically, create the 'index.html' file that nginx returns when you browse to it.
kubectl exec --stdin --tty [pod-name] -- /bin/bash

# Now you are in the pod's file system.  Perform the following commands:

#   cd /usr/share/nginx/html
#   echo "Hi there!" > index.html
#   exit

# Refresh the page in your browser - observe that "Hi there!" is returned.

# Delete the pod - kubernetes will re-create it
kubectl delete pod [pod-name]

# Refresh the page in your browser - observe that "Hi there!" is returned.
# This is because index.html was created on a disk attached to the node.

# Show the pod *and* the node that the pod is running on.
# Make a mental note about which node your pod is running on.
kubectl get pods --output wide

# Scale out the replicas to 20 so that you get pods spread across the two nodes.
kubectl scale deployment node-storage --replicas=20

# Show all the pods *and* the node that each pod is running on.
kubectl get pods --output wide

# Notice that the pods scheduled to the node your first pod is running on get into a 'Running' state,
# while the pods scheduled to the other node are stuck in the 'ContainerCreating' state.
# This is because Azure disks can only be mounted by a single node/VM at a time.

# Copy the name of the node where the containers are all in a 'Running' state.
# For example, it will be something like 'aks-nodepool1-13512373-vmss000003'

# Drain the node that has all the 'Running' pods.  This makes the node unavailable for pod scheduling.
kubectl drain [node name] --ignore-daemonsets

# Refresh the page in your browser - observe that "Hi there!" is returned.

# Show all the pods *and* the node that each pod is running on.
kubectl get pods --output wide

# Notice that about ~10 pods will be 'Running' on the other node.  The remainder of the
# 20 replicas will be in a 'Pending' state.  This is because the disk has been attached to the
# other node.

# Uncordon the node previously drained, so that pods can get schedule to it again.
kubectl uncordon [node name]

# Show all the pods *and* the node that each pod is running on.
kubectl get pods --output wide

# Notice that all the 'Pending' pods change to 'ContainerCreating'.  Just like before though, they will be
# stuck in this state because the node they are scheduled for cannot attach the volume/disk.

# Refresh the page in your browser - observe that "Hi there!" is returned.
# Notice that you still get a page that says "Hi there!" 

# Delete the deployment
kubectl delete -f ./02-node-storage.yaml

# Delete the Azure disk
# If you get an error indicating the disk is still attached, wait a few seconds and try again.
az disk delete --ids $DISK_RESOURCE_ID --yes
```

### Summary

In this tutorial, you learned how an `azureDisk` volume can be used to provide storage at the node level.  As a result, you were able to observe that when a pod is deleted, data it has written to the volume is not lost.  You also observed a limitation with this kind of volume, which is, it can only be attached to a single node at a time.  This is by design.  However, there is a way enable simultaneous RW access to a volume form multiple nodes.  You will learn how to do that in the next tutorial.

## Static NFS/SMB storage

```bash
# Create a storage account
STG_ACCOUNT_NAME=staticfileshare$RANDOM
az storage account create --resource-group $NODE_RG --name $STG_ACCOUNT_NAME --sku Premium_LRS --kind FileStorage

# Create a file share in the storage account
STG_CONN_STRING=$(az storage account show-connection-string --name $STG_ACCOUNT_NAME --resource-group $NODE_RG --output tsv)
az storage share create --name data --connection-string $STG_CONN_STRING --output tsv

# Create a kubernetes secret to hold the primary key to the storage account
STG_ACCOUNT_KEY=$(az storage account keys list --account-name $STG_ACCOUNT_NAME --query "[0].value" -o tsv)
kubectl create secret generic azure-storage --from-literal=azurestorageaccountname=$STG_ACCOUNT_NAME --from-literal=azurestorageaccountkey=$STG_ACCOUNT_KEY

# Apply deployment
kubectl apply -f ./nginx-deployment-03.yaml

# Wait for public IP address
# Open browser to public IP address - observe the response, which is the hostname
# from the node the pod is running on.

# Scale out the replicas to 20 so that you get pods spread across the two nodes.
kubectl scale deployment nginx-deployment-03 --replicas=20

# Show all the pods *and* the node that each pod is running on.
# Notice that this time all 20 pods are running.  That is because the volume mount
# is mounting an Azure File Share, which supports SMB 3.0 and muliple R/W nodes simultaneously.
kubectl get pods --output wide

# Open browser to public IP address - observe the response, now includes the hostnames
# from all the pods running across the two nodes.
# This shows that the nodes were not only able to read the file simultanously, but also write to it.

# Using the Azure Portal, view the index.html file in the storage account.

# Delete the deployment
kubectl delete -f ./nginx-deployment-03.yaml

```

## Static NFS/SMB storage - explicit definitions

```bash
# Apply deployment
kubectl apply -f ./nginx-deployment-04.yaml

# Wait for public IP address
# Open browser to public IP address - observe the response, which is the hostname
# from the node the pod is running on.

# Scale out the replicas to 20 so that you get pods spread across the two nodes.
kubectl scale deployment nginx-deployment-03 --replicas=20

# Show all the pods *and* the node that each pod is running on.
# Notice that this time all 20 pods are running.  That is because the volume mount
# is mounting an Azure File Share, which supports SMB 3.0 and muliple R/W nodes simultaneously.
kubectl get pods --output wide

# Show the pv and pvc that was created in the Azure portal.
# This is a good time to introduce Storage classes and to point out the 4 default storage classes AKS provides.
# The reason this is important is because we can use a storage class to dynamically create PV/PVC's, which is
# the recommended best practice and what we will see in the next section.

# Delete the deployment
kubectl delete -f ./nginx-deployment-03.yaml

# Delete the secret
kubectl delete secret azure-storage

# Delete the storage account
az storage account delete --name $STG_ACCOUNT_NAME --resource-group $NODE_RG --yes
```

## Dynamic NFS/SMB storage

To do... 

```bash
# Apply deployment
kubectl apply -f ./nginx-deployment-05.yaml

# Wait for public IP address
# Notice also the extra time it takes for the pod to get to a running state.  This is because
# - The storage account is getting created
# - The share is created
# - The PV and PVC are getting generated
# - The pod then can mount
# Open browser to public IP address - observe the response, which is the hostname
# from the node the pod is running on.

# Scale out the replicas to 20 so that you get pods spread across the two nodes.
kubectl scale deployment nginx-deployment-05 --replicas=20

# Delete the deployment
kubectl delete -f ./nginx-deployment-05.yaml

# Show that the pvc is deleted.
# Also, notice that the pv that was dynamically created is deleted.
# And finally, notice that the storage account is *not* deleted and the data in the share is still there.
# - explain why...
# - explain what you would change if you wanted the storage account deleted too.
```

## Increase capacity in PVC

To do...

```bash
# Apply deployment
kubectl apply -f ./nginx-deployment-06.yaml

# Show how the container just creates a 500MB file in the file share.

# Scale out the replicas to 11 and observe the files getting created in the file share
kubectl scale deployment nginx-deployment-06 --replicas=11

# Get a list of all the pods running
k get pods

# observe that one of the pods is going to fail.  This is because the file share ran out of space.
# show the error
k logs [failed pod name]

# Need to expand the volume, which we can do because the azurefile storage class supports volume expansion by default.
# Show this in the yaml

# edit the pvc to increase the size from 5GB to 8GB
k edit pvc file-storage-claim
# This will open VIM editor
# move cursor on top of 5 in the spec.  Type 'r8', then ':wq' to save the change.
# Now the pvc has been expanded to 8GB

# Delete the failed pod 
k delete pod [failed pod name]

# Notice that Kubernetes scheduled a new pod so your replicaset is at 11.  This time, it doesn't fail.
# observe the new file created in the share

# Scale out the replicas to 14.
kubectl scale deployment nginx-deployment-06 --replicas=14

# Observe the files created in the share and all the pods are 'Running'.  This is because we haven't exceeded the
# new capacity of 8GB.

# Delete the deployment
kubectl delete -f ./nginx-deployment-05.yaml

# Show that the pvc is deleted.
# Also, notice that the pv that was dynamically created is deleted.
# And finally, notice that the storage account is *not* deleted and the data in the share is still there.
# - explain why...
# - explain what you would change if you wanted the storage account deleted too.
```

## Wrap up

To do...

```bash

# Re-enable the cluster autoscaler
az aks update --resource-group fruit-smoothies-rg --name fruit-smoothies-6485-aks --enable-cluster-autoscaler --min-count 1 --max-count 3

# (optional) Delete the storage accounts that were dynamically created
```
