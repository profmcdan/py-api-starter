## Build docker image 

```docker build . -t profmcdan/django-starter-api:v0 -f docker/prod/Dockerfile```

## if using docker-desktop, install and view the kube dashboard 
https://andrewlock.net/running-kubernetes-and-the-dashboard-with-docker-desktop/
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml
kubectl patch deployment kubernetes-dashboard -n kubernetes-dashboard --type 'json' -p '[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--enable-skip-login"}]'
kubectl proxy

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.4.2/components.yaml
kubectl patch deployment metrics-server -n kube-system --type 'json' -p '[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

```

Open the dashboard here

http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login


## Run Deployment 
```
kubectl apply -f deployment-api.yaml
kubectl get deployments
kubectl get pods 
kubectl delete -f deployment-api.yaml
```
## Run Service 
```kubectl delete -f deployment-api.yaml```


