
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
     <dl>
   
## Install a LoadBalancer solution for bare metal deployment
I use [MetalLB](https://metallb.org/) for local load balancer solution. I'll use the addon directly that microk8s provides. (if you need to install the package by yourself you'll need to deploy a configMap file to configure the range, check out the docs)
    <dl>
     <dt>> `microk8s enable metallb`</dt>
     <dt>> Enter the range of ip you want metallb handles for you </dt>
    <dl>
</br>

# Creating container image
## 1. Prepare registry
   You can use docker hub or any other private registry out there   
   I use [microk8s built-in registry](https://microk8s.io/docs/registry-built-in)  
   You can enable it by `microk8s enable registry`. This will create a generic v2 image repository on your own kubernetes cluster. (That's why we'll use localhost:32000)
## 2. Generate Image for project
   To build a new image we use `docker build -t localhost:32000/hello-world:latest images/hello-world/` that will include all files and folders on the path `images/hello-world`. By default it will search for a `Dockerfile` file under that directory, but you can change using the `--file` flag.
## 3. Push image to hub/repository
   We have already tagged the image so no need to do it again. If you don't, you can do it using `docker tag {imageID} localhost:32000/hello-world:latest`<br>
   Next step is to push to the registry using the image tag: `docker push localhost:32000/hello-world:latest`. (If you want to validate that the image is pushed correctly, you can do it by doing `curl http://localhost:32000/v2/hello-world/tags/list`, this will return a json with correct data from registry)
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
   Once you have install helm, you can deploy the hello-world chart, by providing the chart path and a name for the release.
    <dl>
      <dt>> `helm install hello-world-release k8s/hello-world/helm-chart`</dt>
    </dl>
   You can also create a package and then deploy the package by doing:
    <dl>
      <dt>> `helm package k8s/hello-world/helm-chart -d k8s/hello-world/helm-chart`</dt>
      <dt>> `helm install hello-world-release k8s/hello-world/helm-chart/TBD`</dt>
    </dl>
   You can now see the deployment of the release using `helm list` and the kubernetes deployment running using `kubectl get deploy` and `kubectl get service`
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


