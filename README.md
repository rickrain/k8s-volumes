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
