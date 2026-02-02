# Zero-Downtime Deployment con Rolling Update e Health Probes
Questo documento descrive come configurare una strategia di deployment **zero-downtime** in Kubernetes utilizzando **Rolling Update, Readiness/Liveness Probes e procedure di rollout controllate**.
In un ambiente production, gli aggiornamenti delle applicazioni non devono mai interrompere il servizio agli utenti finali.[web:43] Kubernetes offre meccanismi nativi per garantire che durante un rollout:

- Almeno un Pod resti sempre attivo e pronto a servire traffico
- I nuovi Pod vengano testati prima di ricevere richieste
- I vecchi Pod vengano terminati solo dopo che i nuovi sono operativi

Ho aggiunto queste righe al mio file portfolio.yml:

- **maxUnavailable: 0**	Impedisce la terminazione di Pod finché i nuovi non sono Ready --> Zero secondi di downtime garantito 
- **maxSurge: 1**  Permette un Pod extra temporaneo durante il rollout --> Capacità sempre >= 100% 
- **Readiness/Liveness Probe**	Testa quando un Pod può ricevere traffico per evitare error 502 durante rollout e rileva Pod bloccati e li riavvia automaticamente  --> Self-healing automatico
- **Stakater Reloader**	Automazione completa del rollout quando cambia la ConfigMap
​- **kubectl rollout history**	Facilita rollback rapidi in caso di problemi
​- **Metrics Server + kubectl top**	Monitoring real-time delle risorse per validare i limits

```bash
nano portfolio.yaml​
```

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portfolio
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0  # Mai spegnere un Pod finché il nuovo non è Ready
      maxSurge: 1        # Permetti 1 Pod extra durante il rollout (max 3 temporanei)
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
      nodePort: *****    #metti porta
```
## Applica le modifiche
```bash
kubectl apply -f portfolio.yaml
```
**Kubernetes eseguirà un aggiornamento progressivo (Rolling Update) per attivare le nuove Probe e le Strategy senza mai interrompere il servizio.**

IMMAGINE ROLLOUT

## Collaudo delle Strategie e delle Probe
Verifichiamo che la strategia Zero-Downtime e le sentinelle di controllo (Readiness e Liveness) siano attive.

```bash
kubectl describe deployment portfolio | grep -A 3 "RollingUpdate"
```

StrategyType:           RollingUpdate
RollingUpdateStrategy:  0 max unavailable, 1 max surge

```bash
​kubectl describe pod -l app=portfolio | grep -E "Readiness|Liveness"
```

**Verifica dei Pod**

```bash
kubectl get pods -l app=portfolio
```
# Gestione degli Aggiornamenti (Rollout)
In Kubernetes, se modifichi una ConfigMap, i Pod già in esecuzione non se ne accorgono perché hanno già caricato il file in memoria. Per rendere effettive le modifiche senza spegnere il sito, utilizziamo il comando di Rollout.

Il **comando Rollout Restart** comando ordina a Kubernetes di ricreare tutti i Pod del deployment seguendo la strategia definita:

```bash
kubectl rollout restart deployment portfolio && kubectl rollout status deployment portfolio         #per vedere lo stato in diretta
```
IMMAGINE restart

## Processo in background

- **Creazione (Surge)**: Grazie a maxSurge: 1, Kubernetes crea un terzo Pod aggiornato.
- **Test di Readiness**: Il nuovo Pod viene messo "in prova". Solo dopo che la readinessProbe dà esito positivo (il sito risponde), il Pod è considerato pronto.
- **Sostituzione**: Kubernetes inizia a terminare uno dei vecchi Pod.
- **Continuità**: La procedura si ripete finché tutti i vecchi Pod non sono sostituiti.
- **Risultato finale**: Durante tutto il processo, il traffico gestito da Caddy viene indirizzato solo ai Pod sani. L'utente non percepirà alcuna interruzione (Zero-Downtime).


# HPA (Horizontal Pod Autoscaler)
Il suo compito sarà aumentare o diminuire il numero di repliche (Pod) della mia applicazione in base a quanto carico riceverà.

- **Monitoraggio**: Ogni 15 secondi (di default), l'HPA interroga il Metrics Server (un componente già incluso nel tuo K3s) per sapere quanta CPU o RAM stanno usando i tuoi Pod.
- **Calcolo**: Confronta il consumo attuale con l'obiettivo (target) che gli hai dato. (Se hai impostato un target del 50% di CPU e i tuoi Pod sono all'80%, l'HPA capisce che l'app è sotto stress.)
- **Azione**: L'HPA ordina al Deployment di aumentare il numero di repliche. Se invece il carico è basso, spegne i Pod in eccesso per risparmiare risorse.

```bash
kubectl autoscale deployment portfolio --cpu-percent=50 --min=2 --max=5

