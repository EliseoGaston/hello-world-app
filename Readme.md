
# Pre-Requisits
 I'm using ubuntu 20.04 here.. You'll need just a code editor
 
## Install docker
I use [Docker](https://docs.docker.com). You can install it by:
    <dl>
		<dt>> `sudo snap install docker`</dt>
		<dt>> `sudo addgroup --system docker`</dt>
		<dt>> `sudo adduser $USER docker`</dt>
		<dt>> `newgrp docker`</dt>
		<dt>> `sudo snap disable docker`</dt>
		<dt>> `sudo snap enable docker`</dt>
    <dl>

## Install kubernetes local cluster
I use [MicroK8s](https://microk8s.io). You can install it by:
    <dl>
     <dt>> `sudo snap install microk8s --classic`</dt>
     <dt>> `microk8s status --wait-ready`</dt>
     <dt>> `microk8s enable dns storage`</dt>
     <dt>> `microk8s enable metallb`</dt>
     <dl>
   
## Install a LoadBalancer solution for bare metal
I use [MetalLB]() as the bare metal load balancer solution. I'll use the addon directly that microk8s provides, if you need to install the package by yourself it will need to deploy a configMap file to configure the range.
    <dl>
     <dt>>`microk8s enable metallb`</dt>
     <dt>> Enter the range of ip you want him to manage for this. In my case I'll put (192.168.100.100-192.168.100.130)</dt>
     <dl>
</br>

# Creating container image
## 1. Prepare registry
   You can make use of docker hub or any other private registry out there   
   I use [microk8s built-in registry](https://microk8s.io/docs/registry-built-in)  
   You can enable it by `microk8s enable registry`. This will create a generic v2 image repository on your own kubernetes cluster. (That's why we use localhost:32000 instead of a outside internet url)
## 2. Generate Image for project
   Now that docker works and you have set a registry on your machine, you can build your project using the Dockerfile and the html + config files. To build a new image `docker build -t localhost:32000/hello-world:latest images/hello-world/`.   
   To Push to a hub/repository you should tag an image as we have done here just pointing to the repository url(in this case it's localhost:32000 because I'm using local registry addon provided by microk8s, but you can use docker hub if you want, or another private generic repo)
   We are specifing that this is the latest hello-world image we are building so the hub will not cache anything there.
## 3. Push image to hub/repository
   To be able to push a image to a repository, you need to tag with the correct url so docker can understand. We have already tagged the image when building so no need to do it again. If you don't, you can do it using `docker tag {imageID} localhost:32000/hello-world:latest`<br>  
   Next step is to push to the registry using the image tag: `docker push localhost:32000/hello-world:latest`. (If you want to validate it, you can do it by doing `curl http://localhost:32000/v2/hello-world/tags/list`, this will return a json with correct data from registry)  
</br>
</br>

# A Helm package for the image
## 1. Helm installation
   You can install [Helm v3](https://v3.helm.sh/docs/intro/install/)
    <dl>
      <dt>> `sudo snap install helm --classic`</dt>
      <dt>> `export KUBECONFIG=/var/snap/microk8s/current/credentials/client.config` ( microk8s does not use the default path for .kube/config so we need to export the correct path for helm)</dt>
    </dl>

## 2. Building and deploying helm chart
   Once you have install helm, you can deploy the hello-world chart that I made to consume the docker image we previusly built
    <dl>
      <dt>> `helm install hello-world-release k8s/hello-world/helm-chart`</dt>
    </dl>
   You can also create a package and then deploy the package by doing:
    <dl>
      <dt>> `helm package k8s/hello-world/helm-chart -d k8s/hello-world/helm-chart`</dt>
      <dt>> `helm install hello-world-release k8s/hello-world/helm-chart/TBD`</dt>
    </dl>
   You can now see the release using `helm list` and you can also check that the hello-world-release deployment is successfully running on the k8s cluster by doing `kubectl get deploy` and `kubectl get service`
   We have created a `ClusterIP` service for this deployment so we can only access from inside the cluster. You can `kubectl exec -it $(k get pods --selector=app.kubernetes.io/name=hello-world --output name) -- sh` and now you can `curl http://localhost` to see the hello-world html site.  
</br>
</br>

# Traefik Ingress
## 1. Install Traefik
   You can deploy traefik to kubernetes using traefik oficial helm package
    <dl>
      <dt>> `helm repo add traefik https://helm.traefik.io/traefik`</dt>
      <dt>> `helm repo update`</dt>
      <dt>> `helm install -n traefik --create-namespace traefik traefik/traefik --values k8s/hello-world/ingress/traefik/values.yaml` </dt>
    </dl>
   If you have already configured metallb correctly, now you can see the EXTERNAL-IP assigned to traefik service using `kubectl get all -n traefik`.
   We are using a values file because we want to enable he dashboard and tweak some other defaults. It's okey if you do not want to change anything. It will still work the same. 
   
## 2. Create IngressRoute for our service
   You have already a ClusterIP service for our hello-world backend. To expose this to the internet using traefik, as an ingress controller, you need to can create an ingress route that redirect to the backend service. 
      <dl>
        <dt>> `kubectl apply -f k8s/hello-world/ingress/traefik/hello-world-ingress-route.yaml`</dt>
    </dl>
   Here we are defining a route that match `hello-world.local` on default http traefik endpoint called `web` and redirecting traffic to service `hello-world-release` on default http port 80, that what we deploy with our helm chart.
   To make it work we should use a DNS server or something similiar beyond your network, but for demo porpouse and simpler I'll make use of /etc/hosts file under my system.
       <dl>
         <dt>> `kubectl get service -n traefik`</dt>
         <dt>> Copy the EXTERNAL-IP from the LoadBalancer called traefik</dt>
         <dt>> `sudo vim /etc/hosts`</dt>
         <dt>> Add a line at the bottom of the file to include {EXTERNAL-IP}{space-space}hello-world.local</dt>
         <dt>> Save and exit</dt>
    </dl>
   You should be able to navigate to hello-world.local on your favorite browser to see the hello-world app.


