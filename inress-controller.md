# Exercice Pratique : NGINX Ingress Controller

## Objectif

Déployer et configurer NGINX Ingress Controller pour exposer plusieurs services avec différentes configurations.

## Étape 1 : Installation de NGINX Ingress Controller

```bash
# Via Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --create-namespace \
  --namespace ingress-nginx
```

## Étape 2 : Déployer les Applications de Test

### 1. Application Frontend

```yaml
# frontend-app.yaml
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
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
```

### 2. Application Backend

```yaml
# backend-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
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
          image: httpd:alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80
```

## Étape 3 : Configuration des Ingress Rules

### 1. Configuration de Base

```yaml
# basic-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: basic-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-svc
                port:
                  number: 80
```

### 2. Configuration SSL/TLS

```bash
# Générer un certificat auto-signé
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=app.example.com"

# Créer un secret TLS
kubectl create secret tls app-tls \
  --key tls.key \
  --cert tls.crt
```

```yaml
# ssl-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ssl-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
```

## Étape 4 : Tests de Validation

```bash
# Obtenir l'IP du Ingress Controller
export INGRESS_IP=$(kubectl get svc -n ingress-nginx \
  ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Ajouter une entrée dans /etc/hosts
echo "$INGRESS_IP app.example.com" | sudo tee -a /etc/hosts

# Tester l'accès
curl -H "Host: app.example.com" http://$INGRESS_IP
curl -H "Host: app.example.com" http://$INGRESS_IP/api
```

## Étape 5 : Debugging

### 1. Vérifier les logs

```bash
# Logs du controller
kubectl logs -n ingress-nginx \
  -l app.kubernetes.io/name=ingress-nginx --tail=100

# Événements
kubectl get events -n ingress-nginx --sort-by='.lastTimestamp'
```

### 2. Vérifier la configuration

```bash
# Configuration active de NGINX
kubectl exec -it -n ingress-nginx \
  $(kubectl -n ingress-nginx get pods -l app.kubernetes.io/name=ingress-nginx \
  -o jsonpath='{.items[0].metadata.name}') \
  -- cat /etc/nginx/nginx.conf
```

## Critères de Réussite

1. **Configuration de Base**
    - Frontend accessible sur /
    - Backend accessible sur /api
    - Routage correct des requêtes

2. **Sécurité**
    - TLS fonctionnel
    - Rate limiting efficace
    - CORS correctement configuré

3. **Performance**
    - Load balancing fonctionnel
    - Session affinity respectée
    - Pas d'erreurs dans les logs

4. **Maintenance**
    - Logs accessibles et utiles
    - Configuration facilement modifiable
    - Métriques disponibles