# ☸️ Kubernetes Deployment — Netflix Clone

This document describes the Kubernetes setup used to deploy the **Netflix Clone application** as part of the **Netflix Clone DevSecOps CI/CD project**.

The Kubernetes cluster was created on **AWS EC2** using `kubeadm`, with a dedicated **Control Plane** and the existing DevSecOps EC2 instance reused as a **Worker Node**.

---

## 📌 Kubernetes Architecture

The Kubernetes cluster consists of two AWS EC2 instances:

| Node | Role | Description |
|---|---|---|
| `k8s-master` | Control Plane | Manages the Kubernetes cluster |
| `netflix-devsecops` | Worker Node | Runs the Netflix application Pods |

The Worker Node also hosts:

- Jenkins
- Docker
- SonarQube
- Prometheus
- Grafana
- Node Exporter

### Architecture

```text
                    AWS VPC
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
    k8s-master              netflix-devsecops
   Control Plane               Worker Node
          │                         │
          │                         ├── Jenkins
          │                         ├── Docker
          │                         ├── SonarQube
          │                         ├── Prometheus
          │                         ├── Grafana
          │                         │
          │                         ├── Netflix Pod 1
          │                         └── Netflix Pod 2
          │
          └──── Manages Worker ─────┘
```

---

# 1. Kubernetes Components

The cluster uses:

- **Kubernetes v1.37**
- **kubeadm** — creates and manages the cluster
- **kubelet** — runs on every Kubernetes node
- **kubectl** — communicates with the Kubernetes API
- **containerd** — container runtime
- **Flannel** — Container Network Interface (CNI)
- **AWS EC2** — hosts the Kubernetes nodes

---

# 2. Prepare the Nodes

Before installing Kubernetes, both the Control Plane and Worker Node were prepared.

## Disable Swap

Kubernetes requires swap to be disabled.

```bash
sudo swapoff -a
```

---

## Load Required Kernel Modules

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

These modules are required for container and Kubernetes networking.

---

## Configure Network Parameters

The following kernel parameters were enabled:

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

They allow Kubernetes networking and packet forwarding to work correctly.

---

# 3. Container Runtime

The cluster uses **containerd** as the container runtime.

The containerd configuration was updated to use systemd cgroups:

```text
SystemdCgroup = true
```

This allows Kubernetes and containerd to use compatible cgroup management.

---

# 4. Install Kubernetes

The following Kubernetes components were installed on the nodes:

```text
kubeadm
kubelet
kubectl
```

The installed Kubernetes version is:

```text
v1.37.0
```

The installation can be verified using:

```bash
kubectl version --client
```

---

# 5. Create the Control Plane

A separate AWS EC2 instance named:

```text
k8s-master
```

was created to act as the Kubernetes **Control Plane**.

The Control Plane is responsible for:

- Managing the cluster
- Managing the Kubernetes API
- Scheduling workloads
- Maintaining the desired cluster state
- Managing Worker Nodes

The cluster was initialized using `kubeadm`.

---

# 6. Configure kubectl

After initializing the Control Plane, the Kubernetes configuration was configured so that `kubectl` could communicate with the cluster.

The kubeconfig is stored under:

```text
~/.kube/config
```

> ⚠️ The kubeconfig contains sensitive cluster credentials and should never be committed to a public GitHub repository.

---

# 7. Join the Worker Node

The existing **DevSecOps EC2 instance** was reused as the Kubernetes Worker Node.

The Worker Node joined the Control Plane using the `kubeadm join` command generated during cluster initialization.

Architecture after joining:

```text
Kubernetes Cluster
│
├── Control Plane
│   └── k8s-master
│
└── Worker Node
    └── netflix-devsecops
```

---

# 8. Install Flannel CNI

After initializing the cluster, **Flannel** was installed as the Kubernetes Container Network Interface.

Flannel provides networking between:

- Pods
- Worker Nodes
- Control Plane
- Kubernetes services

After Flannel was installed successfully, the nodes reached the:

```text
Ready
```

state.

---

# 9. Verify the Cluster

The nodes can be checked using:

```bash
kubectl get nodes
```

The cluster contains:

```text
Control Plane  → k8s-master
Worker Node    → netflix-devsecops
```

More information can be displayed using:

```bash
kubectl get nodes -o wide
```

---

# 10. Netflix Kubernetes Deployment

The Netflix application is deployed using a Kubernetes **Deployment**.

The Deployment manages the application Pods and ensures that the requested number of replicas remains running.

File:

```text
Kubernetes/deployment.yml
```

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
          image: haja69245/netflix:27

          ports:
            - containerPort: 80
```

---

## Replicas

The Deployment uses:

```yaml
replicas: 2
```

This means Kubernetes maintains **two Netflix Pods**.

```text
Netflix Deployment
       │
       ├── Netflix Pod 1
       │      └── Netflix Container
       │
       └── Netflix Pod 2
              └── Netflix Container
