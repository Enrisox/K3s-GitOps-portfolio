# Portfolio Website: Stateless Application on K3s

The goal of this phase is to implement a stateless application to test load balancing between x86 nodes (VMs) and ARM nodes (Raspberry Pi), while experimenting with Kubernetes' **Self-Healing** and **Scalability** logic.

## Creating index.html on the Master Node (Proxmox)

The static content is initially created on the master node:

```bash
mkdir -p ~/portfolio
cd ~/portfolio
nano index.html

```

## Content Abstraction: Kubernetes ConfigMap

Instead of the traditional workflow (building a custom Docker image for every minor change), I opted for a **ConfigMap**.
A ConfigMap is a Kubernetes internal database that stores data in key-value pairs.
By using this, I can update the site quickly without managing a Container Registry or redeploying the image. Kubernetes "injects" the `index.html` file into the pods as if it were a physical volume, regardless of the node's architecture (Master or Raspberry Pi).

**Advantages of this approach:**

* **Decoupling:** HTML code is separated from the Nginx infrastructure.
* **Agility:** Rapid updates without rebuilding images.
* **Automatic Distribution:** Seamless injection across multi-architecture nodes.

```bash
kubectl create configmap portfolio-html --from-file=index.html

```

## Creating the Deployment (portfolio.yaml)

In Kubernetes, a **Deployment** declaratively describes how an application should run: how many replicas to have, which image to use, and how to manage updates.

**Key functions of a Deployment:**

* **Desired State:** Maintains the specified number of Pods (e.g., 2 replicas).
* **Rolling Updates:** Gradually replaces old Pods with new versions to ensure zero downtime.
* **Self-Healing:** Automatically restarts failed Pods.
* **Scaling:** Easy up/down scaling by modifying the `replicas` field.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portfolio
spec:
  replicas: 2
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
      nodePort: ***** # Specify your desired port

```

Apply the configuration:

```bash
kubectl apply -f portfolio.yaml

```

## Security and Resource Analysis

### 1. Pod Security

I implemented several rules to follow the **Principle of Least Privilege**:

* **Nginx-unprivileged:** Unlike standard Nginx, this version runs on port 8080, allowing the container to function without root privileges.
* **Alpine Base:** A minimal image that reduces the attack surface and minimizes CVEs.
* **Rootless Execution:** The container runs with UID 101, preventing administrative access within the pod.
* **Drop Capabilities:** Removed all Linux capabilities.
* **Seccomp Profile:** Enabled the `RuntimeDefault` filter to block dangerous kernel calls.

### 2. Filesystem Hardening

* **Read-Only Root Filesystem:** Prevents attackers from downloading malware or creating backdoors.
* **EmptyDir Volumes:** Since Nginx needs to write temporary PID and cache files, I mounted `emptyDir` volumes on `/tmp` and `/var/cache/nginx` to allow writes only in these specific, temporary locations.

### 3. Resource Management (Anti-DoS)

* **CPU Limits:** Max 0.5 core (500m).
* **RAM Limits:** Max 256 MiB.
Setting these limits ensures that if one Pod is overwhelmed, it won't crash the entire physical host or starve other services.

### 4. Network & Client Security (Caddy Headers)

To harden the public-facing side, Caddy is configured with security headers:

* **X-Frame-Options "DENY":** Prevents Clickjacking.
* **HSTS:** Forces HTTPS communication.
* **Content-Type-Options "nosniff":** Prevents MIME-type sniffing.

## Caddy Reverse Proxy Configuration

On the worker node, I updated the Caddyfile to proxy traffic to the Kubernetes Service.

```text
portfolio.enrisox-devops.it {
    reverse_proxy 192.168.1.XX:port
}

```


Restart Caddy
```text
docker restart caddy
```

## DNS and Verification

A new DNS **A Record** was created pointing to my public IP (managed via a dockerized DDNS service).

**Verification commands:**
This shows the pods distributed across the different node architectures (x86 and ARM).

```bash
kubectl get pods -o wide

```


## Update Procedures

### A. Updating index.html (Content only)

To update the site content with zero downtime:

1. Edit "index.html".
2. Update the ConfigMap and trigger a rolling restart:

```bash
kubectl create configmap portfolio-html --from-file=index.html --dry-run=client -o yaml | kubectl apply -f - && kubectl rollout restart deployment portfolio
```

### B. Updating Deployment Configuration

If changing replicas, Docker images, or resource limits:

1. Edit `portfolio.yaml`.
2. Apply changes:

```bash
kubectl apply -f portfolio.yaml
```

Check the status:

```bash
kubectl rollout status deployment portfolio
```

This setup ensures a highly secure, scalable, and resilient portfolio hosting environment using a mix of virtual and physical hardware.

