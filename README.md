#  Netflix Clone DevSecOps CI/CD Pipeline

An end-to-end **DevSecOps project** that automates the build, security analysis, containerization, deployment, and monitoring of a Netflix Clone application using **Jenkins, Docker, Kubernetes, SonarQube, OWASP Dependency-Check, Trivy, Prometheus, Grafana, and AWS EC2**.

> This repository documents the DevSecOps infrastructure and CI/CD workflow. The Netflix Clone application source code is maintained in a separate fork and is used by Jenkins as the application repository.

---

##  Project Overview

The goal of this project is to build a complete **CI/CD pipeline** for a Netflix Clone application and deploy it to a **Kubernetes cluster running on AWS**.

The pipeline performs:

1. Source code checkout from GitHub
2. Dependency installation
3. SonarQube code analysis
4. SonarQube Quality Gate verification
5. OWASP Dependency-Check
6. Trivy filesystem scan
7. Docker image build
8. Trivy Docker image scan
9. Docker image push to DockerHub
10. Kubernetes deployment
11. Infrastructure monitoring with Prometheus and Grafana

---

##  Architecture

```text
GitHub
   │
   ▼
Jenkins
   │
   ├── Install Dependencies
   ├── SonarQube Analysis
   ├── Quality Gate
   ├── OWASP Dependency-Check
   ├── Trivy Filesystem Scan
   ├── Docker Build
   ├── Trivy Image Scan
   ├── DockerHub Push
   └── Kubernetes Deployment
                    │
                    ▼
               Kubernetes
                    │
              ┌─────┴─────┐
              ▼           ▼
           Pod #1       Pod #2
              │           │
              └─────┬─────┘
                    ▼
             Netflix Application
```

### Monitoring Architecture

```text
Kubernetes Nodes
       │
       ▼
 Node Exporter
       │
       ▼
  Prometheus
       │
       ▼
    Grafana
```

---

#  Technologies Used

| Category | Technology |
|---|---|
| Cloud | AWS EC2 |
| Operating System | Ubuntu |
| Version Control | Git & GitHub |
| CI/CD | Jenkins |
| Code Quality | SonarQube |
| Dependency Security | OWASP Dependency-Check |
| Vulnerability Database | NVD |
| Security Scanner | Trivy |
| Containers | Docker |
| Container Registry | DockerHub |
| Container Orchestration | Kubernetes |
| Cluster Bootstrap | kubeadm |
| Container Runtime | containerd |
| Kubernetes Networking | Flannel CNI |
| Monitoring | Prometheus |
| Metrics | Node Exporter |
| Visualization | Grafana |
| Web Server | Nginx |
| Frontend | React / Vite |

---

#  1. AWS EC2 Infrastructure

The project is hosted on **AWS EC2**.

The final architecture uses **two EC2 instances**:

```text
AWS VPC
│
├── DevSecOps / Worker EC2
│   │
│   ├── Jenkins
│   ├── Docker
│   ├── SonarQube
│   ├── Prometheus
│   ├── Grafana
│   ├── Node Exporter
│   └── Kubernetes Worker
│
└── Kubernetes Control Plane EC2
    │
    ├── Kubernetes API Server
    ├── Scheduler
    ├── Controller Manager
    ├── etcd
    └── Node Exporter
```

The existing DevSecOps EC2 instance was reused as the **Kubernetes worker**, avoiding the need for a third EC2 instance.

---

##  AWS Security Groups

AWS Security Groups control which services can communicate with the EC2 instances.

Important ports used during the project:

| Port | Service | Purpose |
|---:|---|---|
| 22 | SSH | Remote administration |
| 3000 | Grafana | Monitoring dashboards |
| 6443 | Kubernetes API | Kubernetes Control Plane API |
| 8080 | Jenkins | CI/CD server |
| 9000 | SonarQube | Code analysis |
| 9090 | Prometheus | Monitoring |
| 9100 | Node Exporter | Server metrics |
| 10250 | Kubelet | Control Plane → Worker communication |
| 30007 | NodePort | Netflix application |

