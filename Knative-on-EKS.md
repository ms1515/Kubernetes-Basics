## Setting up Knative on EKS ##
AWS EKS Docs: https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html

1. IAM role configured for EKS to use. This is already created: `AWSServiceRoleForAmazonEKS`
2. SSH key created for communicating with EC2 worker nodes in the EKS cluster: `your-ec2-key-pair-name`
3. Have Kubernetes and Knative CLI installed (see local machine setup guide).
4. Install AWS Cli for your machine, as it is required for EKS Cli (eksctl) to work: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
- Configure the AWS Cli with your IAM User credentials:
```
aws configure
```
It will prompt you for your 
- Access Key ID (for your IAM User)
- Access Secret (for your IAM User)
- default region (Ireland for us): eu-west-1
- output: json

6. Install EKS Cli (eksctl) for your machine: https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html

**Docs for eksctl usage:** https://eksctl.io/usage/managing-nodegroups/

## Test Cluster Config ##

1. First confirm you're in the right Kubernetes context i.e AWS EKS by checking docker context.

2. Then, a Test cluster was created on EKS using eksctl with the cmd:
```
eksctl create cluster -f cluster-config.yaml
```

where cluster-config.yaml is:
```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: YOUR-EKS-Cluster-name
  region: your-region
managedNodeGroups:
  - name: your-nodegroup-name
    instanceType: m5.large
    desiredCapacity: 3
    ssh: # use EC2 key created for EKS
      publicKeyName: your-ec2-key-pair
```

3. Scale a cluster by providing the cluster config file

```
eksctl scale nodegroup --config-file=eks-test-cluster.yaml 
```

or specify the config in cmd:

```
eksctl scale nodegroup --cluster=HeartkeyAI-EKS-Test-Cluster --nodes=2 --name=hkai-ng-m5-large
```


## Knative Setup ##
1. Install Knative and other dependencies to see everything runs ok.

- Knative serving: v1.1.1
- istio: v1.0.0

Installed directly from Knative installation docs with these versions substituted in URLs. Works with Gitlab private registries.

- You can uninstall Knative using these instructions (remember to specify your versions): https://knative.dev/docs/install/uninstall/

2. Setup DNS on Route53 using this guide: https://knative.dev/docs/serving/using-a-custom-domain/#verification-steps

## Adding HTTPS (AWS instructions) ##

### Resources ###

1. Knative Docs - Configure HTTPS: https://knative.dev/docs/serving/using-a-tls-cert/#using-certbot-to-manually-obtain-lets-encrypt-certificates
2. Istio Docs - Secure Gateways: https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/
3. Istio Blog: https://rtfm.co.ua/en/istio-external-aws-application-loadbalancer-and-istio-ingress-gateway/
4. AWS ingress setup: https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/

### Steps to add HTTPS ###
1. Create a TLS Cert on ACM for your services domain.

Depending on your requirement you may create an ACM for a single root/nonroot domain. For example, if you own a root domain: 'domain.com' and you want several services running on it using this pattern:
- service1.domain.com
- service2.domain.com
- etc


You will need to create SSL certs for all these **subdomains** or create an SSL cert for a wildcard domain like this:
- \*.domain.com

Same is true for multiple level of subdomains, for example:
- service1.namespace.domain.com
- service2.namespace.domain.com


You will need to create a SSL cert for all the subdomains or a wild card one like:
- \*.namespace.domain.com
3. Load the TLS cert on Load balancer
4. OR: Add the following annotations to your istio-ingressgateway (which creates the load balancer) yaml to configure HTTPS and SSL. You can get that using:

```
kubectl edit svc istio-ingressgateway -n istio-system 
```

- And then add the annotations for HTTPS port mappings to use and SSL certificate ARN.

```
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:xx-xxxx-1:1234567890:certificate/xxxxxx-xxx-dddd-xxxx-xxxxxxxx"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "https"
```

- you will also need to map the ports for HTTP and HTTPS in the file like this (which the "ssl-ports" references for "https"):

```
  ports:
  - name: status-port
    nodePort: 31302
    port: 15021 # Load balancer Port
    protocol: TCP
    targetPort: 15021
  - name: http
    nodePort: 30945
    port: 80 # Load balancer Port
    protocol: TCP
    targetPort: 8080
  - name: https
    nodePort: 30979
    port: 443 # Load balancer Port
    protocol: TCP
    targetPort: 8080
```
- You need to ensure that the target port (on the node/instance for the Pod) is a suitable value for "http" and "https" mappings. SSL terminates at the load balancer and traffic is forwarded to our instances (and thereby our Pods) using HTTP.
- TargetPort is the port on which your pod will be listening on and where service will send requests to. It is the port number on the endpoint where the traffic will be received. Applicable only when used with ServiceEntries.
Your application in the container will need to be listening on this port also. NodePort exposes a service externally to the cluster by means of the target nodes IP address and the NodePort. 
- It does not matter too much which nodePort (or EC2 Instance Port) you choose, as long as it is a sensible value.
- Note, that in `metadata.annotations` for last applied configurations, you do not need to change it, as it is simply the last applied configuration. It has no effect on port mappings.

### Ingress Gateways ###

1. **Knative Ingress Gateway** for inside the cluster. Edit the config Gateway from here:

```
kubectl edit gateway knative-ingress-gateway -n knative-serving 
```

2. **Istio Ingress Gateway** for routing traffic from outside to the cluster. Edit the service from here:

``` 
kubectl edit svc istio-ingressgateway -n istio-system 
```

### Issues in setting up HTTPS ###
Links to relevant issue:

1. https://stackoverflow.com/questions/58005126/configure-istio-ingress-using-aws-elb-for-https-traffic-with-a-custom-acm-certif
2. https://stackoverflow.com/questions/63618094/after-adding-aws-acm-eks-elb-is-not-opening-on-https/63763088#63763088
3. https://stackoverflow.com/questions/54829341/unable-to-access-pod-over-https-via-istio-gateway-running-as-elb-on-aws-eks-wi

## Fargate ##

Delve into using EKS on Fargate to save money; as using Fargate will only charge us for the times Pods are running. 

- knative installs fine, but knative serving pods don't run
``` 
Warning  FailedScheduling  18s (x17 over 14m)  default-scheduler  0/2 nodes are available: 2 node(s) had taint {eks.amazonaws.com/compute-type: fargate}, that the pod didn't tolerate.

``` 

- Istio installs fine, but istio-system pods don't run
``` 
Warning  FailedScheduling  3s (x6 over 112s)  default-scheduler  0/2 nodes are available: 2 node(s) had taint {eks.amazonaws.com/compute-type: fargate}, that the pod didn't tolerate.
``` 

- Error Running Example App

``` 
Error: Internal error occurred: failed calling webhook "webhook.serving.knative.dev": Post "https://webhook.knative-serving.svc:443/?timeout=10s": no endpoints available for service "webhook"
Run 'kn --help' for usage
```

## API Gateway ##

1. Add Api Gateway for unified API paths 
2. Add Auth to API Gateway using Cognito userpool and implement in backend services.




## TroubleShooting ##

1. Sometimes it can take a few minutes for everyting to configure properly.

Especialy: If you have set up everything correctly: DNS, HTTPS, Gateway and Load balancers and SSL certs; 

## Next Steps ##



3. Add Mongo Db Deployment

4. Add K8 Dashboard with Auth; to easily see what's happening in Kubernetes environment.

5. Implement CI/CD; look into GitOps

6. Add EC2 User data; scripts to install required dependencies upon creation of cluster i.e Knative etc

