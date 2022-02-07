# Kubernetes-Basics

## Private Repositories ##

- Gitlab auth server: https://registry.gitlab.com
- Docker auth server: https://index.docker.io/v1/

## Kubernetes Secrets ##

General guide on injecting secrets into applications: https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/

- Create repository secret: 
```
kubectl create secret docker-registry gitlab-auth \
  --docker-server=https://registry.gitlab.com \
  --docker-email=YOUR-GITLAB-EMAIL \
  --docker-username=YOUR-GITLAB-USERNAME \
  --docker-password=YOUR-GITLAB-DEPLOY-TOKEN or YOUR-GITLAB-ACCESS-TOKEN
```
- Create generic secret: 
```
kubectl create secret generic mongodb-credentials \                          
--from-literal=MONGO_DB_NAME=YOUR_DATABASE_NAME \
--from-literal=MONGO_DB_USERNAME=YOUR_DATABASE_USERNAME \
--from-literal=MONGO_DB_PASSWORD=YOUR_DATABASE_PASSWORD
```

## Istio Install ##

Istio can ahve problems installing. Can get stuck on:

- Processing resources for Istiod. Waiting for Deployment/istio-system/istiod 

1. So to debug why its happening, inspect the istio pods in istio namespace:
```
kubectl get pods -n istio-system
```
2. And then examine the pod for errors:
```
kubectl describe pod YOUR_ISTIO_POD_NAME -n istio-system
```

3. For me it was giving me insufficient memory (RAM) error which makes sense as there are so many k8 related containers being run on docker, and I have only provisioned 2GB of RAM to Docker.

So I increased the RAM to 4GB for Docker and istio installed properly.

4. So being able to **debug your pods** in the right namespace is crucial.


## Gitlab Auth Issues ##

Been having secrets issues.

## Debugging ##

After your service/deployment is created, it is important to be able to know what is happening at the resource level. Some commonly used commands to quickly debug are the following:

- Get all your pods: 

```
kubectl get pods
```
- See detailed description of what's happening in your pod: 
```
kubectl describe pod YOUR_POD_NAME
```
- See Logs for all containers in your pod: 

```
kubectl logs YOUR_POD_NAME --all-containers=true
```
- Get pods in a specific namespace
```
kubectl get pods -n NAMESPACE
```
- Describe a Pod in a certain Namespace
```
kubectl describe pod YOUR_ISTIO_POD_NAME -n istio-system
```


