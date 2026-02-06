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


## creazione app
Dalla dashboard di ArgoCD, clicca su + New App e compila così:

- Application Name: portfolio-app
- Project: default
- Sync Policy: Automatic (spunta anche Prune Resources e Self Heal)

**Source:**

- Repository URL: Scegli il tuo repo ghp_...
- Path: Scrivi apps/portfolio

**Destination:**

- Cluster URL: https://kubernetes.default.svc
- Namespace: default

**Clicca su Create.**
