# 📊 Monitoring — Prometheus & Grafana

This document describes the monitoring infrastructure implemented for the **Netflix Clone DevSecOps CI/CD project**.

The monitoring stack uses:

- Prometheus
- Grafana
- Node Exporter
- Jenkins Prometheus Plugin

The goal is to monitor the DevSecOps server, Jenkins, and the Kubernetes Control Plane from a centralized Prometheus instance.

---

# 1. Monitoring Architecture

Prometheus and Grafana run on the main DevSecOps EC2 instance.

Node Exporter runs on both the DevSecOps/Worker server and the Kubernetes Control Plane.

```text
                         AWS VPC
                            │
          ┌─────────────────┴─────────────────┐
          │                                   │
          ▼                                   ▼
  netflix-devsecops                       k8s-master
   Worker Node                         Control Plane
          │                                   │
          ├── Jenkins                         └── Node Exporter
          ├── Prometheus                            │
          ├── Grafana                               │
          └── Node Exporter                         │
                  │                                 │
                  └────────────┬────────────────────┘
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

# 2. Prometheus

Prometheus is used to collect and store metrics from the infrastructure.

Prometheus runs on the DevSecOps EC2 instance and listens on:

```text
Port 9090
```

Prometheus can be accessed using:

```text
http://<DEVSECOPS-PUBLIC-IP>:9090
```

> Public IP addresses are represented as placeholders because EC2 public IPs may change.

---

# 3. Prometheus Configuration

The Prometheus configuration is located at:

```text
/etc/prometheus/prometheus.yml
```

The configuration contains multiple monitoring targets.

The main targets used in this project are:

```yaml
scrape_configs:

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['localhost:8080']

  - job_name: 'k8s-master'
    static_configs:
      - targets: ['<K8S_MASTER_PRIVATE_IP>:9100']
```

This allows Prometheus to collect metrics from:

```text
Prometheus itself
DevSecOps Worker Node
Jenkins
Kubernetes Control Plane
```

---

# 4. Validate Prometheus Configuration

Before restarting Prometheus, the configuration was validated using:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

A valid configuration returns:

```text
SUCCESS: /etc/prometheus/prometheus.yml is valid prometheus config file syntax
```

After changing the configuration, Prometheus can be restarted:

```bash
sudo systemctl restart prometheus
```

The service status can be checked using:

```bash
sudo systemctl status prometheus
```

---

# 5. Node Exporter

Node Exporter exposes Linux system metrics that Prometheus can collect.

It provides metrics such as:

- CPU usage
- Memory usage
- Disk usage
- Network activity
- System load
- Filesystem information
- System uptime

Node Exporter listens on:

```text
Port 9100
```

---

# 6. Worker Node Monitoring

Node Exporter runs on the main DevSecOps EC2 instance.

Because Prometheus is running on the same server, the target is:

```yaml
- job_name: 'node'
  static_configs:
    - targets: ['localhost:9100']
```

The architecture is:

```text
DevSecOps Worker
       │
       ├── Node Exporter :9100
       │
       ▼
   Prometheus :9090
```

---

# 7. Kubernetes Control Plane Monitoring

Node Exporter was also installed on the Kubernetes Control Plane.

The Kubernetes Control Plane uses its private AWS network address for communication with Prometheus.

Prometheus configuration:

```yaml
- job_name: 'k8s-master'
  static_configs:
    - targets: ['<K8S_MASTER_PRIVATE_IP>:9100']
```

This allows Prometheus to monitor the Kubernetes Control Plane without requiring Node Exporter port `9100` to be exposed publicly.

---

# 8. Verify Node Exporter Connectivity

From the DevSecOps/Worker server, connectivity to the Kubernetes Control Plane Node Exporter can be tested using:

```bash
curl http://<K8S_MASTER_PRIVATE_IP>:9100/metrics | head
```

A successful response displays Node Exporter metrics.

A message such as:

```text
curl: (23) Failure writing output to destination
```

can appear when piping the command to `head`.

In this case, it does not indicate a Node Exporter connection failure. `head` closes the output stream after reading the requested lines.

---

# 9. Jenkins Monitoring

Jenkins is also monitored by Prometheus.

The **Prometheus Metrics Plugin** was configured in Jenkins.

It exposes Jenkins metrics through:

```text
http://localhost:8080/prometheus
```

Prometheus configuration:

```yaml
- job_name: 'jenkins'
  metrics_path: '/prometheus'

  static_configs:
    - targets: ['localhost:8080']
```

This allows Prometheus to collect Jenkins metrics.

---

# 10. Prometheus Targets

Prometheus provides a Targets page that shows whether monitored services are reachable.

The monitoring targets configured in this project are:

```text
jenkins
k8s-master
node
prometheus
```

The final state was:

```text
jenkins       → UP
k8s-master    → UP
node          → UP
prometheus    → UP
```

Architecture:

```text
                    Prometheus
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
     Jenkins       Worker Node      k8s-master
     :8080             :9100            :9100
