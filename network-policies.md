# Lab : NetworkPolicies

## Objectif

Comprendre les NetworkPolicies en contrôlant le trafic entre deux pods.

## Exercice 1 : Configuration initiale

1. Créer le fichier `app.yaml` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-a
  labels:
    app: web-a
spec:
  containers:
    - name: nginx
      image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: web-a-svc
spec:
  selector:
    app: web-a
  ports:
    - port: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: web-b
  labels:
    app: web-b
spec:
  containers:
    - name: nginx
      image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: web-b-svc
spec:
  selector:
    app: web-b
  ports:
    - port: 80
```

2. Déployer et tester :

```bash
# Déployer les applications
kubectl apply -f app.yaml

# Test depuis web-a vers web-b
kubectl exec web-a -- curl web-b-svc

# Test depuis web-b vers web-a
kubectl exec web-b -- curl web-a-svc
```

## Exercice 2 : Bloquer le trafic

1. Créer le fichier `deny-all.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: { }
  policyTypes:
    - Ingress
```

2. Appliquer et tester :

```bash
# Appliquer la politique
kubectl apply -f deny-all.yaml

# Test (doit échouer)
kubectl exec web-a -- curl web-b-svc
```

## Exercice 3 : Autoriser un Trafic Spécifique

1. Créer le fichier `allow-a-to-b.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-a-to-b
spec:
  podSelector:
    matchLabels:
      app: web-b
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: web-a
```

2. Appliquer et tester :

```bash
# Appliquer la politique
kubectl apply -f allow-a-to-b.yaml

# Test de web-a vers web-b (doit réussir)
kubectl exec web-a -- curl web-b-svc

# Test de web-b vers web-a (doit échouer)
kubectl exec web-b -- curl web-a-svc
```

## Nettoyage

```bash
# Supprimer toutes les ressources créées
kubectl delete -f app.yaml
kubectl delete networkpolicy deny-all
kubectl delete networkpolicy allow-a-to-b
```

## Points à Retenir

- Par défaut, tout trafic est autorisé
- Une NetworkPolicy permet de contrôler le trafic
- Les sélecteurs de pods utilisent les labels
- Les politiques sont additives
