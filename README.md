# K3s GitOps Cluster

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


