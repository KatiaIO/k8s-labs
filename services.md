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

## Exercice 2 : NodePort Service

1. Déployer une application web :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
        ports:
        - containerPort: 80
```

2. Créer un Service NodePort :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080  # Port entre 30000-32767
```

3. Accéder au service :
```bash
# Obtenir l'IP du node
kubectl get nodes -o wide

# Accéder via navigateur ou curl
curl http://NODE_IP:30080
```

## Exercice 3 : Multi-Port Service

1. Déployer une application avec plusieurs ports :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-port-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: multi-port
  template:
    metadata:
      labels:
        app: multi-port
    spec:
      containers:
      - name: container
        image: nginx
        ports:
        - containerPort: 80
          name: http
        - containerPort: 443
          name: https
```

2. Créer un Service multi-port :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
spec:
  selector:
    app: multi-port
  ports:
  - name: http
    port: 80
    targetPort: http
  - name: https
    port: 443
    targetPort: https
```

## Exercice 4 : Service avec Session Affinity

1. Créer un Service avec session affinity :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  selector:
    app: web
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
  ports:
  - port: 80
    targetPort: 80
```

## Exercice 5 : LoadBalancer Service (pour cloud)

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

## Exercice 6 : Service avec Endpoints Externes

1. Créer un Service sans sélecteur :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
```

2. Définir les Endpoints manuellement :
```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service
subsets:
- addresses:
  - ip: 192.168.1.100
  - ip: 192.168.1.101
  ports:
  - port: 80
```

## Exercice 7 : Application Multi-Tier

1. Déployer une application complète :
```yaml
# Frontend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx
        env:
        - name: BACKEND_URL
          value: "http://backend-service"
---
# Frontend Service
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    nodePort: 30080
---
# Backend Deployment
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
---
# Backend Service
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
```

## Commandes Utiles

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

## Troubleshooting

1. Service non accessible :
- Vérifier les labels et sélecteurs
- Vérifier les endpoints
- Vérifier les ports
- Vérifier l'état des pods

2. NodePort non accessible :
- Vérifier les règles de firewall
- Vérifier l'IP du node
- Vérifier que le port est ouvert

3. LoadBalancer en attente :
- Vérifier la configuration cloud
- Vérifier les logs du cloud controller

## Nettoyage
```bash
kubectl delete service backend-service web-service multi-port-service sticky-service lb-service external-service frontend-service
kubectl delete deployment backend web-app multi-port-app frontend
kubectl delete endpoints external-service
```