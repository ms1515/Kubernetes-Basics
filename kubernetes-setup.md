<h1> Set up a local Knative cluster </h1>

<h1> 1. Install Docker Desktop </h1>

Install Docker Desktop: https://docs.docker.com/get-docker/

Once you have followed the installation process of Docker for your platform, ensure that your Docker engine is running, you will see a green icon on the bottom left of Docker Desktop.

![image](uploads/cc8e50355e5fb3f9b6896977a6e1c490/image.png)

<h1> 2. Enable Kubernetes(K8s) </h1>

From Docker Desktop, you can enable Kubernetes (K8s) to run on your local machine by going to settings, Kubernetes on the left and then clicking on the Enable Kubernetes checkbox, as seen below:

![Screenshot_2022-01-20_at_14.35.11](uploads/de0e0b76da3a2696997ac4919f4c83db/Screenshot_2022-01-20_at_14.35.11.png)

you should then install kubectl - a k8s tool: https://kubernetes.io/docs/tasks/tools/#kubectl

<h1> 3. Install Istio </h1>

Next we will install Istio, think of Istio as a means of traffic managment and routing for our services.

You can either install istio from the cmd line or download the binary from its website.

Docs: https://istio.io/latest/docs/setup/getting-started/#download

For the cmd line option (using the docs), run this command to download the binary: 

```
curl -L https://istio.io/downloadIstio | sh -
```

then after downloading the binary: 

```
cd istio-1.12.2
```

then set the binary in your PATH by running: 

```
export PATH=$PWD/bin:$PATH

```
then finally install using: 

```
istioctl install

```
<h1> 4. Install Knative Serving </h1>

Ton install Knative serving, run this command: 

```
kubectl apply --filename https://github.com/knative/serving/releases/download/knative-v1.1.1/serving-crds.yaml

```
and then: 

```
kubectl apply --filename https://github.com/knative/serving/releases/download/knative-v1.1.1/serving-core.yaml

```
Next we need to add the knative istio connection:

```
kubectl apply --filename https://github.com/knative-sandbox/net-istio/releases/download/knative-v1.0.0/release.yaml

```
and now: 

```
kubectl --namespace istio-system get service istio-ingressgateway

```
<h1> 5. Install Knative CLI (Kn CLI) </h1>

you can get an overview on how to install kn CLI here: https://knative.dev/docs/install/client/install-kn/

<h1> 6. Running your first example service </h1>

Lets install an example service into your new cluster, let's use the newly install Kn CLI:

```
kn service create helloworld-go --image gcr.io/knative-samples/helloworld-go --env TARGET="Go Sample v1"

```
Let's check that it was created by using kn service describe to check:

```
kn service describe helloworld-go

```
you should get a response like this:
![image](uploads/1a664e01e6338ef844ae431e42e6a4aa/image.png)

Now let's use a curl command to test it is working, remember this is the first request so it will be slower on the first request as it deploys your service:

```
curl -H "Host: helloworld-go.default.example.com" http://localhost:80

```
If all is well, you shuold see "Hello Go Sample v1!"

to remove the service when you are happy, just run:

```
kn service delete helloworld-go

```
<h1> 7. Set up Authentication Secrets </h1>

Set up Authentication Secrets to pull images from our private gitlab container registry by following **one** of the guides here:

- Knative guide: https://knative.dev/docs/serving/deploying-from-private-registry/
- Kubernetes guide: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/

<h3> TLDR; </h3>

Gitlab Auth Server: https://registry.gitlab.com

1. The command to create a secret: 

```
kubectl create secret docker-registry gitlab-auth \
  --docker-server=https://registry.gitlab.com \
  --docker-email=YOUR-GITLAB-EMAIL \
  --docker-username=YOUR-GITLAB-USERNAME \
  --docker-password=YOUR-GITLAB-DEPLOY-TOKEN or YOUR-GITLAB-ACCESS-TOKEN
```

then apply that secret to your default service account for the namespace to use as an imagePullSecret:

```
kubectl patch serviceaccount default -p "{\"imagePullSecrets\": [{\"name\": \"gitlab-auth\"}]}"

```
This means that any service that gets created in the default service account will use that as the imagePullSecret.