```

---

# 11. Grafana

Grafana is used to visualize the metrics collected by Prometheus.

Grafana runs on the DevSecOps EC2 instance and listens on:

```text
Port 3000
```

It can be accessed using:

```text
http://<DEVSECOPS-PUBLIC-IP>:3000
```

---

# 12. Connect Grafana to Prometheus

Prometheus was configured as the Grafana data source.

Because Grafana and Prometheus run on the same EC2 instance, the Prometheus URL is:

```text
http://localhost:9090
```

The monitoring flow becomes:

```text
Infrastructure
      │
      ▼
Node Exporter / Jenkins Metrics
      │
      ▼
Prometheus
      │
      ▼
Grafana
```

---

# 13. Grafana Node Exporter Dashboard

The **Node Exporter Full** Grafana dashboard was imported.

Dashboard ID:

```text
1860
```

The dashboard displays infrastructure information such as:

```text
CPU
RAM
Disk
Network
System Load
Uptime
```

For the Kubernetes Control Plane, the dashboard can use:

```text
Job      → k8s-master
Instance → <K8S_MASTER_PRIVATE_IP>:9100
```

---

# 14. Monitoring Flow

The complete monitoring flow is:

```text
Netflix DevSecOps Infrastructure
              │
              ├─────────────────────────┐
              │                         │
              ▼                         ▼
        Worker Node                k8s-master
              │                         │
       Node Exporter               Node Exporter
          :9100                       :9100
              │                         │
              └──────────┬──────────────┘
                         │
                         ▼
                     Prometheus
                        :9090
                         │
                         ▼
                       Grafana
                        :3000
                         │
                         ▼
                     Dashboards
```

Jenkins is monitored separately by its Prometheus metrics endpoint:

```text
Jenkins
  │
  │ /prometheus
  ▼
Prometheus
  │
  ▼
Grafana
```

---

# 15. Monitoring Ports

| Service | Port | Purpose |
|---|---:|---|
| Jenkins | 8080 | Jenkins web interface and metrics |
| Prometheus | 9090 | Metrics collection and querying |
| Grafana | 3000 | Monitoring dashboards |
| Node Exporter | 9100 | Linux infrastructure metrics |

Node Exporter does not need to be publicly exposed when communication happens through the AWS private network.

---

# 16. Security Considerations

The monitoring infrastructure follows several basic security practices.

### Node Exporter

Port `9100` should not be exposed publicly unless necessary.

Prometheus can communicate with the Kubernetes Control Plane using the AWS private network.

### Jenkins

Jenkins should not expose credentials through Prometheus metrics or pipeline configuration.

### Grafana

Grafana access should be protected with authentication.

### AWS Security Groups

Only required ports should be allowed, and administrative access such as SSH should be restricted to trusted IP addresses.

---

# 17. Useful Monitoring Commands

Check Prometheus:

```bash
sudo systemctl status prometheus
```

Restart Prometheus:

```bash
sudo systemctl restart prometheus
```

Validate Prometheus configuration:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

Check Node Exporter:

```bash
sudo systemctl status prometheus-node-exporter
```

Test local Node Exporter:

```bash
curl http://localhost:9100/metrics | head
```

Test Kubernetes Control Plane Node Exporter:

```bash
curl http://<K8S_MASTER_PRIVATE_IP>:9100/metrics | head
```

Check Prometheus targets from the Prometheus web interface:

```text
Status → Targets
```

All configured targets should display:

```text
UP
```

---

# 18. Monitoring Result

The final monitoring architecture successfully monitors:

- ✅ DevSecOps EC2 infrastructure
- ✅ Kubernetes Control Plane
- ✅ Jenkins
- ✅ Prometheus
- ✅ CPU metrics
- ✅ Memory metrics
- ✅ Disk metrics
- ✅ Network metrics
- ✅ System uptime

The metrics are collected by **Prometheus** and visualized using **Grafana**.

---

# 19. What I Learned

Through this monitoring implementation, I learned how to:

- Install and configure Prometheus
- Understand the Prometheus pull model
- Configure Prometheus scrape targets
- Validate Prometheus configuration using `promtool`
- Install and configure Node Exporter
- Monitor multiple AWS EC2 instances
- Monitor Jenkins using Prometheus
- Connect Grafana to Prometheus
- Import and use Grafana dashboards
- Monitor Kubernetes node infrastructure
- Troubleshoot monitoring connectivity
- Use private AWS networking for monitoring

---

# 20. Final DevSecOps Monitoring Architecture

```text
GitHub
   │
   ▼
Jenkins ─────────────────────────────┐
   │                                │
   ▼                                │ metrics
CI/CD Pipeline                      │
   │                                ▼
   ▼                            Prometheus
DockerHub                            ▲
   │                                │
   ▼                                ├── Worker Node
Kubernetes                           ├── k8s-master
   │                                └── Jenkins
   ▼                                │
Netflix Pods                        ▼
                                 Grafana
                                    │
                                    ▼
                                Dashboards
```

---

## 👩‍💻 Author

**Hajar Azaou**

Software Engineering Student  
Backend • DevOps • Cloud