SSH access is restricted to a trusted client IP.

---

#  2. SSH Access

SSH is used to securely connect from the local machine to the AWS Ubuntu servers.

Example:

```bash
ssh -i ~/Downloads/netflix-devsecops-key.pem ubuntu@<EC2_PUBLIC_IP>
```

SSH allows remote administration of the EC2 instances, including installing tools, configuring Kubernetes, checking logs, and troubleshooting services.

---

#  3. Jenkins Setup

Jenkins is the **CI/CD orchestrator** of the project.

It automatically executes the different stages required to build, analyze, secure, containerize, and deploy the Netflix application.

Jenkins tools configured:

- NodeJS
- JDK
- SonarQube Scanner
- OWASP Dependency-Check
- Docker
- kubectl

Jenkins plugins/integrations include:

- SonarQube
- OWASP Dependency-Check
- NodeJS
- Prometheus

---

#  4. GitHub Checkout

Jenkins retrieves the Netflix Clone source code from GitHub.

```groovy
stage('Checkout from Git') {
    steps {
        git branch: 'main',
            url: '<NETFLIX_CLONE_REPOSITORY>'
    }
}
```

Flow:

```text
GitHub
   ↓
Jenkins Workspace
```

---

#  5. Install Dependencies

Jenkins installs the Node.js dependencies required by the React application.

```groovy
stage('Install Dependencies') {
    steps {
        sh 'node --version'
        sh 'npm --version'
        sh 'npm install'
    }
}
```

---

#  6. SonarQube Analysis

SonarQube performs static code analysis.

It helps detect:

- Bugs
- Code smells
- Maintainability problems
- Security issues

Jenkins uses **SonarScanner** to send the project to SonarQube.

```groovy
stage('SonarQube Analysis') {
    steps {
        script {
            def scannerHome = tool 'sonar-scanner'

            withSonarQubeEnv('sonar-server') {
                sh """
                ${scannerHome}/bin/sonar-scanner \
                -Dsonar.projectKey=Netflix \
                -Dsonar.projectName=Netflix \
                -Dsonar.sources=.
                """
            }
        }
    }
}
```

Flow:

```text
Source Code
    ↓
SonarScanner
    ↓
SonarQube
```

---

#  7. SonarQube Quality Gate

After SonarQube finishes the analysis, Jenkins checks the **Quality Gate**.

```groovy
stage('Quality Gate') {
    steps {
        script {
            waitForQualityGate abortPipeline: false,
                               credentialsId: 'Sonar-token'
        }
    }
}
```

A SonarQube webhook communicates the result back to Jenkins.

```text
Jenkins
   ↓
SonarScanner
   ↓
SonarQube
   ↓
Quality Gate
   ↓
Webhook
   ↓
Jenkins
```

---

#  8. OWASP Dependency-Check

OWASP Dependency-Check scans application dependencies for known vulnerabilities.

It uses vulnerability information from the **National Vulnerability Database (NVD)**.

```groovy
stage('OWASP Dependency Check') {
    steps {
        dependencyCheck additionalArguments: '--scan ./',
                        odcInstallation: 'DP-Check',
                        nvdCredentialsId: 'nvd-api-key'

        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
    }
}
```

The NVD API key is securely stored inside **Jenkins Credentials** and is not committed to GitHub.

Flow:

```text
Application Dependencies
        ↓
OWASP Dependency-Check
        ↓
NVD
        ↓
Vulnerability Report
```

---

#  9. Trivy Filesystem Scan

Trivy scans the project filesystem before the Docker image is built.

```groovy
stage('Trivy File System Scan') {
    steps {
        sh 'trivy fs .'
    }
}
```

This provides another security layer in the CI pipeline.

---

#  10. Docker

Docker packages the Netflix application and its runtime into a portable image.

The application uses a **multi-stage Docker build**.

