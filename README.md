# Netflix Clone with Full DevSecOps Implementation

This project demonstrates a **Netflix clone** deployed with a **full DevSecOps pipeline**, including code quality checks, security scanning, containerization, CI/CD automation, monitoring, and deployment to **EKS** with **ArgoCD**.

```mermaid
%%{init: {"theme": "neutral", "themeVariables": { "fontSize": "14px", "fontFamily": "Inter, Arial, sans-serif" }}}%%
flowchart LR
    %% PHASE 1: DEVELOPMENT
    subgraph DEV[Development]
        A[fa:fa-user Developer]
        B[fa:fa-github GitHub Repository]
        A -.->|Push Code| B
    end

    %% PHASE 2: CI/CD & SECURITY
    subgraph CICD[CI/CD & Security]
        C[fa:fa-jenkins Jenkins Pipeline]

        %% Analysis & Scans
        D[fa:fa-check-circle SonarQube<br/>Code Quality & SAST]
        E[fa:fa-shield-alt OWASP Dependency-Check<br/>SCA]
        F1[fa:fa-shield-virus Trivy FS<br/>Filesystem Scan]
        F2[fa:fa-shield-virus Trivy Image<br/>Image Scan]

        %% Build & Push
        G[fa:fa-docker Docker Build<br/>Containerization]
        H[fa:fa-docker Docker Hub<br/>Registry]

        %% Stage order
        C ==> D
        C ==> E
        C ==> F1
        D ==> G
        E ==> G
        F1 ==> G
        G ==> F2
        F2 ==> H

        B -.->|CI Trigger| C
    end

    %% PHASE 3: DEPLOYMENT
    subgraph DEPLOY[Deployment (EKS)]
        I[fa:fa-sync-alt ArgoCD<br/>GitOps]
        J[fa:fa-kubernetes Amazon EKS<br/>Kubernetes Cluster]

        I ==> J
        H ==> I
        B -.->|Manifest Watch| I
    end

    %% PHASE 4: MONITORING
    subgraph MON[Monitoring]
        K[fa:fa-server Node Exporter]
        L[fa:fa-chart-line Prometheus]
        M[fa:fa-chart-bar Grafana]

        J ==> K
        K ==> L
        C -.-> L
        L ==> M
    end

    %% NOTIFICATIONS
    N[fa:fa-envelope Email Notification]
    C -.->|Build Status & Reports| N
    F1 -.->|FS Scan Report| N
    F2 -.->|Image Scan Report| N
```

## Tools Used

- **Jenkins**: CI/CD automation
- **SonarQube**: Code quality and static analysis
- **Trivy**: Vulnerability scanning for files and container images
- **OWASP Dependency Check**: Security scanning of project dependencies
- **Prometheus & Grafana**: Monitoring and observability
- **Node Exporter**: Node-level metrics collection
- **Docker & Kubernetes (EKS)**: Containerization and cloud deployment
- **ArgoCD**: GitOps-based deployment

## Jenkins Setup

1. Install Jenkins on your local machine.

2. Install the following **Jenkins Plugins**:

   | Plugin                    | Purpose                                                      |
   | ------------------------- | ------------------------------------------------------------ |
   | Eclipse Temurin Installer | Provides JDK installation for builds                         |
   | NodeJS Plugin             | Allows Node.js builds and npm tasks                          |
   | OWASP Dependency Check    | Scans project dependencies for vulnerabilities               |
   | Git Plugin                | For fetching source code from GitHub (update if builds fail) |
   | Prometheus Metrics        | Exposes Jenkins metrics to Prometheus for monitoring         |

3. Configure tools:

   - Go to **Manage Jenkins → Tools**
   - Create **jdk17** under JDK Installations
   - Create **node16** under NodeJS Installations (needed for this pipeline)

## SonarQube Setup

SonarQube analyzes code quality and security vulnerabilities. Run it as a Docker container:

```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:lts
```

- Access SonarQube at: `http://localhost:9000`
- Configure `sonar-server` in Jenkins (**Manage Jenkins → System → SonarQube installations**)
- Enable Sonar Scanner in Jenkins (**Manage Jenkins → Tools → SonarQube Scanner Installations**)

## Trivy Installation

Trivy scans for vulnerabilities in both **filesystems** and **Docker images**, generating reports that are sent via email in this pipeline.

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```

## Prometheus & Grafana Monitoring Setup

Prometheus collects metrics (from Jenkins, Node Exporter, Kubernetes). Grafana visualizes them.

We’ll run Prometheus and Grafana as Docker containers, connected via a Docker network.

### Prometheus

Create `prometheus.yml`:

```yaml
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

  - job_name: "k8s"
    metrics_path: "/metrics"
    static_configs:
      - targets: ["51.21.150.243:9100"]
```

Run Prometheus:

```bash
docker run -d \
  --name prometheus \
  --network monitoring \
  -p 9090:9090 \
  -v ~/monitoring/prometheus.yml:/etc/prometheus/prometheus.yml \
  --add-host=host.docker.internal:host-gateway \
  prom/prometheus