<h1> 8. Using Secrets within the Application </h1>

For a general guide on injecting your secrets/configurations into your applications, check out the following docs: https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/

<h2> 8.1 Springboot Application </h2>

There are two files in a Springboot project where you have to define authentication secrets and other configurations for repositories and databases: build.gradle and application.yml files.

This section will explain how to set up your local machine to work with this project without pushing up your secrets to git repo.

<h3> 8.1.1 build.gradle file </h3>

1. Create an `environment.properties` file in the root of the project with these properties and their respective values:

```
registry_username=YOUR_REGISTRY_USERNAME
registry_password=YOUR_REGISTRY_ACCESS_TOKEN or YOUR_REGISTRY_DEPLOY_TOKEN
```

Since this filename is included in .gitignore, it will not be added to git.

2. This file is loaded and referenced in `build.gradle` as follows:

```
def environmentProperties = new Properties()
def environmentPropertiesFile = rootProject.file('environment.properties')
if (environmentPropertiesFile.exists()) {
    environmentProperties.load(new FileInputStream(environmentPropertiesFile))
}
....
jib {
	....
		auth {
			username = environmentProperties["registry_username"]
			password = environmentProperties["registry_password"]
		}
	}
}
```

**Alternatively,** you can add these values as environment variables on your machine and reference them inside 'build.gradle' using 'System.getenv("YOUR_VARIABLE_NAME").toString()'.

<h3> 8.1.2 application.yml file </h3>

1. Create your secrets for your database connection: 

```
kubectl create secret generic mongodb-credentials \                          
--from-literal=MONGO_DB_NAME=YOUR_DATABASE_NAME \
--from-literal=MONGO_DB_USERNAME=YOUR_DATABASE_USERNAME \
--from-literal=MONGO_DB_PASSWORD=YOUR_DATABASE_PASSWORD
```

2. This secret is referenced in `knative-service.yaml` file for exposing it to the application container to use:

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

3. The application references the keys of this secret in the `application.yml` file for mongo:

```
spring:
    application:
        name: application-name
    data:
        mongodb:
            uri: "mongodb+srv://${MONGO_DB_USERNAME}:${MONGO_DB_PASSWORD}@cluster0.g1nor.mongodb.net/${MONGO_DB_NAME}?retryWrites=true&w=majority"
   
```

**Note:** The variable names for your secret have to be unique across secrets to avoid mismatch of values when referencing.

<h1> 9. Simplify deployment with Scaffold</h1>

**Installation Instructions:** https://skaffold.dev/docs/install/
**Documentation Overview** → http://bit.ly/skaffold-overview 
**Cloud Blog** → http://bit.ly/blog-Skaffold 
**GitHub** → http://bit.ly/GitHub-Skaffold 


Scaffold will scan your application code base and find docker files, Kubernetes manifests (for pods, deployments, services) and create a scaffold.yaml that points to these configurations, allowing you to run your application from one scaffold file.

A skaffold yaml file can be generated from your project by running the following cmd at the root of your project:

```
skaffold init -f skaffold.yaml
```

You can then run your application using:

```
skaffold run 
```

And scaffold will take care of building your docker images, deploying them to container registries, and running your application using the Kubernetes manifests (yaml files for pods, services, deployments) in your k8 cluster.

### Skaffold with multiple apps in separate directories (front-end, back-end) ###

This time you will have to specify what docker files to use for each build artifact (build image):

![image](images/skaffold-multiple-apps.png)

Skaffold is smart enough to detect which directory/app had changes and will only rebuild that docker image again, and will only update the deployement associated with that  build artifact (docker image).

**For example, These are the Kubernetes manifests for an example application:**

![image](uploads/186285aee9e43706c825ac508bc1a8d8/image.png)

**And this is the resulting skaffold yaml file for the above configuration after running 'skaffold init':**

![image](uploads/896719152f7bf025093dd5a826f672db/image.png)

The best thing about skaffold is that it will monitor changes in your code base using its file watcher and rebuild and deploy your changes to your cluster.

<h1> 10. Debugging </h1>

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
