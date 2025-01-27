# Cycle de Vie des Pods et Gestion des Erreurs

## Objectifs

- Comprendre les différents états d'un pod
- Observer le comportement avec une image invalide
- Examiner les logs et les événements
- Comprendre les politiques de redémarrage
- Expérimenter avec les probes

## Exercice 1 : Image Invalide

### 1. Créer un pod avec une image invalide

```yaml
# bad-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  labels:
    app: test
spec:
  containers:
  - name: bad-container
    image: nginx:doesnotexist
    ports:
    - containerPort: 80
```

### 2. Observer et Analyser

```bash
# Déployer le pod
kubectl apply -f bad-pod.yaml

# Observer l'état du pod
kubectl get pod bad-pod -w

# Voir les détails du pod
kubectl describe pod bad-pod

# Examiner les événements
kubectl get events --sort-by=.metadata.creationTimestamp
```

### État Attendu

- Le pod devrait être en état "ImagePullBackOff"
- Les événements devraient montrer des erreurs de pull d'image

## Exercice 2 : CrashLoopBackOff

### 1. Créer un pod qui crash

```yaml
# crash-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: crash-pod
spec:
  containers:
  - name: crash-container
    image: busybox
    command: ["/bin/sh", "-c", "sleep 10 && exit 1"]
```

### 2. Observer le Comportement

```bash
# Déployer le pod
kubectl apply -f crash-pod.yaml

# Observer les redémarrages
kubectl get pod crash-pod -w

# Voir les logs
kubectl logs crash-pod --previous
```

### État Attendu

- Le pod devrait entrer en état "CrashLoopBackOff"
- Le nombre de redémarrages devrait augmenter
- Le délai entre les redémarrages devrait s'allonger (backoff)

## Exercice 3 : Politiques de Redémarrage

### 1. Tester Différentes Politiques

```yaml
# restart-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-pod
spec:
  restartPolicy: OnFailure  # Tester avec: Always, Never, OnFailure
  containers:
  - name: restart-container
    image: busybox
    command: ["/bin/sh", "-c", "sleep 10 && exit 1"]
```

### 2. Observer avec Chaque Politique

```bash
# Tester chaque politique
kubectl apply -f restart-pod.yaml

# Observer le comportement
kubectl get pod restart-pod -w
```

## Exercice 4 : Probes

### 1. Créer un Pod avec des Probes

```yaml
# probe-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-pod
spec:
  containers:
  - name: nginx
    image: nginx
    livenessProbe:
      httpGet:
        path: /nonexistent
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
    readinessProbe:
      httpGet:
        path: /
        port: 80
```

### 2. Observer et Déboguer

```bash
# Déployer le pod
kubectl apply -f probe-pod.yaml

# Observer l'état
kubectl get pod probe-pod -w

# Voir les événements
kubectl describe pod probe-pod
```

## Questions de Compréhension

1. Pourquoi le pod avec l'image invalide entre-t-il en état ImagePullBackOff ?
2. Quelle est la différence entre CrashLoopBackOff et ImagePullBackOff ?
3. Comment le backoff exponentiel fonctionne-t-il ?
4. Quand utiliser chaque politique de redémarrage ?
5. Quelle est la différence entre liveness et readiness probes ?

## Déboggage

Pour chaque scénario d'erreur :

1. Identifier le problème dans les événements
2. Comprendre la cause racine
3. Appliquer la correction appropriée
4. Vérifier la résolution

## Solution des Problèmes

1. Pour l'image invalide :
    - Corriger le nom/tag de l'image
    - Vérifier l'accès au registry

2. Pour le CrashLoopBackOff :
    - Vérifier les logs du conteneur
    - Corriger la commande/configuration

3. Pour les probes :
    - Ajuster les paramètres (délais, seuils)
    - Vérifier les endpoints