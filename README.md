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

What I have found is that by doing istio install, which doesn't work but still installs the istio core, I can get an external IP.