```

If one Pod fails, Kubernetes automatically creates another Pod to maintain the desired number of replicas.

---

# 11. Kubernetes Service

Pods are internal to the Kubernetes cluster.

To make the Netflix application accessible from outside the cluster, a Kubernetes **NodePort Service** is used.

File:

```text
Kubernetes/service.yml
```

Example:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: netflix-app

spec:
  selector:
    app: netflix-app

  type: NodePort

  ports:
    - port: 80
      targetPort: 80
      nodePort: 30007
```

---

## Understanding the Ports

```text
User
 │
 │ :30007
 ▼
Worker Node
 │
 ▼
Kubernetes Service
 │
 │ targetPort: 80
 ▼
Netflix Pod
 │
 ▼
Nginx Container :80
```

The application is therefore accessible through:

```text
http://<WORKER-PUBLIC-IP>:30007
```

---

# 12. Deploy the Application

The Kubernetes resources can be created using:

```bash
kubectl apply -f Kubernetes/deployment.yml
```

and:

```bash
kubectl apply -f Kubernetes/service.yml
```

---

# 13. Verify the Deployment

## Check Deployment

```bash
kubectl get deployment
```

## Check Pods

```bash
kubectl get pods
```

For more information:

```bash
kubectl get pods -o wide
```

The Netflix Pods should run on the Worker Node.

## Check Service

```bash
kubectl get service
```

The Netflix Service should expose:

```text
80:30007/TCP
```

---

# 14. Jenkins → Kubernetes Integration

Kubernetes deployment was integrated directly into the Jenkins CI/CD pipeline.

After Jenkins:

1. Builds the application
2. Runs security scans
3. Builds the Docker image
4. Scans the Docker image
5. Pushes the image to DockerHub

Jenkins automatically deploys the new image to Kubernetes.

The Jenkins stage is:

```groovy
stage('Deploy to Kubernetes') {
    steps {
        withKubeConfig([credentialsId: 'k8s']) {

            sh 'kubectl apply -f Kubernetes/deployment.yml'

            sh 'kubectl apply -f Kubernetes/service.yml'

            sh 'kubectl set image deployment/netflix-app netflix-app=haja69245/netflix:${BUILD_NUMBER}'
        }
    }
}
```

---

# 15. Automatic Docker Image Update

Every Jenkins build creates a new Docker image.

For example:

```text
Jenkins Build #27
        │
        ▼
haja69245/netflix:27
```

A later build could create:

```text
Jenkins Build #28
        │
        ▼
haja69245/netflix:28
```

Jenkins pushes the image to DockerHub and executes:

```bash
kubectl set image deployment/netflix-app netflix-app=haja69245/netflix:${BUILD_NUMBER}
```

This updates the Kubernetes Deployment with the image generated by the current Jenkins build.

Flow:

```text
Source Code
    │
    ▼
Jenkins
    │
    ▼
Docker Build
    │
    ▼
haja69245/netflix:<BUILD_NUMBER>
    │
    ▼
DockerHub
    │
    ▼
kubectl set image
    │
    ▼
Kubernetes Deployment
    │
    ▼
New Netflix Pods
```

---

# 16. Jenkins Kubernetes Credentials

Jenkins uses a Kubernetes kubeconfig credential with the ID:

```text
k8s
```

It is used by:

```groovy
withKubeConfig([credentialsId: 'k8s'])
```

This allows Jenkins to execute `kubectl` commands against the Kubernetes cluster.

> ⚠️ The actual kubeconfig content and Kubernetes certificates are not stored in this repository.

---

# 17. AWS Security Groups

AWS Security Groups were configured to allow the required communication between the Control Plane and Worker Node.

One important Kubernetes port is:

```text
TCP 10250
```

Port `10250` is used by the **kubelet**.

The Worker Node Security Group allows port `10250` from the Kubernetes Control Plane Security Group.

---

# 18. Troubleshooting — Kubelet Port 10250

Initially, running Kubernetes commands such as Pod logs from the Control Plane produced an error similar to:

```text
dial tcp <WORKER-PRIVATE-IP>:10250: i/o timeout
```

The Control Plane could see the Worker Node, but it could not communicate with the Worker's kubelet.

The problem was caused by the AWS Security Group.

The solution was to allow:

```text
Protocol: TCP
Port: 10250
Source: Kubernetes Control Plane Security Group
```

on the Worker Node.

After adding the rule:

```bash
kubectl logs <pod-name>
```

worked successfully.

---

# 19. Kubernetes Monitoring

The Kubernetes infrastructure is also monitored using:

- Prometheus
- Node Exporter
- Grafana

Node Exporter runs on both:

```text
k8s-master
netflix-devsecops
```

