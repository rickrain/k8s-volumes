# Persistent Volumes & Persistent Volume Claims

## Container/Pod storage (emptyDir)

```bash
# Apply deployment
kubectl apply -f ./nginx-deployment-01.yaml

# Setup port forwarding
kubectl port-forward [pod-name] 8080:80

# Open browser to localhost:8080

# Make a change in the container's file system
kubectl exec --stdin --tty [pod-name] -- /bin/bash

# Change the index.html file in the container
# cd /usr/share/nginx/html
# echo "Hi there!" > ./index.html
# exit

# Setup port forwarding
kubectl port-forward [pod-name] 8080:80

# Open browser to localhost:8080
# Observe the response is different

# Kill the pod - kubernetes will re-create it
kubectl delete pod [pod-name]

# Setup port forwarding
kubectl port-forward [pod-name] 8080:80

# Open browser to localhost:8080
# Observe the response is the original nginx response.
# Our change was not persisted.

# Delete the deployment
kubectl delete -f ./nginx-deployment-01.yaml
```


## Static Node Storage

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
az disk delete --ids $DISK_RESOURCE_ID --yes
```



## NFS storage


https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/

```bash
# Disable the cluster autoscaler
az aks update --resource-group fruit-smoothies-rg --name fruit-smoothies-6485-aks --disable-cluster-autoscaler

# Scale cluster down to a single node
az aks scale --resource-group fruit-smoothies-rg --name fruit-smoothies-6485-aks --node-count 1 --nodepool-name nodepool1

# Apply deployment
kubectl apply -f ./nginx-deployment-02.yaml

# Setup port forwarding
kubectl port-forward [pod-name] 8080:80

# Open browser to localhost:8080
# Notice that you get a 403 forbidden error.  That's because there is nothing for nginx to respond with.

# Create an index.html file in the mount path location.
kubectl exec --stdin --tty [pod-name] -- /bin/bash

# Change the index.html file in the container
# cd /usr/share/nginx/html
# echo "Hi there!" > ./index.html
# exit

# Setup port forwarding
kubectl port-forward [pod-name] 8080:80

# Open browser to localhost:8080
# Notice that you get a page that says "Hi there!"

# Kill the pod - kubernetes will re-create it
kubectl delete pod [pod-name]

# Setup port forwarding
kubectl port-forward [pod-name] 8080:80

# Open browser to localhost:8080
# Notice that you still get a page that says "Hi there!" 
# This is because index.html was created on the nodes files system.

# Remove deployment
kubectl delete -f ./nginx-deployment-02.yaml

# Re-enable the cluster autoscaler
az aks update --resource-group fruit-smoothies-rg --name fruit-smoothies-6485-aks --enable-cluster-autoscaler --min-count 1 --max-count 3

```
