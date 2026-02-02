# Sito web Portfolio - App stateless su K3S

Per sperimentare con il load balancing tra i nodi ho creato un sito web stateless, "**Portfolio**", che gira su Nginx in cluster Kubernetes. 

## Creazione sito Sulla VM Master node

```bash
mkdir -p ~/portfolio
cd ~/portfolio
nano index.html
```

## Caricamento del sito in Kubernetes sulla VM Master
Per rimuovere il sudo permanentemente e non doverlo usare ogni volta:

```bash​
sudo chmod 644 /etc/rancher/k3s/k3s.yaml       # a ravvio proxmox si resettano i permessi
sudo chown userX:userX /etc/rancher/k3s/k3s.yaml        #definitivo
```
Creazione **ConfigMap**:

- Una ConfigMap è un database interno a Kubernetes che memorizza dati in formato chiave-valore.
- Invece di creare una nuova immagine Docker ogni volta che cambi una riga di HTML, carichi il file in una ConfigMap.
- Il file index.html diventa disponibile per tutti i nodi del cluster (Master e Raspberry) senza che debba esistere fisicamente su quei nodi. È Kubernetes che lo "inietta" nel container al momento dell'avvio.

```bash
kubectl create configmap portfolio-html --from-file=index.html
```

**Creazione del Deployment (portfolio.yaml**):
In Kubernetes un Deployment è una risorsa che descrive, in modo dichiarativo, come deve girare una certa applicazione nel cluster: quante repliche avere, che immagine usare e come aggiornare le versioni.

**Cosa fa un Deployment**

- Mantiene sempre in esecuzione il numero desiderato di Pod (es. 3 repliche di un’app web).
- Gestisce gli aggiornamenti “rolling” cambiando gradualmente i Pod alla nuova versione dell’immagine, senza downtime (se configurato correttamente).
- Permette rollback facili: se una nuova versione rompe qualcosa, puoi tornare a quella precedente con un comando.
-Consente di scalare in su/giù modificando solo replicas nel manifest o con kubectl scale.

```bash
nano portfolio.yaml
```

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
      nodePort: *****

```

**Infine applicare tutto con questo comando:**

```bash
kubectl apply -f portfolio.yaml`
```

## Analisi Sicurezza e Risorse

### 1. Sicurezza del Container (Pod Security)

Ho implementato diverse regole per impedire a un attaccante di prendere il controllo del sistema:

* **Non-Root (Rootless):** Il container gira con UID 101 (nginx). Senza permessi di amministratore.
* **Drop Capabilities:** Rimossi TUTTI i privilegi Linux (capabilities: drop: ["ALL"]).
* **No Privilege Escalation:** Bloccata la possibilità di guadagnare privilegi (allowPrivilegeEscalation: false).
* **Seccomp Profile:** Attivato il filtro `RuntimeDefault` per bloccare chiamate al Kernel pericolose.

### 2. Sicurezza del Filesystem

* **Read-Only Root Filesystem:** L'intero sistema operativo del container è in sola lettura. Un attaccante non può scaricare malware o creare backdoor.
* **Eccezione:** Montati volumi temporanei (`emptyDir`) su `/tmp` e `/var/cache`.

### 3. Protezione dalle risorse (Anti-DoS)

**Resource Limits:**

* CPU: Max 0.5 core (500m).
* RAM: Max 256 MiB.

**Cosa vuol dire "100m"?** In Kubernetes, 1000m = 1 Core intero. Impostando 100m, il container usa al massimo il 10% di un core. Se qualcuno bombarda il sito (DoS), il pod rallenta ma non blocca le macchine ospitanti.

### 4. Sicurezza di Rete & Client (Caddy Headers)

* **X-Frame-Options "DENY":** Protegge dal Clickjacking.
* **X-Content-Type-Options "nosniff":** Impedisce l'esecuzione di file camuffati.
* **X-XSS-Protection:** Filtri anti-scripting.
* **HSTS:** Forza l'uso di HTTPS.
* **Server Token Removal:** Nasconde il Reverse Proxy in uso.


## Configurare Reverse Proxy 

Sul nodo worker, diciamo a Caddy di fare da Reverse Proxy:

**Modifica del Caddyfile:**

```text
portfolio.enrisox-devops.it {
    reverse_proxy 192.168.1.XX:port
}

```
**Riavvio del container Caddy:**

```text
docker restart caddy
```

## DNS e Test

Dal mio provider DNS ho creato un nuovo record A che punta al mio IP pubblico(inoltre tramite container DDNS, Cloudflare aggiorna il mio IP qualora cambiasse)

https://portfolio.enrisox-devops.it

IMMAGINE SITO

**Comandi di verifica:**

```bash
kubectl get pods
kubectl get pods -o wide
```
IMMAGINE PODS

### Procedura di Aggiornamento index.html

Ogni volta che modificherò l'HTML sulla VM Master:

1. nano index.html
2. Aggiornare la ConfigMap:

```bash
kubectl delete configmap portfolio-html
kubectl create configmap portfolio-html --from-file=index.html
kubectl rollout restart deployment portfolio

```

**Comando per applicare modifiche al file .yaml:**
```bash
kubectl apply -f portfolio.yaml
```