Prometheus collects infrastructure metrics from the nodes.

---

## Control Plane Monitoring

Node Exporter runs on the Control Plane and exposes metrics on:

```text
<K8S_MASTER_PRIVATE_IP>:9100
```

Prometheus was configured to scrape these metrics.

Example:

```yaml
- job_name: 'k8s-master'
  static_configs:
    - targets: ['<K8S_MASTER_PRIVATE_IP>:9100']
```

> The private IP is represented as a placeholder in the public documentation because infrastructure addresses may change and do not need to be exposed in the repository.

---

# 20. Prometheus Targets

Prometheus currently monitors:

```text
Prometheus
    │
    ├── Jenkins
    │     └── localhost:8080/prometheus
    │
    ├── DevSecOps Worker
    │     └── localhost:9100
    │
    ├── Kubernetes Control Plane
    │     └── <K8S_MASTER_PRIVATE_IP>:9100
    │
    └── Prometheus
          └── localhost:9090
```

The configured monitoring targets reached the:

```text
UP
```

state.

---

# 21. Grafana Dashboard

Grafana uses Prometheus as its data source.

The **Node Exporter Full** dashboard is used to visualize Kubernetes node infrastructure metrics.

The dashboard provides information such as:

- CPU usage
- RAM usage
- Disk usage
- Network traffic
- System load
- Uptime

Monitoring flow:

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
      │
      ▼
Dashboards
```

---

# 22. Complete Kubernetes CI/CD Flow

```text
Developer
    │
    ▼
GitHub
    │
    ▼
Jenkins
    │
    ├── Install Dependencies
    │
    ├── SonarQube Analysis
    │
    ├── Quality Gate
    │
    ├── OWASP Dependency Check
    │
    ├── Trivy File System Scan
    │
    ├── Docker Build
    │
    ├── Trivy Image Scan
    │
    └── Docker Push
            │
            ▼
        DockerHub
            │
            ▼
       Kubernetes
            │
            ▼
        Deployment
            │
       ┌────┴────┐
       ▼         ▼
     Pod 1      Pod 2
       │         │
       └────┬────┘
            ▼
      NodePort Service
          :30007
            │
            ▼
          User
```

---

# 23. Kubernetes + Monitoring Architecture

```text
                        AWS VPC
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
       k8s-master                 netflix-devsecops
      Control Plane                 Worker Node
             │                           │
      Node Exporter                     ├── Jenkins
             │                           ├── SonarQube
             │                           ├── Prometheus
             │                           ├── Grafana
             │                           ├── Node Exporter
             │                           │
             │                           ├── Netflix Pod 1
             │                           └── Netflix Pod 2
             │
             └──────────┐   ┌────────────┘
                        ▼   ▼
                     Prometheus
                         │
                         ▼
                       Grafana
```

---

# 24. Useful Kubernetes Commands

### View nodes

```bash
kubectl get nodes
```

### View detailed node information

```bash
kubectl get nodes -o wide
```

### View Pods

```bash
kubectl get pods
```

### View Pod location

```bash
kubectl get pods -o wide
```

### View Deployments

```bash
kubectl get deployments
```

### View Services

```bash
kubectl get services
```

### View Pod logs

```bash
kubectl logs <pod-name>
```

### Describe a Pod

```bash
kubectl describe pod <pod-name>
```

### Describe a Node

```bash
kubectl describe node <node-name>
```

### Apply Deployment

```bash
kubectl apply -f Kubernetes/deployment.yml
```

### Apply Service

```bash
kubectl apply -f Kubernetes/service.yml
```

### Update Docker Image

```bash
kubectl set image deployment/netflix-app netflix-app=haja69245/netflix:<IMAGE_TAG>
```

---

# 25. What I Learned

Through this Kubernetes implementation, I learned how to:

- Create a Kubernetes cluster using `kubeadm`
- Understand Control Plane and Worker Node architecture
- Configure `containerd`
- Configure Kubernetes networking
- Use Flannel as a CNI
- Deploy an application using a Kubernetes Deployment
- Manage multiple Pod replicas
- Expose an application using a NodePort Service
- Integrate Jenkins with Kubernetes
- Automatically deploy new Docker images
- Configure AWS Security Groups for Kubernetes
- Troubleshoot kubelet communication
- Monitor Kubernetes infrastructure using Prometheus and Grafana

---

# 26. Final Result

The Netflix Clone is successfully deployed on a Kubernetes cluster running on AWS EC2.

The application runs with:

```text
2 Netflix Pods
      │
      ▼
NodePort Service
      │
      ▼
Port 30007
```

The Kubernetes infrastructure is integrated with the Jenkins CI/CD pipeline and monitored using **Prometheus + Grafana**.

---

## 👩‍💻 Author

**Hajar Azaou**

Software Engineering Student  
Backend • DevOps • Cloud
