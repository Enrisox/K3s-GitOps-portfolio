# Dal controllo imperativo al GitOps dichiarativo

Nei passi precedenti, l’autoscaling e i rollout erano gestiti in modalità imperativa, con comandi diretti al cluster per modificare repliche o aggiornare Pod.

Con GitOps, tutto lo stato desiderato del cluster — Deployment, ConfigMap, HPA, servizi — viene definito in file YAML nel repository Git. Un controller come ArgoCD confronta continuamente lo stato desiderato con quello attuale e applica automaticamente le modifiche necessarie.

Questo approccio garantisce aggiornamenti idempotenti, riproducibili e sicuri, con rollback automatici, self-healing e sincronizzazione continua, eliminando la necessità di interventi manuali diretti sul cluster.

Il GitOps è un paradigma che dice: "Tutto ciò che deve essere installato nel mio cluster deve essere scritto dentro un repository Git (come GitHub)".

**I pilastri del GitOps sono:**

1. **Git come unica fonte di verità**: Non si usa più il terminale (kubectl apply) per cambiare le cose. Se vuoi 5 repliche invece di 2, lo scrivi nel file su GitHub.
2. **Stato Desiderato vs Stato Attuale**: Tu descrivi su Git come vuoi che sia il cluster (Stato Desiderato). Lo strumento GitOps controlla come il cluster è veramente (Stato Attuale).
3. **Sincronizzazione Automatica**: Se i due stati non coincidono, lo strumento corregge il cluster automaticamente.

## Cos'è ArgoCD? 
![Screenshot 2026-02-06 174652](../imgs/Screenshot%202026-02-06%20174652.png)

**Argo CD** è uno **strumento di continuous delivery dichiarativo GitOps per Kubernetes**. Automatizza il deployment e gli aggiornamenti delle applicazioni nei cluster Kubernetes, garantendo coerenza e riducendo gli errori manuali. È uno dei quattro sottoprogetti del Progetto Argo, che a dicembre 2022 ha ricevuto la graduation dalla Cloud Native Computing Foundation (CNCF), il che ne garantisce la sicurezza e l’affidabilità anche su larga scala.

- **Monitora GitHub**: Guarda il mio repository ogni pochi secondi.
- **Monitora il Cluster**: Guarda i miei Pod e Service su K3s.
- **Confronta**: Nota le differenze. Se su GitHub si aggiunge un commento o cambia una porta, lui se ne accorge.
- **Applica**: Se vede una differenza, pulla i file da GitHub e li applica al cluster.

## Installazione ArgoCD
```bash
kubectl create namespace argocd   # Crea il namespace
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml   
```
**Ottimizzazione: Forzare ArgoCD sulla VM Master**

Per evitare che il Raspberry venga rallentato, ho forzato tutti i componenti di ArgoCD a girare esclusivamente sulla VM (nodo k3s-master) tramite il **nodeSelector**.

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
## Crea o modifica il Caddyfile :

```Plaintext
argo.enrisox-devops.it {
    reverse_proxy https://192.168.1.X:30260 {
        transport http {
            tls_insecure_skip_verify
        }
    }
}
```
## Ricarica poi container Caddy :
```
sudo systemctl restart caddy
```

## Estrai e copia la password decriptata 
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

**Dal browser PC ho aperto il sottodominio che ho creato per accedere ad Argo :**

Login:

- User: admin
- Password: Quella copiata

**Verifica dei Pod**
```bash
kubectl get pods -n argocd
```

## Creazione repository privata su Github

![Repository](../imgs/repogithub.png)

**Ho creato due file sulla repository**

```bash
- apps/portfolio/deployment.yaml   #con dentro sia il deployment che il service
- apps/portfolio/configmap.yaml    #con dentro il file html del mio sito web portfolio
```

## Token Github
![Token](../imgs/token.png)
- In settings account Github
- Nella colonna a sinistra, scorri fino in fondo e clicca su Developer settings.
- Lì troverai Personal access tokens. Cliccaci sopra e scegli Tokens (classic).
- spunta casella **repo** che contiene i permessi necessari
- SCADENZA 30 GG o 60

![Token](../imgs/argo1.png)

Settings -> Repositories -> + CONNECT REPO e scegli https:

![Token](../imgs/argo3.png)
<br>
**Ecco cosa scrivere esattamente nei vari box:**

- **Type**: Lascia git.
- **Project**: Lascia default.
- **Repository URL**: Incolla l'indirizzo HTTPS del tuo repo (es: https://github.com/TuoNomeUtente/k3s-gitops-infra.git).
- **Username**: Inserisci il tuo nome utente GitHub.
- **Password**: Incolla qui il Token (PAT) che ha generato prima (quello che inizia con ghp_).
- **TLS client certificate / SSH private key**: Lascia vuoti, non servono per la connessione HTTPS con token.

Se dopo aver premuto connect c'è pallino verde e scritto successfull, è andata a buon fine

![Token](../imgs/token4.png)

## Creazione Applicazione su ArgoCD
![Argo-app](../imgs/argo5.png)

Dalla dashboard di ArgoCD, clicca su + New App e compila così:

- Application Name: portfolio-app
- Project: default
- Sync Policy: Automatic (spunta anche Prune Resources e Self Heal)
- Repository URL:
- Path: apps/portfolio

**Destination:**

- Cluster URL: https://kubernetes.default.svc
- Namespace: default

**Create.**
![Argo-app](../imgs/argo6.png)




