# Exercice Pratique : NetworkPolicies

## Objectif
Mettre en place une architecture trois tiers sécurisée avec des NetworkPolicies.

## Étape 1 : Déploiement de l'Application

### 1. Création des Déploiements
```yaml
# frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
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

---
# backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: api
        image: python:alpine

---
# database.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
spec:
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:alpine
```

## Étape 2 : Configuration des NetworkPolicies

### 1. Politique "Deny All"
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### 2. Politique Frontend
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 8080
```

### 3. Politique Backend
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

### 4. Politique Database
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
```

## Étape 3 : Tests de Connectivité

### 1. Test Direct
```bash
# Test Frontend → Backend
kubectl exec -it $(kubectl get pod -l app=frontend -o name | head -n1) -- \
  wget -qO- http://backend:8080

# Test Backend → Database
kubectl exec -it $(kubectl get pod -l app=backend -o name | head -n1) -- \
  nc -zv database 5432

# Test Frontend → Database (devrait échouer)
kubectl exec -it $(kubectl get pod -l app=frontend -o name | head -n1) -- \
  nc -zv database 5432
```

### 2. Test avec un Pod Temporaire
```bash
# Créer un pod de test
kubectl run test-pod --rm -i -t --image=alpine -- sh

# Tester les connexions
wget -qO- http://frontend
wget -qO- http://backend:8080
nc -zv database 5432
```

## Étape 4 : Scénarios de Débogage

### 1. Vérification des Politiques
```bash
# Lister toutes les politiques
kubectl get networkpolicies

# Examiner les détails
kubectl describe networkpolicy frontend-policy
kubectl describe networkpolicy backend-policy
kubectl describe networkpolicy database-policy
```

### 2. Analyse des Logs
```bash
# Vérifier les logs des pods
kubectl logs -l app=backend
kubectl logs -l app=database

# Observer les événements
kubectl get events --sort-by=.metadata.creationTimestamp
```

## Bonus : Ajout de Règles Avancées

### 1. Autoriser les Mises à Jour
```yaml
# Ajouter à la politique backend
egress:
- to:
  - ipBlock:
      cidr: 0.0.0.0/0
  ports:
  - protocol: TCP
    port: 443
```

### 2. Monitoring
```yaml
# Politique pour Prometheus
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: monitoring-policy
spec:
  podSelector:
    matchLabels:
      app: prometheus
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 9090
```

## Critères de Réussite

1. **Isolation**
   - Frontend peut uniquement accéder au Backend
   - Backend peut uniquement accéder à la Database
   - Aucun autre accès n'est possible

2. **Fonctionnalité**
   - L'application fonctionne normalement
   - Les services peuvent communiquer
   - Les restrictions sont appliquées

3. **Sécurité**
   - Politique "deny all" en place
   - Accès minimal configuré
   - Pas de failles de sécurité