```dockerfile
FROM node:18-alpine as builder

WORKDIR /app

COPY ./package.json .
COPY ./yarn.lock .

RUN yarn install

COPY . .

ARG TMDB_V3_API_KEY

ENV VITE_APP_TMDB_V3_API_KEY=${TMDB_V3_API_KEY}
ENV VITE_APP_API_ENDPOINT_URL="https://api.themoviedb.org/3"

RUN yarn build


FROM nginx:stable-alpine

WORKDIR /usr/share/nginx/html

RUN rm -rf ./*

COPY --from=builder /app/dist .

EXPOSE 80

ENTRYPOINT ["nginx", "-g", "daemon off;"]
```

The first stage builds the React/Vite application.

The second stage uses Nginx to serve the generated static files.

```text
React Source
     ↓
Node.js Builder
     ↓
dist/
     ↓
Nginx
     ↓
Docker Image
```

---

#  11. Docker Image Build

Jenkins builds the Docker image.

```groovy
stage('Docker Build') {
    steps {
        sh 'docker build -t netflix:${BUILD_NUMBER} .'
    }
}
```

The Jenkins build number is used as the image tag.

Example:

```text
netflix:27
netflix:28
netflix:29
```

This allows every pipeline execution to produce a versioned image.

---

# 🔎 12. Trivy Docker Image Scan

After building the image, Jenkins scans it using Trivy.

```groovy
stage('Trivy Image Scan') {
    steps {
        sh 'trivy image netflix:${BUILD_NUMBER}'
    }
}
```

Flow:

```text
Docker Image
     ↓
Trivy
     ↓
Vulnerability Report
```

---

#  13. DockerHub Push

After the image is built and scanned, Jenkins pushes it to DockerHub.

DockerHub credentials are stored securely inside Jenkins.

```groovy
stage('Docker Push') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'dockerhub-credentials',
                usernameVariable: 'DOCKERHUB_USER',
                passwordVariable: 'DOCKERHUB_TOKEN'
            )
        ]) {
            sh '''
                echo "$DOCKERHUB_TOKEN" | docker login \
                    -u "$DOCKERHUB_USER" \
                    --password-stdin

                docker tag netflix:${BUILD_NUMBER} \
                    <DOCKERHUB_USERNAME>/netflix:${BUILD_NUMBER}

                docker push \
                    <DOCKERHUB_USERNAME>/netflix:${BUILD_NUMBER}
            '''
        }
    }
}
```

Flow:

```text
Jenkins
   ↓
Docker Build
   ↓
Trivy Scan
   ↓
DockerHub
```

---

#  14. Kubernetes Cluster

The Kubernetes cluster was created manually using **kubeadm**.

The cluster contains:

```text
Kubernetes Cluster
       │
 ┌─────┴─────┐
 │           │
 ▼           ▼
Control     Worker
Plane        Node
               │
               ▼
              Pods
```

The environment uses Kubernetes **v1.37**.

---

#  15. Kubernetes Node Preparation

Before initializing Kubernetes, the nodes were prepared.

Configuration included:

- Disable swap
- Load `overlay`
- Load `br_netfilter`
- Enable IP forwarding
- Configure bridge networking
- Install containerd
- Enable systemd cgroups
- Install kubelet
- Install kubeadm
- Install kubectl

Important network settings:

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

---

#  16. containerd

Kubernetes uses **containerd** as its container runtime.

The configuration was generated and:

```text
SystemdCgroup = true
```

was enabled.

This ensures that Kubernetes and containerd use compatible cgroup management.

---

# 🧠 17. Kubernetes Control Plane

The Control Plane manages the Kubernetes cluster.

It contains:

```text
Control Plane
│
├── kube-apiserver
├── kube-scheduler
├── kube-controller-manager
└── etcd
```

The Control Plane was initialized using `kubeadm`.

After initialization, Kubernetes generated a `kubeadm join` command for adding worker nodes.

---

#  18. Kubernetes Worker

The existing DevSecOps EC2 server was joined to the Kubernetes cluster as the **worker node**.

The worker is responsible for running application workloads.

