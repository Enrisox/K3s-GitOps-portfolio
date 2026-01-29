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
# creo k3s master clonando il templete e cambiando le impostaizoni

```bash
qm clone 100 103 --name k3s-master --full && \
qm set 103 --cores 2 --memory 4096 --cpu host && \
qm resize 103 scsi0 +32G
```

**1. Prepara la Chiave SSH su Proxmox**
Per passare la chiave via comando, Proxmox deve leggere un file. Crea un file temporaneo con la tua chiave pubblica 


# Sulla shell di Proxmox
```bash
nano /root/chiave.pub
```

# INCOLLA DENTRO LA TUA CHIAVE PUBBLICA (inizia con ssh-rsa...)
# Salva con CTRL+X, Y, Invio
2. Lancia i comandi di configurazione (tutti insieme)
Sostituisci l'IP che vuoi dare al Master (es. 192.168.1.50) e il Gateway del tuo router (es. 192.168.1.1).


# Configura il drive Cloud-Init

```bash
qm set 103 --ide2 local-lvm:cloudinit
```

# Utente e Password
```bash

qm set 103 --ciuser ubuntu
qm set 103 --cipassword "passwordsegreta" 
```
# Chiave SSH (prende il file creato prima)
```bash
qm set 103 --sshkeys /root/chiave.pub
```

# Rete: IP Statico e Gateway (ADATTA GLI IP!)

```bash
qm set 103 --ipconfig0 ip=192.168.1.50/24,gw=192.168.1.1
```

# Abilita l'agente QEMU (per vedere IP e info su Proxmox)
```bash
qm set 103 --agent enabled=1
```

3. Rigenera e Avvia
Per applicare le modifiche all'immagine ISO virtuale di Cloud-Init e avviare:

```bash
qm cloudinit update 103
qm start 103
```

ok ora sono dentro in ssh alla vm 103 tramite key auth..

# Aggiorna il sistema e installa agente sul nodo master

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install qemu-guest-agent -y
```

# Disabilita il firewall di Ubuntu (per evitare mal di testa con le porte di K3s)

```bash
sudo ufw disable
```

## Installazione di K3s (Senza Traefik)
Questo è il comando chiave. Come abbiamo deciso, disabilitiamo Traefik perché useremo Caddy (sul Raspberry) e NodePort per gestire il traffico. Se lasciassimo Traefik, occuperebbe le porte e farebbe conflitto.
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -
```

Di default, k3s richiede sudo per ogni comando kubectl. È noioso. Per usare kubectl senza sudo, lancia questo comando:

```bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

## abilita qemu agent sulla vm master
```bash
sudo systemctl enable qemu-guest-agent
sudo systemctl start qemu-guest-agent
sudo systemctl status qemu-guest-agent    #controlla se sta runnando ed è enabled
```

IMMAGINE

# copia token da dare a raspberry

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

# I Raspberry Pi spesso hanno i "cgroups" disabilitati di default. Senza questi, K3s non parte.

```bash
cat /boot/firmware/cmdline.txt
```

Cosa cercare: In fondo alla riga, devi vedere scritto: cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory

Se NON ci sono:

Modifica il file: sudo nano /boot/firmware/cmdline.txt

```bash
sudo nano /boot/firmware/cmdline.txt
sudo reboot
```

# dopo ravvio unisci il raspberry al cluster

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.50:6443 K3S_TOKEN="K10***********token::server:token****************" sh -
```

IMMAGINE NODI 

se il comando ha successo, verifica dal master con 

```bash
sudo kubectl get nodes
```



Creiamo un **Portfolio statico** (solo HTML e CSS) che gira su Nginx dentro Kubernetes.
Lo terremo separato dal resto usando un nuovo sottodominio (es. `portfolio.enrisox-devops.it`) e una porta diversa (es. **30081**).

Ecco il piano d'attacco DevOps:

1. **Creiamo il sito** (HTML) sulla VM.
2. **Impacchettiamo il sito** dentro Kubernetes (usando un oggetto chiamato `ConfigMap`, così non devi impazzire a creare immagini Docker per ora).
3. **Lanciamo Nginx** che legge quel sito.
4. **Configuriamo Caddy** sul Raspberry per puntare lì.

---

### FASE 1: Crea il tuo sito (Sulla VM Master)

Colleghiamoci alla VM Master
1. Crea una cartella per il progetto:

```bash
mkdir -p ~/portfolio
cd ~/portfolio

```


2. Crea il file `index.html`:
```bash
nano index.html

```

### FASE 2: Carica il sito in Kubernetes (Sulla VM Master)

Invece di costruire un'immagine Docker complessa, usiamo una **ConfigMap**. In pratica, diciamo a Kubernetes: *"Prendi questo file index.html e tienilo in memoria"*.

1. Crea la ConfigMap dal file appena creato:
```bash
kubectl create configmap portfolio-html --from-file=index.html

```


2. Ora creiamo il Deployment che usa Nginx e gli "monta" dentro il tuo file HTML. Crea il file `portfolio.yaml`:
```bash
nano portfolio.yaml

```


3. Incolla questo (nota la porta **30081**):
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
      nodePort: 30081