```

### Grafana

```bash
docker run -d --name grafana -p 3000:3000 grafana/grafana
```

- Add **Prometheus** as a data source.
- Import dashboards:

  - **Node Exporter Dashboard** (ID: 1860) → CPU, memory, disk, network metrics
  - **Jenkins Dashboard** (ID: TBD) → CI/CD pipeline and build monitoring

**Note**: Jenkins runs on the host, Prometheus/Grafana in Docker. Use `http://host.docker.internal:8080` for Jenkins in `prometheus.yml`.

## AWS EKS Setup

1. Configure AWS CLI:

```bash
aws configure
aws eks update-kubeconfig --name Netflix --region eu-north-1
```

2. Verify cluster:

```bash
kubectl get ns
kubectl get pods
```

## ArgoCD Installation (GitOps)

Install ArgoCD inside the Kubernetes cluster:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

- Get ArgoCD DNS:

```bash
export ARGOCD_SERVER=$(kubectl get svc argocd-server -n argocd -o jsonpath="{.status.loadBalancer.ingress[0].hostname}")
echo $ARGOCD_SERVER
```

- Get ArgoCD admin password:

```bash
export ARGO_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
```

- Configure ArgoCD Application:

  - **name** → Application name
  - **destination** → Kubernetes cluster/namespace
  - **source** → GitHub repository (URL, branch, path)
  - **syncPolicy** → Enable auto-sync, pruning, self-healing

**Access your app**:
Open `NodeIP:30007` in a browser (ensure port 30007 is allowed in your Security Group).

## Helm & Node Exporter

Install Node Exporter for Kubernetes node monitoring:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
kubectl create namespace prometheus-node-exporter
helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter --namespace prometheus-node-exporter
```

## Jenkins Pipeline

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/kishgi/netflix-clone.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix'''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps { sh "npm install" }
        }
        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Trivy FS Scan') {
            steps { sh "trivy fs . > trivyfs.txt" }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build --build-arg TMDB_V3_API_KEY=bc373fb16fe8de9c49dd747e899dbaee -t netflix ."
                        sh "docker tag netflix kishgi/netflix:latest"
                        sh "docker push kishgi/netflix:latest"
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps { sh "trivy image kishgi/netflix:latest > trivyimage.txt" }
        }
        stage('Deploy to Container') {
            steps {
                sh "docker ps -q --filter 'publish=8081' | xargs -r docker stop"
                sh "docker ps -aq --filter 'publish=8081' | xargs -r docker rm"
                sh "docker run -d -p 8081:80 --name netflix-app kishgi/netflix:latest"
            }
        }
    }
    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                      "Build Number: ${env.BUILD_NUMBER}<br/>" +
                      "URL: ${env.BUILD_URL}<br/>",
                to: 'kishgi1234@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
```

## Pipeline Features

- Clean workspace
- Checkout code from GitHub
- SonarQube static code analysis & quality gate
- Install Node.js dependencies
- OWASP Dependency Check for vulnerabilities
- Trivy filesystem scan (`trivyfs.txt`)
- Docker build & push, image vulnerability scan (`trivyimage.txt`)
- Deploy Docker container locally
- Email notifications with logs & reports

## Notes / Issues Solved

- Port conflicts (e.g., 8081) handled with `docker stop` and `docker rm` steps.
- Jenkins metrics integrated with Prometheus for monitoring.
- Google App Passwords required for email notifications (if 2FA enabled).

## Summary

This project demonstrates a **complete DevSecOps pipeline**:

1. **CI/CD automation** with Jenkins
2. **Code quality & security scanning** with SonarQube, Trivy, OWASP Dependency Check
3. **Containerization & deployment** using Docker and Kubernetes (EKS)
4. **GitOps deployment** using ArgoCD
5. **Monitoring & observability** using Prometheus and Grafana

It is a **hands-on example** of integrating security, automation, monitoring, and cloud deployment in a real-world project.

## Future Improvements

- **Secrets Management**: Integrate AWS Secrets Manager or Kubernetes Secrets instead of hardcoding API keys/credentials.
- **Immutable Docker Tags**: Use Git commit SHA or build numbers rather than `latest` for reproducibility.
- **SonarQube Setup**: Move to a production-ready setup with PostgreSQL and persistent storage.
- **Centralized Monitoring**: Deploy Prometheus & Grafana inside Kubernetes via Helm for scalable service discovery.
- **ArgoCD Best Practices**: Replace NodePort exposure with Ingress + TLS termination (e.g., AWS ALB Ingress).
- **Enhanced Security Scans**: Add SAST and DAST tools for broader DevSecOps coverage.
- **Automated Testing**: Add unit, integration, and e2e tests in Jenkins pipeline.
- **Cost Optimization**: Use Terraform for AWS infrastructure management and cost-efficient scaling on EKS.
