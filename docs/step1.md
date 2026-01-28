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

**Etichettiamo il worker**

Serve per schedulare roba solo lì (best practice).

Sul master:

sudo k3s kubectl label node k3s-worker node-role.kubernetes.io/worker=worker
----------------------------------------------

Cos'è kubectl (e Perché sudo k3s kubectl)
kubectl è il programma (CLI) che invii ordini al cluster (tipo "crea pod", "lista nodi").

Su K3s, si chiama k3s kubectl (non solo kubectl), e serve sudo per i permessi.

Crea i File YAML (Sul Master)
YAML sono file di configurazione "umani" per dire a K8s "cosa voglio". Crea 2 file con nano (editor semplice).

mkdir k3s-lab          # Crea folder
cd /k3s-lab             # Entra (prompt diventa ~/k3s-lab$)
nano web-app-deployment.yaml 


apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
Cosa fa: Crea 1 "pod" (contenitore nginx che gira web server su porta 80 interna). Gestisce riavvii automatici.

nano web-app-service.yaml


apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
Cosa fa: "Apre un tunnel" dal mondo esterno (porta 30080 su IP master/worker) al pod interno (porta 80).

Salva: Ctrl+O, Enter, Ctrl+X.

4. Applica i File (Sul Master)
bash
sudo k3s kubectl apply -f web-app-deployment.yaml
sudo k3s kubectl apply -f web-app-service.yaml
Cosa fa: K8s legge YAML e crea risorse. Output: "deployment.apps/web-app created".

5. Verifica Tutto Funziona (Sul Master)
bash
sudo k3s kubectl get pods     # Vedi pod "Running"
sudo k3s kubectl get services # Vedi service con NodePort 30080
sudo k3s kubectl get all      # Tutto insieme

Applica il Fix
bash
sudo k3s kubectl delete svc web-app-service  # Rimuovi vecchio Service
sudo k3s kubectl apply -f web-app-service.yaml  # Ricrea corretto

Verifica
bash
sudo k3s kubectl get svc     # Vedi NodePort=30080
curl http://localhost:30080  # Test locale sul master

IMMAGINE NGINX 

----

sudo k3s kubectl top pod    # RAM/CPU uso (installa metrics-server se "not found")
sudo k3s kubectl describe pod -l app=web-app  # Dettagli risorse

scaliamo a 2 repliche 

sudo k3s kubectl scale deployment web-app --replicas=2

sudo k3s kubectl get endpoints web-app-service   #deprecato
sudo k3s kubectl get endpointslices -l kubernetes.io/service-name=web-app-service   #piu moderno

**PER CONTROLLARE LOADBALANCING TRA DUE POD**

sudo k3s kubectl exec pod/web-app-b4f479ccd-48x5s -- cat /var/log/nginx/access.log | tail -5



rifaccio da zero..
-------------------------------------------------------
creo k3s master clonando il templete e cambiando le impostaizoni

qm clone 100 103 --name k3s-master --full && \
qm set 103 --cores 2 --memory 4096 --cpu host && \
qm resize 103 scsi0 +32G

1. Prepara la Chiave SSH su Proxmox
Per passare la chiave via comando, Proxmox deve leggere un file. Crea un file temporaneo con la tua chiave pubblica (quella che hai sul tuo PC in id_rsa.pub):

Bash
# Sulla shell di Proxmox
nano /root/chiave.pub
# INCOLLA DENTRO LA TUA CHIAVE PUBBLICA (inizia con ssh-rsa...)
# Salva con CTRL+X, Y, Invio
2. Lancia i comandi di configurazione (tutti insieme)
Sostituisci l'IP che vuoi dare al Master (es. 192.168.1.50) e il Gateway del tuo router (es. 192.168.1.1).

Bash
# Configura il drive Cloud-Init
qm set 103 --ide2 local-lvm:cloudinit

# Utente e Password
qm set 103 --ciuser ubuntu
qm set 103 --cipassword "passwordsegreta" 

# Chiave SSH (prende il file creato prima)
qm set 103 --sshkeys /root/chiave.pub

# Rete: IP Statico e Gateway (ADATTA GLI IP!)
qm set 103 --ipconfig0 ip=192.168.1.50/24,gw=192.168.1.1

# Abilita l'agente QEMU (per vedere IP e info su Proxmox)
qm set 103 --agent enabled=1

3. Rigenera e Avvia
Per applicare le modifiche all'immagine ISO virtuale di Cloud-Init e avviare:

Bash
qm cloudinit update 103
qm start 103

ok ora sono dentro in ssh alla vm 103 tramite key auth..

# Aggiorna il sistema e installa agente sul nodo master

sudo apt update && sudo apt upgrade -y
sudo apt install qemu-guest-agent -y

# Disabilita il firewall di Ubuntu (per evitare mal di testa con le porte di K3s)
sudo ufw disable

## Installazione di K3s (Senza Traefik)
Questo è il comando chiave. Come abbiamo deciso, disabilitiamo Traefik perché useremo Caddy (sul Raspberry) e NodePort per gestire il traffico. Se lasciassimo Traefik, occuperebbe le porte e farebbe conflitto.

curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -

Di default, k3s richiede sudo per ogni comando kubectl. È noioso. Per usare kubectl senza sudo, lancia questo comando:

Bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml

## abilita qemu agent sulla vm master
sudo systemctl enable qemu-guest-agent
sudo systemctl start qemu-guest-agent
sudo systemctl status qemu-guest-agent    #controlla se sta runnando ed è enabled

IMMAGINE