The cluster was verified using:

```bash
kubectl get nodes
```

Final result:

```text
Control Plane → Ready
Worker        → Ready
```

---

#  19. Flannel CNI

Initially, the Kubernetes nodes appeared as:

```text
NotReady
```

and CoreDNS Pods were:

```text
Pending
```

The reason was that Kubernetes did not yet have a **CNI network plugin**.

Flannel was installed to provide Kubernetes Pod networking.

After installing Flannel:

```bash
kubectl get nodes
```

showed both nodes as:

```text
Ready
```

and:

```bash
kubectl get pods -A
```

showed CoreDNS and Flannel Pods as:

```text
Running
```

---

#  20. Kubernetes Deployment

The Netflix application repository contains:

```text
Kubernetes/
├── deployment.yml
└── service.yml
```

The Deployment defines the Netflix application and maintains **two replicas**.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: netflix-app

spec:
  replicas: 2

  selector:
    matchLabels:
      app: netflix-app

  template:
    metadata:
      labels:
        app: netflix-app

    spec:
      containers:
        - name: netflix-app
          image: <DOCKERHUB_USERNAME>/netflix:<TAG>

          ports:
            - containerPort: 80
```

Architecture:

```text
Deployment
    │
    ▼
ReplicaSet
    │
 ┌──┴──┐
 ▼     ▼
Pod 1  Pod 2
```

The two Netflix Pods were successfully scheduled on the worker node.

Verification:

```bash
kubectl get pods -o wide
```

---

#  21. Kubernetes NodePort Service

A Kubernetes **NodePort Service** exposes the Netflix application outside the cluster.

```yaml
apiVersion: v1
kind: Service

metadata:
  name: netflix-app

spec:
  type: NodePort

  ports:
    - port: 80
      targetPort: 80
      nodePort: 30007

  selector:
    app: netflix-app
```

Application traffic follows:

```text
Browser
   ↓
AWS Worker EC2
   ↓
Port 30007
   ↓
Kubernetes NodePort Service
   ↓
Netflix Pod
   ↓
Nginx
   ↓
React Application
```

---

#  22. Kubernetes Kubelet Troubleshooting

While testing:

```bash
kubectl logs <netflix-pod>
```

the following problem appeared:

```text
dial tcp <WORKER_PRIVATE_IP>:10250: i/o timeout
```

The Kubernetes Control Plane could not communicate with the worker's **kubelet**.

Kubelet uses:

```text
TCP 10250
```

The AWS Security Group was updated to allow trusted Kubernetes communication on port `10250`.

After the rule was added:

```bash
kubectl logs <netflix-pod>
```

worked successfully and returned the Nginx logs.

---

#  23. Jenkins → Kubernetes Integration

The next objective was to allow Jenkins to automatically deploy to Kubernetes.

The Kubernetes kubeconfig was configured for Jenkins.

`kubectl` was verified on the Jenkins EC2 server:

```bash
kubectl version --client
```

and also for the Jenkins user:

```bash
sudo -u jenkins kubectl version --client
```

The kubeconfig is stored securely and should **never be committed to GitHub**.

---

#  24. Kubernetes Deployment from Jenkins

Jenkins can execute the Kubernetes manifests after pushing the Docker image.

```groovy
stage('Deploy to Kubernetes') {
    steps {
        withKubeConfig([
            credentialsId: '<KUBECONFIG_CREDENTIAL_ID>'
        ]) {
            sh 'kubectl apply -f Kubernetes/deployment.yml'
            sh 'kubectl apply -f Kubernetes/service.yml'
        }
    }
}
```

The complete deployment flow becomes:

```text
Jenkins
   ↓
Docker Build
   ↓
DockerHub
   ↓
kubectl
   ↓
Kubernetes API Server
   ↓
Deployment
   ↓
Worker Node
   ↓
