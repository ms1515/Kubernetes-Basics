# Kubernetes-Basics

Knative-serving installation guide: https://knative.dev/docs/install/serving/install-serving-with-yaml/#configure-dns

## Private Repositories ##

- Gitlab auth server: https://registry.gitlab.com
- Docker auth server: https://index.docker.io/v1/

## Kubernetes Secrets ##

Setting up secrets: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
General guide on injecting secrets into applications: https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/

- Create repository secret: 
```
kubectl create secret docker-registry registry-auth \
  --docker-server=https://registry.gitlab.com \
  --docker-email=YOUR-GITLAB-EMAIL \
  --docker-username=YOUR-GITLAB-USERNAME \
  --docker-password=YOUR-GITLAB-DEPLOY-TOKEN or YOUR-GITLAB-ACCESS-TOKEN
```
- And decode the secret:
```
kubectl get secret gitlab-auth --output="jsonpath={.data.\.dockerconfigjson}" | base64 --decode
```
- Create generic secret: 
```
kubectl create secret generic generic-credentials \                          
--from-literal=MONGO_DB_NAME=YOUR_DATABASE_NAME \
--from-literal=MONGO_DB_USERNAME=YOUR_DATABASE_USERNAME \
--from-literal=MONGO_DB_PASSWORD=YOUR_DATABASE_PASSWORD
```
- Refer these secrets in your service.yaml

```
spec:
      containers:
        - image: YOUR_IMAGE_REPO_URL
          imagePullPolicy: Always
          ports:
            - containerPort: 8096
          readinessProbe:
            initialDelaySeconds: 15
          envFrom:
            - secretRef:
                name: mongodb-credentials
       imagePullSecrets:
        - name: gitlab-auth
```
- (Or) Patch your default serviceaccount with that secret instead of specifying it in service.yaml
```
kubectl patch serviceaccount default -p "{\"imagePullSecrets\": [{\"name\": \"container-registry\"}]}"
```


## Things not becoming ready ##

It can be due to resource issues: RAM, CPU, Storage etc. For example

### Istio can have problems installing. Can get stuck on ###

```
Processing resources for Istiod. Waiting for Deployment/istio-system/istiod 
```

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

### Some Istio pods not being ready ##

like istio-ingressgateway:
```
NAME                                   READY   STATUS    RESTARTS   AGE
istio-ingressgateway-b899b7b79-klb5p   0/1     Pending   0          57d
istio-ingressgateway-b899b7b79-m45lj   0/1     Pending   0          57d
istio-ingressgateway-b899b7b79-x6kmh   1/1     Running   0          57d
istiod-d845fbcfd-8xhsj                 1/1     Running   1          57d
istiod-d845fbcfd-hjk6p                 1/1     Running   1          57d
istiod-d845fbcfd-ss7zt                 1/1     Running   0          57d
```

So check the pod's (the one with error or pending) description: 
```
kubectl describe pod istio-ingressgateway-... -n istio-system
```
and you will see the reason:
```
Events:
  Type     Reason            Age                       From               Message
  ----     ------            ----                      ----               -------
  Warning  FailedScheduling  4m25s (x5375 over 3d19h)  default-scheduler  0/3 nodes are available: 1 Insufficient memory, 1 node(s) were unschedulable, 2 Insufficient cpu.
```

### MagicDNS not working ###
Again check your resources.

### Istio External IP not available ###

Useful to check out the istio ingress service:
  ```
  kubectl get svc istio-ingressgateway -n istio-system
  ```
  OR
  ```
  kubectl get service istio-ingressgateway -n istio-system
  ```
  - Describe the service to get details
  ```
  kubectl describe svc istio-ingressgateway -n istio-system
  ```
  - Check the pods for istio-system namespace
  ```
  kubectl get pods -n istio-system
  ```
  - And then check that pod
  ```
  kubectl describe pod POD_NAME -n istio-system
  ```

## Auth Issues ##

### Gitlab Private Repos ###

Getting 403/Access Denied/Forbidden errors on pulling images, repos, packages for private projects.

- Check your secret is correct: even an extra space or a newline character introduced during installation can cause this without you even knowing it.
- Copy the token exactly as is, from start to finish. Don't select the entire line as it may introduce newline or space/tab characters
### Maybe your installed version of k8 or knative-serving has a problem with pulling private images 

## Check your comtext ## 

- what context of kubernetes are you in: Docker Desktop, Kind (kubernetes in Docker) etc

![kubernetes contexts image](images/k8Context.png)

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
- For Exampl, get knative-serving namespace pods:
```
kubectl get pods -n knative-serving
```
- Describe a Pod in a certain Namespace
```
kubectl describe pod YOUR_ISTIO_POD_NAME -n istio-system
```


