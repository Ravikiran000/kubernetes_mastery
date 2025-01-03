# K8S Ingress Controller SetUp
This guide will help you set up Ingress Controllers, generate SSL keys, deploy Ingress Controllers, and manage Docker images in a Kubernetes cluster. We'll also create secrets and configure Route 53 records.

## Kubernetes Ingress
An Ingress is an API object that manages external access to the services in a cluster, typically HTTP and HTTPS traffic. It acts as a traffic controller that routes incoming requests to the appropriate backend services based on rules defined in the Ingress resource.
#### When to use Kubernetes Ingress?
There are many different use cases for Ingress:
- Exposing multiple services through a single entry point simplifying traffic routing through URIs, paths, headers, or other methods.
- SSL/TLS termination – simplify certificate management and reduce overhead on your services.
- Authentication and authorization – implement secure access to your services.
- Load balancing – even though Ingress and the load balancer service have a lot in common, ingress is internal to the cluster and allows you to route to different services, while the load balancer component is external to the cluster, letting you route traffic to a single service.
### Ingress vs. LoadBalancer vs. NodePort
Ingress, LoadBalancer, and NodePort are all ways of exposing services within your K8S cluster for external consumption.
NodePort and LoadBalancer let you expose a service by specifying that value in the service’s type.
- With a NodePort, K8S allocates a specific port on each node to the service specified. Any request received on the port by the cluster simply gets forwarded to the service.
- With a LoadBalancer, there needs to be an external service outside of the K8S cluster to provide the public IP address. In AWS, this would be an Application Load Balancer (ALB) in front of your Elastic Kubernetes Service (EKS). Each time a new service is exposed, a new LoadBalancer needs to be created to get a public IP address. Conveniently, the Load balancer provisioning happens automatically for you because of the way the Cloud providers plugin to Kubernetes, so that doesn’t have to be done separately.
- Ingress is a completely independent resource to your service. As well as enabling routing rules to be consolidated in one place (the Ingress object), this has the advantage of being a separate, decoupled entity that can be created and destroyed separately from any services.
## Ingress Controllers
- An ingress controller acts as a reverse proxy and load balancer inside the Kubernetes cluster. It provides an entry point for external traffic based on the defined Ingress rules. Without the Ingress Controller, Ingress resources won’t work.
- The Ingress Controller doesn’t run automatically with a Kubernetes cluster, so you will need to configure your own. An ingress controller is typically a reverse web proxy server implementation in the cluster.
#### Key Components:
**Ingress Resource:**
- A Kubernetes API object that defines rules for routing HTTP and HTTPS traffic.
- Specifies mappings like hostnames, paths, and back-end services.
- Provides the configuration for routing and load balancing.
- Acts as a blueprint that the Ingress Controller uses to set up routing.

**Ingress Controller:**
- A software component running in the cluster that processes Ingress resources.
- Configures the underlying network (e.g., load balancer, reverse proxy) to match the rules defined in the Ingress resource.
Examples: NGINX Ingress Controller, Traefik, HAProxy, AWS ALB Ingress Controller, etc.
- Manages network traffic, SSL termination, and load balancing.

![image](https://github.com/user-attachments/assets/cbf73fc5-3ec9-43bc-af0a-86b8eaebf740)

### Implementation Steps (Nginx Ingress Controller Setup)
1. create one instance and generate TLS certificate & TLS Keys (This isntance is seperate from our cluster. Using this just for TLS cert & Keys)
```
sudo snap install --classic certbot

certbot certonly --manual --preferred-challenges=dns --key-type rsa --email your-email@gmail.com \
--server https://acme-v02.api.letsencrypt.org/directory --agree-tos -d *.your-dns.com
```
2. Copy the "_acme-challenge.your-dns.in" & create a TXT record in with name as "_acme-challenge" and it's respective value.

#####    3. Wait until the record is in sync and then press enter on the screen

4. Copy the generated TLS Cert & TLS Keys somewhere. (will be used later in Cluster).
5. Deploy the KOPS Cluster (same as Day-1 set-up)
6. Deploy Ingress Controller onto the cluster
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```
7. Create 2 files tls.crt & tls.key to store TLS cert & TLS key. Copy the TLS cert & TLS keys into the management instance in /tmp location.
```
cd /tmp
vi tls.crt
vi tls.key
```
8. Create a Secret for the Keys
```
kubectl create secret tls nginx-tls-default --key="tls.key" --cert="tls.crt"
```
9. check the tls secret (which is named as nginx-tls-default. If successful, will show data in num of bytes)
```
kubectl describe secret nginx-tls-default
```
10. Deploying an voting application. Below is the architecture of the application. It uses vote, results as frontend. worker as backend. redis(in-memory database) & Postgres DB for Database. Application will be deployed using yml manifest. vote, results, worker images will be pulled from dockerhub. Official redis, postgres images will be used.

![image](https://github.com/user-attachments/assets/05d6f5fe-4197-4bbe-83fd-cc5201183744)

11. Create one private docker repository to store the images. (we pull the public images and re-tag them and store them in private repository)
  - **Repository name:** votingapp
12. Download the docker images & Pushing the images into a private repository in Dockerhub.
```
# install docker
curl https:/get.docker.com

# login to dockerhub to push repos
docker login -u <username>

# pull vote image
docker pull kiran2361993/testing:latestappvote

# tag the vote image
docker tag kiran2361993/testing:latestappvote ravikiran000/votingapp:vote

#push the image into private repository
docker push ravikiran000/votingapp:vote

# pull result image
docker pull kiran2361993/testing:latestappresults

# tag the result image
docker tag kiran2361993/testing:latestappresults ravikiran000/votingapp:result

#push the image into private repository
docker push ravikiran000/votingapp:result

# pull worker image
docker pull kiran2361993/testing:latestappworker

# tag the worker image
docker tag kiran2361993/testing:latestappworker ravikiran000/votingapp:worker

#push the image into private repository
docker push ravikiran000/votingapp:worker

# Go to dockerhub and check the pushed images
```
13. Delete the images in the cluster since we pushed all the images into the private repository in the Dockerhub and will directly pull the images into the voting.yml manifest from private repository using secrets, which has username and docker token
```
docker rmi $(docker images -aq) -f
```
14. Create Secrets to pull docker images
```
kubectl create secret docker-registry docker-pwd --docker-username=<your-username> --docker-password=<your-password> --docker-email=<your-email>
```
14. Give the vote, result, worker images in voting.yml and Add imagePullSecrets under the images section in your YAML manifest.
```
imagePullSecrets:
  - name: docker-pwd
```
15. Since you deployed the Ingress controller, a network loadbalancer is created. Create an A type record in Route53 using the nlb dns and give subdomain as "www". So, if someone is accessing the www.example.com, it will redirect to your voting app.
16. Go to Route 53 and create the following records:
- www
- vote
- result
17. Deploy Ingress Resource to route traffic according to the service.
  - If users are accessing vote service, they should be redirected to vote service.
       - Defined prefix as vote & www in Ingress resource. If anybody accesses vote.example.com, www.example.com then they'll be redirected to vote service, to cast a vote.
  - If users are accessing result service, they should be redirected to result service.
       - Defined prefix as result in Ingress resource. If anybody access result.example.com then they'll be redirected to result service, to see results.