Netflix Pods
```

---

#  25. Prometheus Monitoring

Prometheus is used to collect infrastructure and Jenkins metrics.

Prometheus monitors:

- Prometheus itself
- Jenkins
- DevSecOps / Worker server
- Kubernetes Control Plane

Prometheus configuration is stored in:

```text
/etc/prometheus/prometheus.yml
```

Before restarting Prometheus, the configuration can be validated with:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

A successful validation returns:

```text
SUCCESS: /etc/prometheus/prometheus.yml is valid prometheus config file syntax
```

---

#  26. Node Exporter

Node Exporter exposes Linux system metrics on:

```text
Port 9100
```

Metrics include:

- CPU
- RAM
- Disk
- Network
- System load
- Uptime

Connectivity can be tested using:

```bash
curl http://<NODE_PRIVATE_IP>:9100/metrics
```

---

#  27. Kubernetes Control Plane Monitoring

Node Exporter was installed on the Kubernetes Control Plane.

Prometheus was configured with an additional target:

```yaml
- job_name: 'k8s-master'
  static_configs:
    - targets:
        - '<K8S_MASTER_PRIVATE_IP>:9100'
```

After restarting Prometheus, the target appeared as:

```text
k8s-master (1/1 up)
```

This confirms that Prometheus can successfully collect metrics from the Kubernetes Control Plane.

---

#  28. Jenkins Monitoring

The Jenkins Prometheus plugin exposes Jenkins metrics.

The monitoring flow is:

```text
Jenkins
   ↓
/prometheus
   ↓
Prometheus
```

The Jenkins target was verified in Prometheus as:

```text
jenkins (1/1 up)
```

---

#  29. Prometheus Targets

The final Prometheus targets include:

```text
jenkins       → UP
k8s-master    → UP
node          → UP
prometheus    → UP
```

This confirms that Prometheus is successfully collecting metrics from the infrastructure.

---

#  30. Grafana

Grafana is used to visualize the metrics collected by Prometheus.

Prometheus was configured as the Grafana data source:

```text
Grafana
   ↓
Prometheus
   ↓
Node Exporter / Jenkins
```

The **Node Exporter Full** dashboard was imported.

The dashboard displays:

- CPU usage
- RAM usage
- System load
- Disk usage
- Network traffic
- Uptime

The Kubernetes Control Plane can be selected from the dashboard to display its metrics.

---

#  Security Throughout the Pipeline

Security is integrated at several levels.

## Source Code Security

```text
SonarQube
     +
Quality Gate
```

## Dependency Security

```text
OWASP Dependency-Check
          ↓
         NVD
```

## Container Security

```text
Trivy Filesystem Scan
        +
Trivy Image Scan
```

## Infrastructure Security

```text
AWS Security Groups
        +
Restricted SSH
        +
Restricted Kubernetes Ports
```

## Secret Management

Sensitive credentials are stored in **Jenkins Credentials**.

The following must never be committed to GitHub:

```text
DockerHub tokens
NVD API keys
SonarQube tokens
TMDB API keys
Kubernetes kubeconfig
SSH private keys (.pem)
Passwords
```

---

#  Problems Solved During the Project

Several real DevOps problems were encountered and solved:

- Jenkins Docker socket permissions
- Jenkins NodeJS configuration
- Jenkins JDK configuration
- SonarScanner configuration
- SonarQube webhook integration
- SonarQube Quality Gate communication
- OWASP Dependency-Check configuration
- NVD API integration
- Dockerfile Node.js compatibility
- Docker image build problems
- DockerHub authentication
- Kubernetes nodes initially `NotReady`
- CoreDNS initially `Pending`
- Missing Kubernetes CNI
- Flannel networking configuration
- Kubernetes worker joining the cluster
- Kubelet TCP `10250` timeout
- AWS Security Group communication
- Kubernetes NodePort configuration
- Jenkins Kubernetes authentication
- Prometheus configuration
- Node Exporter connectivity
- Kubernetes Control Plane monitoring
- Grafana dashboard configuration

---

#  Complete DevSecOps Pipeline

```text
Developer
    │
    ▼
GitHub
    │
    ▼
