# Portfolio hostato su cluster Kubernetes K3S 

Questo progetto descrive la creazione di un'infrastruttura K3s ibrida partendo da Proxmox (Nodo Master) fino a Raspberry Pi (nodo Worker). 
Ibrida sia a livello di architettura: nodo Master su VM in Proxmox ( architettura x86_64/AMD64 ) e nodo worker su Raspberry Pi ( arch ARM64) 
Che a livello di piattaforme: Virtuale e fisico.

**K3s è una versione "miniaturizzata" e super leggera di Kubernetes (K8s).**

È stato creato da Rancher Labs per essere installato facilmente su dispositivi con poche risorse, come un Raspberry Pi, o per ambienti di sviluppo e test rapidi.

Punti chiave:

1. **Automazione con Cloud-Init**: Setup rapido della VM Master tramite template e iniezione di chiavi SSH/Rete.
2. **Ottimizzazione K3s**: Installazione leggera senza Traefik per delegare la gestione del traffico a Caddy esterno.
3. **Configurazione Raspberry**: Abilitazione dei cgroups per permettere al nodo worker di unirsi al cluster.
4. **Deployment Hardened**: Pubblicazione di un sito statico tramite ConfigMap e Nginx, con un forte focus sulla sicurezza (filesystem in sola lettura, utente non-root, limiti di CPU/RAM) per proteggere l'hardware sottostante da attacchi o sovraccarichi.
5. **Esposizione Sicura**: Utilizzo di Caddy come Reverse Proxy con header di sicurezza avanzati e gestione DNS per l'accesso pubblico via HTTPS.


