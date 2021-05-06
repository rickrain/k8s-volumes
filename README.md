# Persistent Volumes, Claims, and Storage Classes

TODO: Introduction here

This module will guide you through the tutorials below.
- Pod storage
- Node storage (static)
- File share storage (static)
- File share storage (dynamic)
- File share volume expansion

## Tutorial: Pod storage (10 minutes)

In this tutorial, you will explore volume storage that is local to a pod.  This kind of volume storage is created when a pod is scheduled to a node and is removed when a pod is removed from the node (for any reason).  If a pod is re-created, even on the same node, any data that was written to the volume previously is lost.  In Kubernetes, this kind of storage/persistence is known as the [`emptyDir` volume](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir).

The following diagram illustrates this kind of storage/persistence in a pod.

TODO: Insert diagram here

This tutorial will use an nginx container running on your cluster to demonstrate the learning objective.  Before proceeding, review [`01-pod-storage.yaml`](./01-pod-storage.yaml) to familiarize yourself with what it does.


```bash
# Apply deployment to the cluster.
kubectl apply -f ./01-pod-storage.yaml

# List the pod that was created.
kubectl get pods

# Setup port forwarding so we can browse to the running container.  Replace '[pod-name]' with the name of your pod.
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

# Kill the pod - kubernetes will re-create it since this
# is part of a 'Deployment'.
kubectl delete pod [pod-name]

# List the new pod that was created.
kubectl get pods

# Setup port forwarding to the new pod
kubectl port-forward [pod-name] 8080:80

# Open browser to localhost:8080
# Observe the response is the original nginx response.
# Our change was *not* persisted when the pod was
# deleted and then re-created.

# Press Ctrl-c to terminate the port forwarding.

# Delete the deployment
kubectl delete -f ./01-pod-storage.yaml
```

### Summary

In this tutorial, you observed that the lifecycle of default storage for a pod follows the lifecycle of the pod.  If a pod is deleted and re-created for any reason, any data written to the filesystem in the pod is lost.

You also observed that you didn't have to add any volume configuration to the configuration for this to work.  However, if you wanted to be explicit about it, you could add the `spec.volumes` and `spec.containers.volumeMounts` as shown in `01-pod-storage-explicit.yaml`.  It would be a good idea to review `01-pod-storage-explicit.yaml` and understand the configuration because it will be used in subsequent tutorials.

## Static Node Storage (local)

```bash
# Disable the cluster autoscaler
az aks update --resource-group fruit-smoothies-rg --name fruit-smoothies-6485-aks --disable-cluster-autoscaler

# Scale cluster to two nodes
az aks scale --resource-group fruit-smoothies-rg --name fruit-smoothies-6485-aks --node-count 2 --nodepool-name nodepool1

# Create a disk in the node resource group of your cluster 
NODE_RG=$(az aks show --resource-group fruit-smoothies-rg --name fruit-smoothies-6485-aks --query nodeResourceGroup -o tsv)
DISK_RESOURCE_ID=$(az disk create --resource-group $NODE_RG --name pv-static-disk --size-gb 20 --query id --output tsv)

# Apply deployment
kubectl apply -f ./nginx-deployment-02.yaml

# Wait for public IP address
# Browse to public IP address - observe error 403

# Create the index.html file that nginx expects to return from it's file system
kubectl exec --stdin --tty [pod-name] -- /bin/bash

# cd /usr/share/nginx/html
# echo "Hi there!" > index.html
# exit

# Wait for public IP address
# Open browser to public IP address - observe that "Hi there is returned.

# Kill the pod - kubernetes will re-create it
kubectl delete pod [pod-name]

# Open browser to public IP address
# Notice that you still get a page that says "Hi there!" 
# This is because index.html was created on disk mounted by the node.

# Show the pod *and* the node that the pod is running on.
# Make a mental note about which node your pod is on.
kubectl get pods --output wide

# Scale out the replicas to 20 so that you get pods spread across the two nodes.
kubectl scale deployment nginx-deployment-02 --replicas=20

# Show all the pods *and* the node that each pod is running on.
# Notice that the pods scheduled to the node your first pod eventually get into the 'Running' state
# and the pods scheduled to the other node are stuck in the 'ContainerCreating' state
# This is because Azure disks can only be mounted by a single node.
kubectl get pods --output wide

# Drain the node that has all the 'Running' pods.  This makes the node unavailable for pod scheduling.
kubectl drain aks-nodepool1-13512373-vmss000003 --ignore-daemonsets

# Show all the pods *and* the node that each pod is running on.
# Notice that (eventually), about ~10 pods will be 'Running' on the other node.  The remainder of the
# 20 replicas will be in a 'Pending' state.
kubectl get pods --output wide

# Uncordon the node previously drained, so that pods can get schedule to it again.
# Notice that all the 'Pending' pods change to 'ContainerCreating'.  Just like before though, they will be
# stuck in this state because the node they are scheduled for cannot mount the volume/disk.
kubectl uncordon aks-nodepool1-13512373-vmss000003

# Open browser to public IP address
# Notice that you still get a page that says "Hi there!" 

# Delete the deployment
kubectl delete -f ./nginx-deployment-02.yaml

# Delete the Azure disk
# If you get an error indicating the disk is still attached, wait a few seconds and try again.
az disk delete --ids $DISK_RESOURCE_ID --yes
```
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
