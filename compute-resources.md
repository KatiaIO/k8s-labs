# Gestion des Resources (Requests et Limits)

## Objectifs

- Comprendre la différence entre requests et limits
- Configurer des ressources pour les pods
- Observer l'impact des limits
- Travailler avec LimitRange et ResourceQuota

## Exercice 1 : Pod Basique avec Resources

1. Créer un pod avec requests et limits :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

2. Vérifier l'état du pod :

```bash
kubectl apply -f resource-pod.yaml
kubectl get pod resource-demo
kubectl describe pod resource-demo
```

## Exercice 2 : Démonstration des Limits

1. Créer un pod qui dépasse ses limites :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
  - name: memory-demo
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
        cpu: "250m"
      limits:
        memory: "100Mi"
        cpu: "500m"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

2. Observer le comportement :

```bash
kubectl apply -f memory-stress.yaml
kubectl get pod memory-demo
kubectl describe pod memory-demo
```

## Exercice 3 : LimitRange

1. Créer un LimitRange pour le namespace :

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: "256Mi"
      cpu: "500m"
    defaultRequest:
      memory: "128Mi"
      cpu: "250m"
    type: Container
```

2. Créer un pod sans spécifier de ressources :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-resources
spec:
  containers:
  - name: nginx
    image: nginx
```

3. Vérifier les ressources attribuées automatiquement :

```bash
kubectl describe pod default-resources
```

## Exercice 4 : ResourceQuota

1. Créer un ResourceQuota :

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 1Gi
    limits.cpu: "4"
    limits.memory: 2Gi
    pods: "4"
```

2. Tenter de déployer plusieurs pods :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
          limits:
            memory: "512Mi"
            cpu: "1"
```

3. Observer le comportement :

```bash
kubectl describe quota compute-quota
kubectl get pods
kubectl describe deployment nginx-deployment
```

## Exercice 5 : Les Classes QoS

1. Pod Guaranteed (requests = limits) :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "256Mi"
        cpu: "500m"
      limits:
        memory: "256Mi"
        cpu: "500m"
```

2. Pod Burstable (requests < limits) :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
        cpu: "500m"
```

3. Pod BestEffort (pas de requests ni limits) :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
spec:
  containers:
  - name: nginx
    image: nginx
```

4. Vérifier les QoS classes :

```bash
kubectl get pod -o custom-columns=NAME:.metadata.name,QoS:.status.qosClass
```

## Exercice 6 : Monitoring des Resources

1. Vérifier l'utilisation des ressources :

```bash
# Par pod
kubectl top pod

# Par nœud
kubectl top node
```

## Nettoyage

```bash
kubectl delete pod resource-demo memory-demo default-resources qos-guaranteed qos-burstable qos-besteffort
kubectl delete limitrange mem-limit-range
kubectl delete resourcequota compute-quota
kubectl delete deployment nginx-deployment
```

## Points Clés à Retenir

1. Requests

- Ressources garanties
- Impact sur le scheduling
- Utilisées pour le placement des pods

2. Limits

- Limite maximale d'utilisation
- Dépassement CPU : throttling
- Dépassement mémoire : OOMKilled

3. QoS Classes

- Guaranteed : meilleure garantie de performance
- Burstable : compromis équilibré
- BestEffort : aucune garantie

4. Bonnes Pratiques

- Toujours définir des requests et limits
- Monitorer l'utilisation réelle
- Utiliser LimitRange pour les valeurs par défaut
- Utiliser ResourceQuota pour le contrôle au niveau namespace

## Troubleshooting

Problèmes courants :

1. Pod en état Pending

- Vérifier les ressources disponibles sur les nœuds
- Vérifier les ResourceQuotas

2. Pod OOMKilled

- Augmenter la limite de mémoire
- Vérifier les fuites mémoire
- Analyser l'utilisation mémoire

3. Pod throttled

- Vérifier l'utilisation CPU
- Ajuster les limits CPU si nécessaire
