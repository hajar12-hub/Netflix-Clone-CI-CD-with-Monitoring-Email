# Netflix Clone DevSecOps CI/CD Pipeline

A complete DevSecOps project that automates the build, security analysis,
containerization, deployment, and monitoring of a Netflix Clone application.

## Architecture

GitHub
↓
Jenkins
↓
SonarQube + OWASP Dependency-Check + Trivy
↓
Docker
↓
DockerHub
↓
Kubernetes
↓
Netflix Application

Monitoring:
Node Exporter → Prometheus → Grafana

## Technologies

- AWS EC2
- Jenkins
- Docker & DockerHub
- Kubernetes (kubeadm)
- Flannel CNI
- SonarQube
- OWASP Dependency-Check
- Trivy
- Prometheus
- Node Exporter
- Grafana
- Git & GitHub

## CI/CD Pipeline

1. Checkout source code from GitHub
2. Install application dependencies
3. Run SonarQube analysis
4. Check the SonarQube Quality Gate
5. Run OWASP Dependency-Check
6. Run Trivy filesystem scan
7. Build the Docker image
8. Run Trivy image scan
9. Push the Docker image to DockerHub
10. Deploy the application to Kubernetes

## Kubernetes

The Kubernetes cluster was created using `kubeadm` and consists of:

- Control Plane
- Worker Node
- Flannel CNI for pod networking
- Kubernetes Deployment with 2 Netflix replicas
- NodePort Service exposing the application

## Monitoring

Infrastructure monitoring is implemented using:

- Node Exporter for host metrics
- Prometheus for metrics collection
- Grafana for visualization

The monitoring setup tracks CPU, memory, disk, network, Jenkins, and
Kubernetes node metrics.

## Project Status

- [x] Jenkins CI/CD
- [x] SonarQube
- [x] OWASP Dependency-Check
- [x] Trivy
- [x] Docker build and push
- [x] Kubernetes deployment
- [x] Prometheus monitoring
- [x] Grafana dashboards
- [ ] Email notifications