```

4. Applica tutto:
```bash
kubectl apply -f portfolio.yaml
```
## Sicurezza del Container (Pod Security)
**Queste regole impediscono a un attaccante di prendere il controllo del sistema anche se riuscisse a entrare nel container.**

- **Non-Root (Rootless)**: Il container gira con l'utente UID 101 (nginx) invece che root (0). Se un attaccante esegue codice remoto, non ha permessi di amministratore.

- **Drop Capabilities**: Abbiamo rimosso TUTTI i privilegi Linux (capabilities: drop: ["ALL"]). Il processo non può cambiare permessi ai file, manipolare la rete o gestire processi di sistema.

- **No Privilege Escalation**: Abbiamo bloccato la possibilità per il processo di guadagnare più privilegi di quelli che ha (allowPrivilegeEscalation: false). Blocca attacchi basati su binari SUID (es. sudo).

- **Seccomp Profile**: Abbiamo attivato il filtro RuntimeDefault che blocca le chiamate al Kernel (syscall) non necessarie o pericolose, riducendo la superficie di attacco verso l'host.

**2. Sicurezza del Filesystem**

- **Read-Only Root Filesystem**: L'intero sistema operativo del container è in sola lettura.

- **Effetto**: Un attaccante non può scaricare malware, non può modificare configurazioni, non può creare backdoor persistenti.

- **Eccezione gestita**: Abbiamo montato volumi temporanei (emptyDir) solo su /tmp e /var/cache perché Nginx ne ha bisogno per vivere, ma quei dati spariscono al riavvio.

**3. Protezione dalle risorse (Anti-DoS)**
Resource Limits: Ho imposto limiti rigidi:

- CPU: Max 0.5 core (500m).
- RAM: Max 256 MiB.

**CPU: Cosa vuol dire "100m"?**
**In Kubernetes, 1 CPU equivale a 1 Core fisico (o virtuale) del processore. Per essere precisi, Kubernetes divide ogni Core in 1000 millicores (da qui la "m").**

1000m = 1 Core intero (100% di potenza).
500m = Mezzo Core (50% di potenza).
100m = Un decimo di Core (10% di potenza).

Cosa ho fatto io? Impostando limit: 100m, ho detto a Kubernetes: "Questo container Nginx può usare al massimo il 10% della potenza di un singolo core del Raspberry Pi."

Se il sito riceve troppe visite e Nginx prova a usare più potenza (es. il 50%), Kubernetes lo "strozza" (Throttling) e lo costringe a restare dentro il suo 10%. Il sito andrà lento, ma non bloccherà il resto del Raspberry Pi.
Effetto: Se qualcuno bombarda il sito di richieste (DoS), il pod muore o rallenta, ma non blocca il Raspberry Pi o la VM Master. Il resto del cluster sopravvive.

4. **Sicurezza di Rete & Client (Caddy Headers)**
Queste misure proteggono chi visita il tuo sito (i client).

- X-Frame-Options "DENY": Nessuno può incorporare il tuo sito in un <iframe> (protegge dal Clickjacking).

- X-Content-Type-Options "nosniff": Impedisce al browser di eseguire file camuffati (es. un .txt che in realtà è uno script maligno).

- X-XSS-Protection: Attiva i filtri anti-scripting dei browser vecchi.

- Strict-Transport-Security (HSTS): Forza il browser a usare sempre e solo HTTPS, prevenendo attacchi Man-in-the-Middle.

- Server Token Removal (-Server): Nascondiamo al mondo che stiamo usando "Caddy". Meno informazioni diamo agli attaccanti, meglio è (Security by Obscurity, come strato aggiuntivo).

### FASE 3: Configura Caddy (Sul Raspberry)

Sul nodo worker,diciamo a Caddy: *"Se qualcuno cerca `portfolio.enrisox-devops.it`, mandalo alla porta 30081"*.

**1. Modifica il Caddyfile:**
```bash
nano Caddyfile
```
**2. Aggiungi questo blocco alla fine (sostituisci sempre con l'IP LAN del Raspberry):**

```bash
portfolio.enrisox-devops.it {
    reverse_proxy 192.168.1.XX:30081
}

```

**3. Riavvia Caddy.**

```bash
docker restart caddy
---

### FASE 4: DNS e Test

1. Vai sul tuo provider DNS.
2. Crea un nuovo record **A** (sottodominio `portfolio`) che punta al tuo IP pubblico di casa.
3. Aspetta qualche minuto e visita `https://portfolio.enrisox-devops.it`.

Dovresti vedere la tua pagina nera e verde stile "Hacker" con l'elenco dei tuoi progetti! 😎

```bash
kubectl get pods
kubectl get pods -o wide
```

# La Procedura di Aggiornamento file index.html
Ogni volta che modifichi l'HTML, devi lanciare questi comandi sulla VM Master:

Modifica il file: nano index.html (incolla il nuovo codice e salva).

Aggiorna la "fotocopia" (ConfigMap):

```bash

kubectl delete configmap portfolio-html
kubectl create configmap portfolio-html --from-file=index.html
kubectl rollout restart deployment portfolio    #Riavvia i Pod per fargli leggere la novità
```

comando per applicare modifiche fatte al file .yaml
```bash
kubectl apply -f portfolio.yaml
```

