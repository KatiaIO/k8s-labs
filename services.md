# Services dans Kubernetes

## Objectifs

- Comprendre les différents types de Services
- Créer et configurer des Services
- Exposer des applications
- Tester la connectivité

## Exercice 1 : ClusterIP Service

1. Déployer une application backend :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: nginx
          ports:
            - containerPort: 80
```

2. Créer un Service ClusterIP :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80
```

3. Tester l'accès :

```bash
# Créer un pod de test
kubectl run test-pod --image=busybox -it --rm -- wget -qO- http://backend-service

# Vérifier les endpoints
kubectl get endpoints backend-service
```

## Exercice 2 : LoadBalancer Service (pour cloud)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lb-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

## Commandes utiles

```bash
# Lister les services
kubectl get services

# Voir les détails d'un service
kubectl describe service <service-name>

# Voir les endpoints
kubectl get endpoints

# Tester la connectivité
kubectl run test-pod --image=busybox -it --rm -- wget -qO- http://service-name

# Voir les logs des pods
kubectl logs -l app=<app-label>
```

## Nettoyage

```bash
kubectl delete service *
kubectl delete deployment *
```
