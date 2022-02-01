Pre-Requisits

You would need to have installed:
	- code editor
	- Docker
	- local kubernetes cluster (microk8s in my case)
	- image repository(local)
	- metallb
	- helm package manager (helm 3)
	- traefik package
	- dns server (for demo porpouse I'll use /etc/hosts file )

git clone demo
sudo snap install microk8s
microk8s enable dns repository storage
sudo snap install helm3

-----
cd hello-world-app/images/hello-world
docker build -t hello-world:latest