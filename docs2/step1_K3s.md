# Portfolio Website Hosted on a K3s Kubernetes Cluster

This project describes the creation of a **hybrid K3s infrastructure**, ranging from a **Proxmox** server hosting an Ubuntu Server VM as the Master Node to a **Raspberry Pi 5 (Worker Node)**.

**Hybrid at two levels:**

* **Architecture**: Master Node on a Proxmox VM (x86_64/AMD64 architecture) and Worker Node on a Raspberry Pi (ARM64 architecture).
* **Platforms**: Virtual and physical.

## Why K3s?

**K3s is a lightweight Kubernetes (K8s) distribution. It was created by Rancher Labs to be easily installed on resource-constrained devices, such as a Raspberry Pi, or for rapid development and testing environments.**

Key features:

1. **Cloud-Init Automation**: Rapid setup of the Master VM via templates and SSH key/Network injection.
2. **K3s Optimization**: Lightweight installation without Traefik to delegate traffic management to an external Caddy instance.
3. **Raspberry Pi Configuration**: Enabling cgroups to allow the Worker Node to join the cluster.
4. **Hardened Deployment**: Publishing a static site via ConfigMap and Nginx, with a strong focus on security (read-only filesystem, non-root user, CPU/RAM limits) to protect the underlying hardware from attacks or overloads.
5. **Secure Exposure**: Using Caddy as a Reverse Proxy with advanced security headers and DNS management for public access via HTTPS.

## Infrastructure Setup: Proxmox and K3s

To create the Master VM, I used a Proxmox VM template previously created in another project: --> [DevOps-CI-CD-Lab-Jenkins-Terraform-Ansible](https://github.com/Enrisox/DevOps-CI-CD-Lab-Jenkins-Terraform-Ansible).

I then performed an over-provisioning of the original template's resources (CPU and RAM), ensuring the Master Node has the computing power required by the Kubernetes control plane.

## Master VM Creation

```bash
qm clone 100 103 --name k3s-master --full && \      
qm set 103 --cores 2 --memory 4096 --cpu host && \
qm resize 103 scsi0 +32G

```

* **qm clone 100 103**: Creates an exact copy of the VM with ID 100 (the pre-configured template) and assigns the new VM ID 103.
* **--name k3s-master**: Renames the new VM to "k3s-master" for easy identification.
* **--full**: This is the important part. It creates a **Full Clone**, meaning it physically copies the entire disk. The new VM will be completely independent of the original template.

Instead of powering on the VM and manually configuring IP, users, and passwords via the console terminal, I used **Cloud-Init** to "inject" these settings from the outside during the boot process.

To pass the key via command, Proxmox must read a file.

On the Proxmox shell:

```bash
nano /root/chiave.pub

```

### Cloud-Init Configuration

**Configure the Cloud-Init drive by changing IP, user, and password:**

```bash
qm set 103 --ide2 local-lvm:cloudinit \
--ciuser ubuntu \
--cipassword "************" \
--sshkeys /root/chiave.pub \
--ipconfig0 ip=192.168.1.X/24,gw=192.168.1.1 \
--agent enabled=1

```

**Regenerate and start the Master VM:**

```bash
qm cloudinit update 103
qm start 103

```

## Master VM Configuration

After accessing the VM via SSH using key authentication, I updated the system and installed the agent on the K3s Master Node (previously it was only enabled, not installed):

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install qemu-guest-agent -y

```

## Disable the Ubuntu Firewall (Test Environment Only)

* **Iptables**: K3s installs specific rules for container-to-container communication. If UFW is active, it might overwrite or block packets passing through the K3s virtual interface (cni0 or flannel), isolating your Pods.
* **Required Ports**: K3s requires several open ports (6443 for the API, 10250 for the Kubelet, VXLAN traffic ports, etc.). To avoid missing specific ports and troubleshooting logs, the host firewall is disabled, as security should be managed at a higher level (e.g., Cloud Security Groups or router firewall).

```bash
sudo ufw disable

```

## Installing K3s (Without Traefik)

I chose to disable **Traefik** (the default K3s Reverse Proxy) because my homelab infrastructure already included a **Caddy container** and I used **NodePort** for traffic management. Leaving Traefik active would have caused port conflicts.

**NodePort**: A Kubernetes configuration object that exposes a service on a static port. It opens a specific port (range 30000-32767) on all network interfaces of every node in the cluster (Master and Worker). It maps an external port to the internal container port (targetPort), making the service reachable via the IP of the physical machine/VM.

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -

```

**By default, k3s requires sudo for every kubectl command. To use kubectl without sudo, run:**

```bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml

```

### Enable QEMU Agent on the Master VM

```bash
sudo systemctl enable qemu-guest-agent
sudo systemctl start qemu-guest-agent
sudo systemctl status qemu-guest-agent      

```

## Worker Node Setup on Raspberry Pi

### Copy the token to be used on the Raspberry Pi:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token           

```

## Cgroups Configuration

**Raspberry Pi often has "cgroups" disabled by default. K3s will not start without them.**

```bash
cat /boot/firmware/cmdline.txt

```

What to look for? At the end of the line, it must say: **`cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory`**

If they are MISSING:

```bash
sudo nano /boot/firmware/cmdline.txt
sudo reboot

```

### Join the Raspberry Pi to the Cluster

After the reboot:

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.X:6443 K3S_TOKEN="K10***********token::server:token****************" sh -

```

### To remove sudo permanently for kubectl:

```bash
#Proxmox resets permissions on reboot
sudo chmod 644 /etc/rancher/k3s/k3s.yaml       
#Permanent solution
sudo chown userX:userX /etc/rancher/k3s/k3s.yaml        
```

**Verify from the Master Node:**

```bash
sudo kubectl get nodes

```

As shown in the image, both nodes are now **Ready**.


