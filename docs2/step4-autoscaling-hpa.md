# Autoscaling with Horizontal Pod Autoscaler (HPA)

In this step, we explore how Kubernetes can automatically scale the number of replicas in a Deployment based on resource consumption.

```bash
kubectl autoscale deployment portfolio --cpu-percent=50 --min=2 --max=5

kubectl get hpa -w     # To monitor HPA status in real-time

```

## Objective

* **Increase** the number of Pods when the application receives higher traffic or consumes more resources.
* **Decrease** the number of Pods during low-load periods to save cluster resources.
* **Ensure continuity and resilience** without additional downtime.

## How HPA Works

The HPA relies on the **Metrics Server**, which monitors the resources used by Pods (CPU, RAM, or custom metrics).

### The HPA Control Loop

1. **Monitoring** - Every 15 seconds (default), the HPA queries the Metrics Server to measure the CPU and memory usage of the Pods.
2. **Calculation** - It compares current usage against the defined target.
* *Example:* Target CPU = 50%, current Pod consumption = 80% → HPA detects high load.


3. **Action** - If load exceeds the target, HPA increases the number of replicas.
* If load drops, it scales the replicas back down.
* This process is seamless and zero-downtime thanks to the **RollingUpdate** strategy defined in Step 3.



## Implementing HPA for this Project

**Creating an HPA for the "portfolio" Deployment:**

```bash
kubectl autoscale deployment portfolio \
  --cpu-percent=50 \
  --min=2 \
  --max=5

```

## GitOps Integration (ArgoCD)

To demonstrate a full **GitOps workflow**, I update the `index.html` within the `configmap.yaml` and modify the `deployment.yaml` directly in the repository. This triggers an automatic **ArgoCD refresh**, ensuring my web portfolio is updated and the HPA configuration is applied as part of the desired state.
