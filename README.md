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

