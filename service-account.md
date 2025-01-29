# ServiceAccounts dans Kubernetes

## Objectifs

- Créer et gérer des ServiceAccounts
- Comprendre les Roles et RoleBindings
- Utiliser des ServiceAccounts dans des Pods
- Appliquer les bonnes pratiques de sécurité

## Exercice 1 : ServiceAccount de Base

1. Créer un ServiceAccount simple :

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: default
automountServiceAccountToken: false
```

```bash
# Appliquer et vérifier
kubectl apply -f service-account.yaml
kubectl get serviceaccount
kubectl describe serviceaccount app-service-account
```

## Exercice 2 : Role et RoleBinding

1. Créer un Role avec des permissions limitées :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [ "" ]
    resources: [ "pods" ]
    verbs: [ "get", "watch", "list" ]
```

2. Créer un RoleBinding pour lier le Role au ServiceAccount :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
  - kind: ServiceAccount
    name: app-service-account
    namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

## Exercice 3 : Utilisation dans un Pod

1. Créer un Pod qui utilise le ServiceAccount :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-sa
spec:
  serviceAccountName: app-service-account
  containers:
    - name: main
      image: bitnami/kubectl
      command: [ "sleep", "infinity" ]
```

2. Tester les permissions :

```bash
# Accéder au pod
kubectl exec -it pod-with-sa -- sh

# Dans le pod, tester les permissions
kubectl get pods
kubectl get services  # Devrait échouer
```

## Exercice 4 : Cas Pratique - Application de Monitoring

1. Créer un ServiceAccount pour une application de monitoring :

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: monitoring-role
rules:
  - apiGroups: [ "" ]
    resources: [ "pods", "services" ]
    verbs: [ "get", "list", "watch" ]
  - apiGroups: [ "" ]
    resources: [ "pods/log" ]
    verbs: [ "get" ]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: monitoring-binding
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
roleRef:
  kind: Role
  name: monitoring-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitoring-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: monitoring
  template:
    metadata:
      labels:
        app: monitoring
    spec:
      serviceAccountName: monitoring-sa
      containers:
        - name: monitoring
          image: bitnami/kubectl
          command: [ "sh", "-c", "while true; do kubectl get pods; sleep 10; done" ]
```

## Exercice 5 : Sécurisation des ServiceAccounts

1. Désactiver le montage automatique du token :

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secure-sa
automountServiceAccountToken: false
```

2. Créer un pod qui monte explicitement le token :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  serviceAccountName: secure-sa
  automountServiceAccountToken: true
  containers:
    - name: main
      image: nginx
```

## Vérification et Debug

```bash
# Vérifier les ServiceAccounts
kubectl get serviceaccounts

# Vérifier les Roles
kubectl get roles

# Vérifier les RoleBindings
kubectl get rolebindings

# Voir les détails du ServiceAccount
kubectl describe serviceaccount <nom-sa>

# Vérifier les permissions
kubectl auth can-i --list --as=system:serviceaccount:default:app-service-account
```

## Nettoyage

```bash
kubectl delete serviceaccount app-service-account monitoring-sa secure-sa
kubectl delete role pod-reader monitoring-role
kubectl delete rolebinding read-pods monitoring-binding
kubectl delete pod pod-with-sa secure-pod
kubectl delete deployment monitoring-app
```

## Points Clés à Retenir

1. Les ServiceAccounts sont spécifiques à un namespace
2. Suivre le principe du moindre privilège
3. Utiliser automountServiceAccountToken avec précaution
4. Toujours définir des rôles spécifiques
5. Vérifier régulièrement les permissions accordées

## Bonnes Pratiques

- Créer des ServiceAccounts dédiés pour chaque application
- Limiter les permissions au strict nécessaire
- Documenter l'utilisation des ServiceAccounts
- Auditer régulièrement les permissions
- Désactiver le montage automatique des tokens si non nécessaire