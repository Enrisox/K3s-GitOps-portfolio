# Creazione VM K3s Master con Terraform su Proxmox
Obiettivo

Creare una VM K3s Master su un nodo Proxmox fisico (x86, Lenovo E73), partendo da un template Ubuntu Server già esistente, usando Terraform, in modo da poter poi configurare un cluster Kubernetes ibrido con un worker ARM (Raspberry Pi).

Prerequisiti

Nodo Proxmox accessibile e funzionante (Proxmox)

Template Ubuntu Server già presente su Proxmox: ubuntu-template

Provider Terraform Proxmox funzionante in un progetto Terraform esistente

SSH key pubblica disponibile sul workstation/host: /home/enrico/.ssh/id_rsa.pub

IP statico disponibile per la VM master: 192.168.1.30

Storage LVM per dischi VM: local-lvm

Da dentro wsl Ubuntu terminal su pc fisso:

mkdir -p ~/lab-proxmox/k3s-cluster
cd ~/lab-proxmox/k3s-cluster
nano k3s-master.tf

resource "proxmox_vm_qemu" "k3s_master" {
  name        = "k3s-master"
  target_node = "Proxmox"          # il tuo nodo fisico
  clone       = "ubuntu-template"   # il template esatto

  cores  = 2
  sockets = 1
  memory = 4096

  scsihw   = "virtio-scsi-pci"
  bootdisk = "scsi0"

  disk {
    slot    = 0
    size    = "40G"
    type    = "scsi"
    storage = "local-lvm"         # storage LVM per prestazioni
  }

  network {
    model  = "virtio"
    bridge = "vmbr0"
  }

  os_type = "cloud-init"

  ciuser  = "ubuntu"
  sshkeys = file("~/.ssh/id_rsa.pub")

  ipconfig0 = "ip=192.168.1.30/24,gw=192.168.1.1"
  nameserver = "192.168.1.1"
}

terraform init
terraform plan

Mostra una sola VM da creare
Conferma che non tocca VM esistenti

**Applicazione del piano:**

terraform apply

- Confermato digitando yes
- Terraform clona il template e crea la VM master con cloud-init

**Stato finale**

VM k3s-master creata su nodo Proxmox
IP statico: 192.168.1.30
Utente ubuntu con chiave SSH installata
Disco: 40GB su local-lvm
RAM: 4Gb, CPU: 2 core

terraform apply

