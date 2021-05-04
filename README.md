# Persistent Volumes & Persistent Volume Claims

## Container/Pod storage

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

# Remove deployment
kubectl apply -f ./nginx-deployment-01.yaml
```

## Node storage

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
kubectl apply -f ./nginx-deployment-02.yaml

# Re-enable the cluster autoscaler
az aks update --resource-group fruit-smoothies-rg --name fruit-smoothies-6485-aks --enable-cluster-autoscaler --min-count 1 --max-count 3

```
