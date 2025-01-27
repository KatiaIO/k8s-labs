# Multi-Container Pods dans Kubernetes

## Exercice Pratique : Application Web avec Sidecar de Logging

### Objectif
Créer un pod multi-conteneurs contenant :
- Une application web simple (nginx)
- Un sidecar container qui surveille et traite les logs d'accès
- Un volume partagé pour les logs

### Étapes

1. **Créer le manifest du pod**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logging
spec:
  containers:
  - name: nginx
    image: nginx:latest
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: log-collector
    image: busybox
    command: ["/bin/sh", "-c", "tail -f /var/log/nginx/access.log"]
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
  volumes:
  - name: logs-volume
    emptyDir: {}
```

2. **Tâches à réaliser**
   - Déployer le pod
   - Vérifier que les deux conteneurs sont en cours d'exécution
   - Générer du trafic vers nginx
   - Observer les logs dans le sidecar
   - Modifier la configuration pour ajouter des filtres de logs

3. **Commandes utiles**
```bash
# Déployer le pod
kubectl apply -f web-with-logging.yaml

# Vérifier l'état des conteneurs
kubectl get pod web-with-logging
kubectl describe pod web-with-logging

# Port-forward pour accéder à nginx
kubectl port-forward web-with-logging 8080:80

# Générer du trafic (dans un autre terminal)
curl http://localhost:8080

# Observer les logs du sidecar
kubectl logs web-with-logging -c log-collector -f
```

4. **Bonus**
   - Ajouter un init container qui configure nginx avant le démarrage
   - Modifier le sidecar pour compter les codes de statut HTTP
   - Implémenter une probe de readiness pour nginx

### Critères de Réussite
- Les deux conteneurs doivent être en état "Running"
- Le sidecar doit afficher les logs d'accès nginx en temps réel
- Les logs doivent être persistent entre les redémarrages des conteneurs
- L'application doit être accessible via port-forward