Jenkins
    │
    ├── Checkout
    │
    ├── Install Dependencies
    │
    ├── SonarQube Analysis
    │
    ├── Quality Gate
    │
    ├── OWASP Dependency-Check
    │
    ├── Trivy Filesystem Scan
    │
    ├── Docker Build
    │
    ├── Trivy Image Scan
    │
    ├── DockerHub Push
    │
    └── Kubernetes Deployment
                    │
                    ▼
             Control Plane
                    │
                    ▼
               Worker Node
                    │
              ┌─────┴─────┐
              ▼           ▼
           Pod #1       Pod #2
              │           │
              └─────┬─────┘
                    ▼
             NodePort :30007
                    │
                    ▼
              Netflix Clone
```

---

#  Complete Monitoring Flow

```text
Kubernetes Control Plane
          │
          ▼
     Node Exporter
          │
          │
          ├─────────────────┐
          │                 │
          ▼                 │
      Prometheus ◄──────── Jenkins
          │
          ▼
        Grafana
          │
          ▼
      Dashboards
```

---

#  Repository Structure

```text
Netflix-Clone-CI-CD-with-Monitoring-Email/
│
├── README.md
│
├── Jenkinsfile
│
├── Kubernetes/
│   ├── deployment.yml
│   └── service.yml
│
├── screenshots/
│   ├── jenkins-pipeline.png
│   ├── sonarqube.png
│   ├── kubernetes-pods.png
│   ├── prometheus-targets.png
│   └── grafana-dashboard.png
│
└── docs/
    └── architecture.md
```

---

#  Recommended Screenshots

The repository can include screenshots showing the results of each major DevSecOps stage:

- Jenkins successful pipeline
- SonarQube dashboard
- SonarQube Quality Gate
- OWASP Dependency-Check report
- DockerHub image
- Kubernetes nodes
- Kubernetes Pods
- Kubernetes Service
- Netflix application running on Kubernetes
- Prometheus Targets
- Grafana Node Exporter dashboard

>  Always verify screenshots before committing them. Never expose API keys, passwords, tokens, private keys, or other credentials.

---

#  Project Status

- [x] AWS EC2 infrastructure
- [x] Git & GitHub
- [x] Jenkins
- [x] Docker
- [x] SonarQube
- [x] SonarScanner
- [x] Quality Gate
- [x] SonarQube Webhook
- [x] OWASP Dependency-Check
- [x] NVD integration
- [x] Trivy filesystem scan
- [x] Docker multi-stage build
- [x] Trivy image scan
- [x] DockerHub push
- [x] Kubernetes Control Plane
- [x] Kubernetes Worker
- [x] containerd
- [x] Flannel CNI
- [x] Kubernetes Deployment
- [x] Kubernetes NodePort Service
- [x] Jenkins → Kubernetes integration
- [x] Prometheus
- [x] Node Exporter
- [x] Jenkins monitoring
- [x] Kubernetes Control Plane monitoring
- [x] Grafana
- [x] Jenkins email notifications

---

# 🎯 What I Learned

Through this project, I gained hands-on experience with:

- Designing an end-to-end DevSecOps pipeline
- Building CI/CD pipelines with Jenkins
- Integrating security into CI/CD
- Static code analysis with SonarQube
- Dependency vulnerability scanning with OWASP
- Container scanning with Trivy
- Docker multi-stage builds
- DockerHub image management
- Building a Kubernetes cluster using kubeadm
- Kubernetes Control Plane and Worker architecture
- Kubernetes Deployments and Services
- Kubernetes networking with Flannel
- AWS EC2 networking and Security Groups
- Debugging Kubernetes networking problems
- Jenkins-to-Kubernetes continuous deployment
- Infrastructure monitoring with Prometheus
- Linux metrics collection with Node Exporter
- Building monitoring dashboards with Grafana
- Secure credential management

---



# 👩‍💻 Author

**Hajar Azaou**

Software Engineering Student | Java Backend Development | DevOps & Cloud