# Deploy to AKS

## Create cluster

```bash
az aks create --resource-group nakise-dev --name nakise-cluster --node-count 2 --generate-ssh-keys
```

Connect to cluster using kubectl

```bash
az aks get-credentials --resource-group nakise-dev --name nakise-cluster
kubectl get nodes
```

Database:

