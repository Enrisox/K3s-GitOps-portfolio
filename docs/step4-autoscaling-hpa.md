# HPA (Horizontal Pod Autoscaler)
Il suo compito sarà aumentare o diminuire il numero di repliche (Pod) della mia applicazione in base a quanto carico riceverà.

- **Monitoraggio**: Ogni 15 secondi (di default), l'HPA interroga il Metrics Server (un componente già incluso nel tuo K3s) per sapere quanta CPU o RAM stanno usando i tuoi Pod.
- **Calcolo**: Confronta il consumo attuale con l'obiettivo (target) che gli hai dato. (Se hai impostato un target del 50% di CPU e i tuoi Pod sono all'80%, l'HPA capisce che l'app è sotto stress.)
- **Azione**: L'HPA ordina al Deployment di aumentare il numero di repliche. Se invece il carico è basso, spegne i Pod in eccesso per risparmiare risorse.

```bash
kubectl autoscale deployment portfolio --cpu-percent=50 --min=2 --max=5

kubectl get hpa -w     #per controllare stato HPA
```
Autoscaling con Horizontal Pod Autoscaler (HPA)

In questo step vediamo come Kubernetes può scalare automaticamente il numero di repliche di un Deployment in base al **carico reale dei Pod**, garantendo che l’applicazione resti performante senza interventi manuali.

---

## Obiettivo

- Aumentare il numero di Pod quando l’applicazione riceve più traffico o usa più risorse.
- Ridurre i Pod quando il carico è basso per risparmiare risorse.
- Garantire continuità e resilienza senza downtime aggiuntivo.

## Come funziona HPA

L’HPA si basa sul **Metrics Server**, che monitora le risorse utilizzate dai Pod (CPU, RAM o metriche personalizzate).

### Ciclo di controllo HPA

1. **Monitoraggio**  
   - Ogni 15 secondi (default) HPA interroga il Metrics Server per misurare CPU e memoria dei Pod.  

2. **Calcolo**  
   - Confronta l’uso attuale con l’obiettivo dichiarato (target).  
   - Esempio: target CPU = 50%, i Pod consumano 80% → HPA rileva carico alto.  

3. **Azione**  
   - Se il carico supera il target, HPA aumenta le repliche.  
   - Se il carico scende, riduce le repliche.  
   - Tutto avviene senza downtime grazie alla strategia RollingUpdate definita nello Step 3.



## Esempi pratici

Creiamo un HPA per il Deployment `portfolio`:

```bash
kubectl autoscale deployment portfolio \
  --cpu-percent=50 \
  --min=2 \
  --max=5

Cambio il file HTML dentro a configmap.yaml e modifco deployment.yaml sepre dalla repo , in modo che parta un refresh automatico su ArgoCD e il mio sito web si aggiorni..


