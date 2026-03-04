# K3s GitOps Cluster: Portfolio Project

![Demo portfolio](imgs/portfolio-enrico.gif)

This repository documents my first hands-on experience with Kubernetes. Ahead of the classroom lessons scheduled for the coming weeks, I decided to get a head start by independently exploring the potential of K3s (a lightweight K8s distribution) to orchestrate my personal portfolio.

The goal is to create a hybrid, scalable Cloud Native infrastructure managed entirely through GitOps logic. The cluster has been optimized for edge computing by removing the default ingress controller (Traefik) to centralize traffic control via an external Caddy instance.

## Architecture
- **Control Plane (Master)**: Ubuntu Server VM on Proxmox.
- **Worker Node**: Ubuntu Server on Raspberry Pi 5.
- **Edge Proxy**: Caddy (external to the cluster) managing SSL certificates and HTTPS routing.
- **GitOps**: ArgoCD for automated deployment from the repository to the cluster.

## Tech Stack
- **Orchestrator**: K3s.
- **Reverse Proxy**: Caddy.
- **Continuous Delivery**: ArgoCD.
- **Networking**: Service exposure via NodePort.

## Traffic Flow
- User requests https://portfolio.enrisox-devops.it
- Caddy intercepts the request, manages TLS, and acts as a Reverse Proxy.
- Traffic is forwarded to the K3s cluster nodes on the specific NodePort.
- The internal service routes the packets to the Portfolio Pods.

## Project Roadmap
I have divided the configuration into logical modules to ensure replicability and ease of maintenance:

1.  [**Step 1: Installazione K3s**](docs2/step1_K3s.md): Master configuration, Worker join (Raspberry Pi), and resource optimization.
2.  [**Step 2: Deployment & Networking**](docs2/step2_deployment.md): Portfolio configuration, NodePort Service management, and Caddy setup.
3.  [**Step 3: Zero Downtime**](docs2/step3_zero-downtime.md): Update strategies (Rolling Update) to guarantee service continuity.
4.  [**Step 4: Autoscaling (HPA)**](docs2/step4-autoscaling-hpa.md): Horizontal Pod Autoscaler configuration based on CPU utilization.
5.  [**Step 5: GitOps con ArgoCD**](docs2/step5-gitops-argocd.md): ArgoCD installation and connection to my private repository for Continuous Deployment.

## ⚠️ Home Lab Disclaimer

As this is a test environment based on limited and shared hardware resources (Home Lab), please note the following:

- **Resource Management**: The Portfolio site may experience slowdowns if computing resources are temporarily diverted to other projects or containers running on the same infrastructure.
- **High Availability (HA)**: The cluster is configured with a Master/Worker architecture. While Pods on the Worker node continue to follow the last known state, any shutdown or maintenance of the Master node (Proxmox VM) may cause temporary errors during page loading. These anomalies are not due to configuration bugs or code errors, but solely to the hardware availability of my personal lab.

In the coming months, I will integrate new features and increase the complexity of the project/cluster: STAY TUNED!

If you liked this project, please leave a ⭐!

**Enrico Soci - DevSecOps**

