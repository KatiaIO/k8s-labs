# Les Deployments dans Kubernetes

## Exercice Pratique : Gestion de Version d'Application

### Objectif
Déployer et gérer une application web avec différentes versions.

### 1. Création du Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.14.2
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
```

### 2. Scénarios à Tester
- Mise à l'échelle horizontale
- Mise à jour d'image
- Rollback en cas de problème
- Pause et reprise de rollout
