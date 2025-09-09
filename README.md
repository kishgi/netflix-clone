# Netflix clone with full DevSecOps Implementation

Installed and Run Jenkins is run on my local machine

SOnar qube is created and run as docker container
docker run -d --name sonarqube -p 9000:9000 sonarqube:lts

Trivy is installed in locla machine
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y

Jenkins Plugins needed and their reasons
Eclipse Temurin Installer 
NodeJs Plugin 
Owasp dependency-check



issues faced
Git plugin i had to update it i realised it after 3 build failures

Prometheus and Grafana Setup

Starting promethrus in docker
Create a yaml file
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]


docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v ~/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus


Grafana
docker run -d \
  --name=grafana \
  -p 3000:3000 \
  grafana/grafana


Node exporter

global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]


docker restart prometheus

Add a Prometheus data source (as before).

Import a Node Exporter dashboard from Grafana dashboards (ID: 1860 is popular).

You’ll see all CPU, memory, disk, and network metrics from your host.

issues faced- 
I run Grafana and Prometgheus in docker container but Jenkins was on my host so i hade to add http://host.docker.internal:8080 to add jenkins 

Jenkins plugin
Prometheus metrics

Go to google account and Take App passwords to send notifictions from pipeline
(make sure whether the google account has 2FA)

global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: "jenkins"
    metrics_path: "/prometheus"
    static_configs:
      - targets: ["host.docker.internal:8080"]


docker run -d \
  --name prometheus \
  --network monitoring \
  -p 9090:9090 \
  -v ~/monitoring/prometheus.yml:/etc/prometheus/prometheus.yml \
  --add-host=host.docker.internal:host-gateway \
  prom/prometheus


issues faced the port 8081 is already in use
implemented - docker ps -q --filter "publish=8081" | xargs -r docker stop in pipeline


EKS part
aws configure - and configure your aws
aws eks update-kubeconfig --name Netflix --region eu-north-1

check with
kubectl get ns
kubectl get pods

Argo Cd in EKS
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

Service
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

kubectl create namespace prometheus-node-exporter

helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter --namespace prometheus-node-exporter

export ARGOCD_SERVER=$(kubectl get svc argocd-server -n argocd -o json | jq -r '.status.loadBalancer.ingress[0].hostname')

echo $ARGOCD_SERVER

You will get the DNS for ArgoCD

GO tthere username is admin

export ARGO_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)


