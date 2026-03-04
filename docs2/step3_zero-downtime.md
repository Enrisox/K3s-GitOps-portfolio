# Zero-Downtime Deployment with Rolling Updates and Health Probes

This step describes how Kubernetes ensures application updates **without service interruption**, utilizing:

* Rolling Updates
* Readiness Probes
* Liveness Probes
* Controlled Rollouts

## Objective

During an update:

* At least one Pod remains available at all times.
* New Pods receive traffic only when fully ready.
* Defective Pods are automatically restarted.

## Rolling Update Strategy

The following configuration is applied to the Deployment:

* **maxUnavailable: 0** → No Pod is terminated until the new one is marked as **Ready**. This guarantees zero downtime.
* **maxSurge: 1** → Kubernetes can temporarily create an extra Pod during the transition, ensuring capacity is always at >= 100%.

### Key Features Added to `portfolio.yaml`:

* **Readiness/Liveness Probes**: Tests when a Pod can receive traffic to avoid **502 errors** during rollouts and automatically detects/restarts hung Pods (**Self-healing**).
* **Stakater Reloader** (Optional automation): Fully automates the rollout when the ConfigMap changes.
* **kubectl rollout history**: Enables rapid rollbacks in case of issues.
* **Metrics Server + kubectl top**: Real-time resource monitoring to validate limits.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portfolio
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0  # Never shut down a Pod until the new one is Ready
      maxSurge: 1        # Allow 1 extra Pod during rollout (max 3 temporary)
  selector:
    matchLabels:
      app: portfolio
  template:
    metadata:
      labels:
        app: portfolio
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        runAsGroup: 101
        fsGroup: 101
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: nginx
        image: nginxinc/nginx-unprivileged:alpine
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          privileged: false
          capabilities:
            drop:
              - ALL
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 5
          failureThreshold: 2
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          failureThreshold: 3
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
        - name: tmp-volume
          mountPath: /tmp
        - name: cache-volume
          mountPath: /var/cache/nginx
      volumes:
      - name: html-volume
        configMap:
          name: portfolio-html
      - name: tmp-volume
        emptyDir: {}
      - name: cache-volume
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: portfolio-service
spec:
  selector:
    app: portfolio
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: ***** # Replace with your designated port

```

## Applying Changes

```bash
kubectl apply -f portfolio.yaml

```

Kubernetes will execute a progressive update to activate the new Probes and Strategies without interrupting the service.

## Testing Strategies and Probes

Verify that the Zero-Downtime strategy and control sentinels (Readiness and Liveness) are active:

```bash
# Check Rolling Update configuration
kubectl describe deployment portfolio | grep -A 3 "RollingUpdate"

# Check Probes status on Pods
kubectl describe pod -l app=portfolio | grep -E "Readiness|Liveness"

# Verify running Pods
kubectl get pods -l app=portfolio

```

## Rollout Management

In Kubernetes, modifying a **ConfigMap** does not automatically notify running Pods (as they have already loaded the file into memory). To apply changes without downtime, we use the **Rollout Restart** command.

This command instructs Kubernetes to recreate all Pods in the deployment following the defined strategy:

```bash
kubectl rollout restart deployment portfolio && kubectl rollout status deployment portfolio

```

### Background Process Breakdown:

1. **Creation (Surge)**: Thanks to `maxSurge: 1`, Kubernetes creates a third updated Pod.
2. **Readiness Test**: The new Pod is put on "probation." Only after the `readinessProbe` succeeds (the site responds), the Pod is considered ready.
3. **Replacement**: Kubernetes begins terminating one of the old Pods.
4. **Continuity**: The procedure repeats until all old Pods are replaced.
5. **Final Result**: Throughout the process, traffic managed by Caddy is routed only to healthy Pods. The user experiences **Zero Downtime**.

