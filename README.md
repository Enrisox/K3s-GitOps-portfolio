# K3s GitOps Cluster

![Demo portfolio](imgs/portfolio-enrico.gif)

Repository per la gestione GitOps di un cluster **K3s** multi-nodo. L'infrastruttura è ottimizzata per l'edge computing, eliminando l'ingress controller interno (Traefik) in favore di un'istanza esterna di Caddy.

## Architettura

* **Control Plane (Master)**: VM Ubuntu server / Proxmox.
* **Worker Node**: Ubuntu server on Raspberry Pi.
* **Edge Proxy**: **Caddy** (esterno al cluster) gestisce i certificati SSL e il routing via HTTPS.
* **GitOps**: **ArgoCD** per il deployment automatico dal repository verso il cluster.

## Stack Tecnologico

* **Orchestratore**: K3s.
* **Reverse Proxy**: Caddy.
* **Continuous Delivery**: ArgoCD.
* **Networking**: Esposizione servizi tramite **NodePort**.


## Flusso del Traffico

1. L'utente richiede https://portfolio.enrisox-devops.it
2. **Caddy** intercetta la richiesta, gestisce il TLS e agisce da Reverse Proxy.
3. Il traffico viene inoltrato ai nodi del cluster K3s sulla specifica porta **NodePort**.
4. Il servizio interno instrada il pacchetto ai Pod del Portfolio.


## Roadmap del Progetto 

Ho suddiviso la configurazione in moduli logici per facilitare la replicabilità e la manutenzione:

1.  [**Step 1: Installazione K3s**](docs/step1_K3S.md)  
    Configurazione del Master, join del Worker (Raspberry Pi) e ottimizzazione delle risorse.
2.  [**Step 2: Deployment & Networking**](docs/step2_Deployment.md)  
    Configurazione del portfolio, gestione del Service NodePort e setup di Caddy.
3.  [**Step 3: Zero Downtime**](docs/step3-zero-downtime.md)  
    Strategie di aggiornamento (Rolling Update) per garantire la continuità del servizio.
4.  [**Step 4: Autoscaling (HPA)**](docs/step4-autoscaling-hpa.md)  
    Configurazione del Horizontal Pod Autoscaler basato sull'utilizzo della CPU.
5.  [**Step 5: GitOps con ArgoCD**](docs/step5-gitops-argocd.md)  
    Installazione di ArgoCD e collegamento a questo repository per il Continuous Deployment.

## ⚠️ Home Lab Disclaimer

Trattandosi di un ambiente di test basato su risorse hardware limitate e condivise (Home Lab), è importante notare quanto segue:

* **Gestione Risorse**: Il sito Portfolio potrebbe subire rallentamenti nel caso in cui le risorse di calcolo vengano temporaneamente dirottate su altri progetti o container presenti sulla stessa infrastruttura.
* **Alta Affidabilità (HA)**: Il cluster è configurato con architettura Master/Worker. Sebbene i Pod sul nodo Worker continuino a seguire l'ultimo stato noto, l'eventuale spegnimento o manutenzione del nodo Master (VM su Proxmox) potrebbe causare errori temporanei nel caricamento delle pagine. Tali anomalie non sono riconducibili a bug di configurazione o errori nel codice, ma esclusivamente alla disponibilità hardware del laboratorio personale.


Nei prossimi mesi integrerò e aumenterò complessità progetto/cluster : STAY TUNED! <br>

**Se ti è piaciuto, lasciami una ⭐!**

**Enrico Soci - DevSecOps Junior**

