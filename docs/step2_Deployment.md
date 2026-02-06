# Sito Web Portfolio: Applicazione Stateless su K3s
L'obiettivo di questa fase è implementare un'applicazione stateless per testare il bilanciamento del carico tra i nodi x86 (VM) e ARM (Raspberry Pi) e sperimentare le logiche di Self-Healing e Scalabilità di Kubernetes.

## Creazione sito Sulla VM del Master node
Il contenuto statico viene creato inizialmente sul nodo master:
```bash
mkdir -p ~/portfolio
cd ~/portfolio
nano index.html
```

## Astrazione del Contenuto: Kubernetes ConfigMap
Invece di seguire il workflow tradizionale (build di un'immagine Docker personalizzata per ogni minima modifica), ho optato per l'uso di una ConfigMap.
Una **ConfigMap** è un database interno a Kubernetes che memorizza dati in formato chiave-valore.
Invece di creare una nuova immagine Docker ogni volta che cambio una riga di HTML, caricherò il file in una ConfigMap.
Il file index.html diventa disponibile per tutti i nodi del cluster (Master e Raspberry) senza che debba esistere fisicamente su quei nodi. È Kubernetes che lo "inietta" nel container al momento dell'avvio.
Vantaggi di questo approccio:

- **Disaccoppiamento**: Il codice (HTML) è separato dall'infrastruttura (Nginx).
- **Agilità**: Posso aggiornare il sito rapidamente senza dover gestire un Container Registry o rifare il deploy dell'immagine.
- **Distribuzione Automatica**: Kubernetes "inietta" il file index.html all'interno dei Pod come se fosse un volume fisico, indipendentemente dall'architettura del nodo (Master o Worker).


```bash
kubectl create configmap portfolio-html --from-file=index.html
```

**Creazione del Deployment (portfolio.yaml**):
In Kubernetes un Deployment è una risorsa che descrive, in modo dichiarativo, come deve girare una certa applicazione nel cluster: quante repliche avere, che immagine usare e come aggiornare le versioni.

**Cosa fa un Deployment**

- Mantiene sempre in esecuzione il numero desiderato di Pod (es. 3 repliche di un’app web).
- Gestisce gli aggiornamenti “rolling” cambiando gradualmente i Pod alla nuova versione dell’immagine, senza downtime (se configurato correttamente).
- Permette rollback facili: se una nuova versione rompe qualcosa, puoi tornare a quella precedente con un comando.
- Consente di scalare in su/giù modificando solo replicas nel manifest o con kubectl scale.

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
kubectl apply -f portfolio.yaml
```

## Analisi Sicurezza e Risorse

### 1. Pod Security

Ho implementato diverse regole per impedire a un attaccante di prendere il controllo del sistema:
* **nginx-unprivileged**: L'immagine Nginx ufficiale "standard" è progettata per avviarsi come root perché deve aprire la porta 80. Usando la versione unprivileged, il processo Nginx è configurato internamente per girare sulla porta 8080. Questo permette al container di funzionare correttamente anche quando Kubernetes impone runAsNonRoot: true, eliminando alla radice il rischio che un attaccante possa scalare i privilegi.
* **alpine** : Attack surfase ridotta per mancanza di pacchetti o shell avanzate e minor numero di CVE rilevate dagli scanne di sicurezza.
* **Non-Root (Rootless):** Il container gira con UID 101 (nginx). Senza permessi di amministratore.
* **Drop Capabilities:** Rimossi TUTTI i privilegi Linux (capabilities: drop: ["ALL"]).
* **No Privilege Escalation:** Bloccata la possibilità di guadagnare privilegi (allowPrivilegeEscalation: false).
* **Seccomp Profile:** Attivato il filtro `RuntimeDefault` per bloccare chiamate al Kernel pericolose.

### 2. Sicurezza del Filesystem

* **Read-Only Root Filesystem:** L'intero sistema operativo del container è in sola lettura. Un attaccante non può scaricare malware o creare backdoor.
Nginx, per funzionare, abbia bisogno di scrivere dei file temporanei (PID e cache). senza i volumi emptyDir su /tmp e /var/cache/nginx , con un filesystem in sola lettura, Nginx crasherebbe all'avvio perché l'immagine unprivileged tenta di scrivere i file di gestione del processo in quelle directory.

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

## Configurazione di Caddy come Reverse Proxy 

Sul nodo worker, ho modificato il Caddyfile per dirgli di fare Reverse proxy verso il sottodominio.

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

![Anteprima Portfolio](../imgs/portfolio1.png)

Dal mio provider DNS ho creato un nuovo record A che punta al mio IP pubblico(inoltre tramite DDNS Dockerizzato, sono sicuro che i miei sottodomini puntino sempre al mio IP corrente.

                                        https://portfolio.enrisox-devops.it

**Comandi di verifica:**
![Portfolio Nodes Running](imgs/portfolio_nodes_running.png)

```bash
kubectl get pods
kubectl get pods -o wide
```

# Due tipi di modifiche
- modifica del solo contenuto HTML
- modifica della configurazione Kubernetes (Deployment)

## Procedura Aggiornamento index.html

1. modifica index.html con **nano index.html**
2. Aggiornamento di ConfigMap:
```bash
kubectl create configmap portfolio-html \
  --from-file=index.html \
  --dry-run=client -o yaml | kubectl apply -f -       #Zero downtime. La ConfigMap non sparisce mai.

kubectl rollout restart deployment portfolio         #ordina al Deployment di creare nuovi Pod (con il nuovo HTML) e spegnere quelli vecchi gradualmente (Rolling Update). Questo assicura che gli utenti vedano subito la modifica senza interrompere il servizio.
```
## Comando all-in-one
```bash
kubectl create configmap portfolio-html --from-file=index.html --dry-run=client -o yaml | kubectl apply -f - && kubectl rollout restart deployment portfolio
```


## Comando per applicare modifiche al file deployment, portfolio.yml:

**Metodo solo se cambia la struttura Kubernetes, ad esempio:**

- numero di repliche
- immagine Docker
- porte
- risorse CPU / RAM
- volumi o mount
- variabili d’ambiente

```bash
nano portfolio.yaml
kubectl apply -f portfolio.yaml
```
## Verifica dello Stato
Dopo aver eseguito una delle due procedure, verifica che i nuovi pod siano attivi e funzionanti:

```bash
kubectl rollout status deployment portfolio
```

