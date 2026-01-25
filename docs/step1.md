# Creazione VM K3s Master con Terraform su Proxmox
Obiettivo

Creare una VM K3s Master su un nodo Proxmox fisico (x86, Lenovo E73), partendo da un template Ubuntu Server già esistente, usando Terraform, in modo da poter poi configurare un cluster Kubernetes ibrido con un worker ARM (Raspberry Pi).
'''
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
'''
Creazione VM K3s Master (VMID 112)
1️⃣ Crea la VM base
qm create 112 \
  --name k3s-master \
  --memory 4096 \
  --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --ostype l26


VMID 112 nuovo, 4 GB RAM, 2 core

virtio network bridge su vmbr0

2️⃣ Importa immagine Ubuntu Cloud corretta

Scarica immagine Ubuntu Jammy cloud se non l’hai già fatto:

wget -P /var/lib/vz/template/iso/ https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img


Importa l’immagine nel disco della VM:

qm importdisk 112 /var/lib/vz/template/iso/jammy-server-cloudimg-amd64.img local-lvm

3️⃣ Configura disco principale e dimensione
qm set 112 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-112-disk-0,size=20G


Dispositivo principale SCSI da 20 GB (minimo per K3s + Ubuntu)

4️⃣ Configura cloud-init
qm set 112 --ide2 local-lvm:cloudinit
qm set 112 --ciuser ubuntu --cipassword porcodio


Crea utente ubuntu con password MioPasswordTemp123

Ora puoi loggarti subito senza chiavi SSH

5️⃣ Abilita agent, console VGA e boot
qm set 112 --agent enabled=1
qm set 112 --serial0 socket --vga std
qm set 112 --boot c --bootdisk scsi0


VGA standard evita il problema della seriale bloccata

QEMU agent abilitato per gestione K3s + cloud-init

6️⃣ Avvia la VM
qm start 112


Apri Proxmox Web GUI → VM → Console → VGA

Login: ubuntu

Password: porcodio



# la vm non ha ip quindi glielo diamo manualmente..
sudo nano /etc/netplan/50-cloud-init.yaml

network:
    version: 2
    ethernets:
        ens18:
            dhcp4: no
            addresses: [192.168.1.30/24]
            gateway4: 192.168.1.1
            nameservers:
                addresses: [192.168.1.1,8.8.8.8]

sudo netplan apply

verifichiamo:
ip a
ping 8.8.8.8
ping <ip raspberry>

## Installa K3s Master senza Traefik

Esegui sulla VM Master:

curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -

--disable=traefik perché userai Caddy sul Raspberry come reverse proxy esterno

Questo comando:

- Installa K3s Master
- Configura il service systemd (k3s.service)
- Installa kubectl locale configurato per il cluster

sudo k3s kubectl get nodes

## Recupera il token K3s per i worker

Il token serve ai nodi worker per unirsi al cluster. Generalo così:

sudo cat /var/lib/rancher/k3s/server/node-token

vm master su proxmox deve avere disco da almeno 40 gb!! altrimenti comando qui sotto fallisce.. 
# su raspberry nodo worker 

sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab    #impedisce che lo swap si riattivi al riavvio.
Dopo questo, il Raspberry sarà pronto per installare K3s come worker senza rischi.

curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.30:6443 K3S_TOKEN=************************************* sh -

# dato che su raspberry non riesco a registrare il worker , provo a creare una vm worker su proxmox 

Creazione nuova VM worker 113 (Ubuntu Server con password)

Qui useremo la Ubuntu cloud-image, ma con cloud-init abilitato così possiamo impostare subito un utente e password a tua scelta.

# Crea la VM
qm create 113 \
  --name k3s-worker \
  --memory 4096 \
  --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --ostype l26 \
  --boot c \
  --bootdisk scsi0 \
  --scsihw virtio-scsi-pci \
  --serial0 socket \
  --vga std

3️⃣ Importa disco Ubuntu cloud-image

Assicurati di avere l’immagine:

wget -P /var/lib/vz/template/iso/ https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img


Poi importa il disco e allocalo su local-lvm (disco grande 40 GB):

qm importdisk 113 /var/lib/vz/template/iso/jammy-server-cloudimg-amd64.img local-lvm
qm set 113 --scsi0 local-lvm:vm-113-disk-0,size=40G

4️⃣ Configura Cloud-init e utente
qm set 113 --ide2 local-lvm:cloudinit
qm set 113 --ciuser ubuntu
qm set 113 --cipassword porcodio
qm set 113 --agent enabled=1

Questo creerà un utente ubuntu con password porcodio e potrai fare login da console o SSH.

a vm non ha ip quindi glielo diamo manualmente..
sudo nano /etc/netplan/50-cloud-init.yaml

network:
    version: 2
    ethernets:
        ens18:
            dhcp4: no
            addresses: [192.168.1.31/24]
            gateway4: 192.168.1.1
            nameservers:
                addresses: [192.168.1.1,8.8.8.8]

sudo netplan apply
# su worker 
curl -sfL https://get.k3s.io | \
K3S_URL=https://192.168.1.30:6443 \
K3S_TOKEN=*******************::server:****************** \
sh -


fallisce perchè mi ha creato una vm con disco minucolo da 2 gb e va aumentato .. diocane!

sudo cat /var/lib/rancher/k3s/server/node-token  per vedere il token master
sudo k3s kubectl get nodes   per vedere i nodi dal master