## Setup Infrastruttura: Proxmox e K3s
Per creare la VM master, ho utilizzato una VM template su Proxmox, precedentemente creata in un altro mio progetto ( https://github.com/Enrisox/DevOps-CI-CD-Lab-Jenkins-Terraform-Ansible) per fare esperienza con Terraform. A questa VM ho cambiato le risorse allocate per venire incontro a quelle richieste dal nodo master.

IMMAGINE VM TEMPLATE PROXMOX 

### Creazione VM Master

```bash
qm clone 100 103 --name k3s-master --full && \      
qm set 103 --cores 2 --memory 4096 --cpu host && \
qm resize 103 scsi0 +32G

```
**qm clone 100 103**crea una copia esatta della VM con ID 100 (che è il template pre-configurato) e assegna alla nuova VM l'ID 103.
**--name k3s-master**: Rinomina la nuova VM in "k3s-master" per riconoscerla facilmente.
**--full**: Questa è la parte importante. Crea un Full Clone. Significa che copia fisicamente tutto il disco. La nuova VM sarà totalmente indipendente dal template originale

Invece di accendere la VM e configurare IP, utenti e password a mano nel terminal della console, ho usato **Cloud-Init** per "iniettare" queste impostazioni dall'esterno mentre la macchina si avvia.

### 1. Preparare la Chiave SSH su Proxmox

Per passare la chiave via comando, Proxmox deve leggere un file. Crea un file temporaneo con la tua chiave pubblica. Sulla shell di Proxmox:

`nano /root/chiave.pub`

**INCOLLA DENTRO LA TUA CHIAVE PUBBLICA**

### Configurazione Cloud-Init
Il template non può avere IP statico (altrimenti tutti i cloni avrebbero lo stesso IP = conflitto).
Lancia i comandi di configurazione (tutti insieme). Sostituisci l'IP che vuoi dare al Master (es. 192.168.1.50) e il Gateway del tuo router (es. 192.168.1.1).

**Configura il drive Cloud-Init**
`qm set 103 --ide2 local-lvm:cloudinit`

**Utente e Password**
`qm set 103 --ciuser ubuntu`
`qm set 103 --cipassword "passwordsegreta"`

**Chiave SSH (prende il file creato prima)**
`qm set 103 --sshkeys /root/chiave.pub`

**Rete: IP Statico e Gateway (ADATTA GLI IP!)**
`qm set 103 --ipconfig0 ip=192.168.1.50/24,gw=192.168.1.1`

**Abilita l'agente QEMU (per vedere IP e info su Proxmox)**
`qm set 103 --agent enabled=1`

**Rigenera e Avvia**
Per applicare le modifiche all'immagine ISO virtuale di Cloud-Init e avviare:

```bash
qm cloudinit update 103
qm start 103

```

---

## Configurazione Interna VM Master

Ok ora sono dentro in ssh alla vm 103 tramite key auth..

### Aggiorna il sistema e installa agente sul nodo master

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install qemu-guest-agent -y

```

### Disabilita il firewall di Ubuntu

(Per evitare mal di testa con le porte di K3s)
`sudo ufw disable`

### Installazione di K3s (Senza Traefik)

Questo è il comando chiave. Come abbiamo deciso, disabilitiamo Traefik perché useremo Caddy (sul Raspberry) e NodePort per gestire il traffico. Se lasciassimo Traefik, occuperebbe le porte e farebbe conflitto.
`curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -`

Di default, k3s richiede sudo per ogni comando kubectl. È noioso. Per usare kubectl senza sudo, lancia questo comando:
`sudo chmod 644 /etc/rancher/k3s/k3s.yaml`

### Abilita qemu agent sulla vm master

```bash
sudo systemctl enable qemu-guest-agent
sudo systemctl start qemu-guest-agent
sudo systemctl status qemu-guest-agent    #controlla se sta runnando ed è enabled

```

---

## Setup Nodi (Raspberry Pi)

### Copia token da dare a raspberry

`sudo cat /var/lib/rancher/k3s/server/node-token`

### Configurazione Cgroups

I Raspberry Pi spesso hanno i "cgroups" disabilitati di default. Senza questi, K3s non parte.
`cat /boot/firmware/cmdline.txt`

Cosa cercare: In fondo alla riga, devi vedere scritto: `cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory`

Se NON ci sono:

1. Modifica il file: `sudo nano /boot/firmware/cmdline.txt`
2. `sudo reboot`

### Unisci il raspberry al cluster

Dopo il riavvio:
`curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.50:6443 K3S_TOKEN="K10***********token::server:token****************" sh -`

**Verifica dal master con:**
`sudo kubectl get nodes`

---

## Deployment Applicazione: Portfolio Statico

Creiamo un Portfolio statico (solo HTML e CSS) che gira su Nginx dentro Kubernetes. Lo terremo separato dal resto usando un nuovo sottodominio (es. portfolio.enrisox-devops.it) e una porta diversa (es. 30081).

### FASE 1: Crea il tuo sito (Sulla VM Master)

```bash
mkdir -p ~/portfolio
cd ~/portfolio
nano index.html

```

### FASE 2: Carica il sito in Kubernetes (Sulla VM Master)

Usiamo una **ConfigMap**: "Prendi questo file index.html e tienilo in memoria".
`kubectl create configmap portfolio-html --from-file=index.html`

Ora creiamo il Deployment (**portfolio.yaml**):
`nano portfolio.yaml`

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

Applica tutto:
`kubectl apply -f portfolio.yaml`

---

## Analisi Sicurezza e Risorse

### 1. Sicurezza del Container (Pod Security)

Queste regole impediscono a un attaccante di prendere il controllo del sistema:

* **Non-Root (Rootless):** Il container gira con UID 101 (nginx). Senza permessi di amministratore.
* **Drop Capabilities:** Rimosse TUTTI i privilegi Linux (`capabilities: drop: ["ALL"]`).
* **No Privilege Escalation:** Bloccata la possibilità di guadagnare privilegi (`allowPrivilegeEscalation: false`).
* **Seccomp Profile:** Attivato il filtro `RuntimeDefault` per bloccare chiamate al Kernel pericolose.

### 2. Sicurezza del Filesystem

* **Read-Only Root Filesystem:** L'intero sistema operativo del container è in sola lettura. Un attaccante non può scaricare malware o creare backdoor.
* **Eccezione:** Montati volumi temporanei (`emptyDir`) su `/tmp` e `/var/cache`.

### 3. Protezione dalle risorse (Anti-DoS)

**Resource Limits:**

* CPU: Max 0.5 core (500m).
* RAM: Max 256 MiB.

> **Cosa vuol dire "100m"?** In Kubernetes, 1000m = 1 Core intero. Impostando 100m, il container usa al massimo il 10% di un core. Se qualcuno bombarda il sito (DoS), il pod rallenta ma non blocca il Raspberry Pi.

### 4. Sicurezza di Rete & Client (Caddy Headers)

* **X-Frame-Options "DENY":** Protegge dal Clickjacking.
* **X-Content-Type-Options "nosniff":** Impedisce l'esecuzione di file camuffati.
* **X-XSS-Protection:** Filtri anti-scripting.
* **HSTS:** Forza l'uso di HTTPS.
* **Server Token Removal:** Nascondiamo che stiamo usando Caddy.

---

## FASE 3: Configura Caddy (Sul Raspberry)

Sul nodo worker, diciamo a Caddy di fare da Reverse Proxy:

1. Modifica il Caddyfile: `nano Caddyfile`
2. Aggiungi il blocco:

```text
portfolio.enrisox-devops.it {
    reverse_proxy 192.168.1.XX:30081
}

```

3. Riavvia Caddy: `docker restart caddy`

---

## FASE 4: DNS e Test

1. Vai sul tuo provider DNS.
2. Crea un nuovo record **A** (sottodominio `portfolio`) che punta al tuo IP pubblico di casa.
3. Aspetta qualche minuto e visita `https://portfolio.enrisox-devops.it`.

**Comandi di verifica:**

```bash
kubectl get pods
kubectl get pods -o wide

```

### Procedura di Aggiornamento index.html

Ogni volta che modifichi l'HTML sulla VM Master:

1. Modifica il file: `nano index.html`
2. Aggiorna la ConfigMap:

```bash
kubectl delete configmap portfolio-html
kubectl create configmap portfolio-html --from-file=index.html
kubectl rollout restart deployment portfolio

```

**Comando per applicare modifiche al file .yaml:**
`kubectl apply -f portfolio.yaml`

---