kubectl get hpa -w     #per controllare stato HPA
```

# Cos'è GitOps?

Il GitOps è un paradigma che dice: "Tutto ciò che deve essere installato nel mio cluster deve essere scritto dentro un repository Git (come GitHub)".

**I pilastri del GitOps sono:**

1. **Git come unica fonte di verità**: Non si usa più il terminale (kubectl apply) per cambiare le cose. Se vuoi 5 repliche invece di 2, lo scrivi nel file su GitHub.
2. **Stato Desiderato vs Stato Attuale**: Tu descrivi su Git come vuoi che sia il cluster (Stato Desiderato). Lo strumento GitOps controlla come il cluster è veramente (Stato Attuale).
3. **Sincronizzazione Automatica**: Se i due stati non coincidono, lo strumento corregge il cluster automaticamente.

## Cos'è ArgoCD? 
**ArgoCD è un controller di Kubernetes che implementa il GitOps. È un software che installiamo dentro la VM Master e che rimane costantemente "in ascolto".**


- **Monitora GitHub**: Guarda il mio repository ogni pochi secondi.
- **Monitora il Cluster**: Guarda i miei Pod e Service su K3s.
- **Confronta**: Nota le differenze. Se su GitHub si aggiunge un commento o cambia una porta, lui se ne accorge.
- **Applica**: Se vede una differenza, pulla i file da GitHub e li applica al cluster.

```bash
kubectl create namespace argocd   # Crea il namespace
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml        # Installa ArgoCD
```
**Ottimizzazione: Forzare ArgoCD sulla VM Master**

Per evitare che il Raspberry venga rallentato, ho forzato tutti i componenti di ArgoCD a girare esclusivamente sulla VM (nodo k3s-master) tramite il nodeSelector.
```bash
kubectl patch deployment argocd-server -n argocd -p '{"spec": {"template": {"spec": {"nodeSelector": {"kubernetes.io/hostname": "k3s-master"}}}}}'
kubectl patch deployment argocd-repo-server -n argocd -p '{"spec": {"template": {"spec": {"nodeSelector": {"kubernetes.io/hostname": "k3s-master"}}}}}'
kubectl patch deployment argocd-redis -n argocd -p '{"spec": {"template": {"spec": {"nodeSelector": {"kubernetes.io/hostname": "k3s-master"}}}}}'
kubectl patch deployment argocd-dex-server -n argocd -p '{"spec": {"template": {"spec": {"nodeSelector": {"kubernetes.io/hostname": "k3s-master"}}}}}'
kubectl patch deployment argocd-notifications-controller -n argocd -p '{"spec": {"template": {"spec": {"nodeSelector": {"kubernetes.io/hostname": "k3s-master"}}}}}'
kubectl patch deployment argocd-applicationset-controller -n argocd -p '{"spec": {"template": {"spec": {"nodeSelector": {"kubernetes.io/hostname": "k3s-master"}}}}}'
kubectl patch statefulset argocd-application-controller -n argocd -p '{"spec": {"template": {"spec": {"nodeSelector": {"kubernetes.io/hostname": "k3s-master"}}}}}'
```
**Esponiamo il servizio come NodePort:**

```Bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```
**Troviamo la porta assegnata :**

```Bash
kubectl get svc -n argocd argocd-server
```
**Configura il Caddyfile sul Raspberry:**

```Plaintext
argo.enrisox-devops.it {
    reverse_proxy https://192.168.1.50:30263 {
        transport http {
            tls_insecure_skip_verify
        }
    }
}
```
**Restartiamo Caddy :**
```
sudo systemctl restart caddy
```

**Estrai la password decriptata e copiala**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
**Attiva l'accesso Web**

```bash
```
**Dal mio browser PC ho aperto il sottodominio creato poco fa sul mio gestore DNS:**

https://argo.enrisox-devops.it

Login:

- User: admin
- Password: Quella copiata poco fa.

**Verifica dei Pod**
```bash
kubectl get pods -n argocd
```

## Creazione repository private su mio Github
ConfigMap = archivio esterno di file/testo per i tuoi container.


IMMAGINE GITHUBREPO

**Ho creato nella repo due file**

```bash
- apps/portfolio/deployment.yaml   #con dentro sia il deployment che il service
- apps/portfolio/configmap.yaml    #con dentro il file html del mio sito web portfolio
```

- settings del profilo github, non della repo
- Nella colonna a sinistra, scorri fino in fondo e clicca su Developer settings.
- Lì troverai Personal access tokens. Cliccaci sopra e scegli Tokens (classic).
- spunta casella repo che contiene i permessi necessari

  IMMAGINE TOKEN2
  
- SCADENZA 30 GG o 60

  immagine token 3

  Vai su Settings -> Repositories -> + CONNECT REPO e scegli https:


**Ecco cosa scrivere esattamente nei vari box:**

- **Type**: Lascia git.
- **Project**: Lascia default.
- **Repository URL**: Incolla l'indirizzo HTTPS del tuo repo (es: https://github.com/TuoNomeUtente/k3s-gitops-infra.git).
- **Username**: Inserisci il tuo nome utente GitHub.
- **Password**: Incolla qui il Token (PAT) che ha generato prima (quello che inizia con ghp_).
- **TLS client certificate / SSH private key**: Lascia vuoti, non servono per la connessione HTTPS con token.


Se dopo aver premuto connect c'è pallino verde e scritto successfull, è andato tutto bene.

IMMAGINE TOKEN4





