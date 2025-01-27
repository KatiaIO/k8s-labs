# SecurityContext dans Kubernetes

## Objectifs

- Comprendre et configurer le SecurityContext
- Appliquer les bonnes pratiques de sécurité
- Déboguer les problèmes de permission courants

## Exercice 1 : SecurityContext Basique

### 1.1 Pod avec User et Group Spécifiques

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-user-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
    - name: sec-ctx-demo
      image: busybox
      command: [ "sh", "-c", "sleep 3600" ]
```

Vérification :

```bash
kubectl apply -f security-user-pod.yaml
kubectl exec security-user-pod -- id
```

### 1.2 Non-Root Container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: non-root-pod
spec:
  securityContext:
    runAsNonRoot: true
  containers:
    - name: non-root-container
      image: nginx
      securityContext:
        runAsUser: 1000
```

## Exercice 2 : Capabilities Linux

### 2.1 Drop All Capabilities

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-caps-pod
spec:
  containers:
    - name: no-caps-container
      image: nginx
      securityContext:
        capabilities:
          drop: [ "ALL" ]
```

### 2.2 Ajout de Capabilities Spécifiques

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: specific-caps-pod
spec:
  containers:
    - name: specific-caps-container
      image: nginx
      securityContext:
        capabilities:
          drop: [ "ALL" ]
          add: [ "NET_BIND_SERVICE" ]
```

## Exercice 3 : Système de Fichiers en Lecture Seule

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readonly-pod
spec:
  containers:
    - name: readonly-container
      image: nginx
      securityContext:
        readOnlyRootFilesystem: true
      volumeMounts:
        - name: temp-vol
          mountPath: /tmp
        - name: var-run
          mountPath: /var/run
        - name: var-cache
          mountPath: /var/cache/nginx
  volumes:
    - name: temp-vol
      emptyDir: { }
    - name: var-run
      emptyDir: { }
    - name: var-cache
      emptyDir: { }
```

## Exercice 4 : Cas Pratique - Application Sécurisée

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
      containers:
        - name: app
          image: nginx
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: [ "ALL" ]
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: nginx-cache
              mountPath: /var/cache/nginx
      volumes:
        - name: tmp
          emptyDir: { }
        - name: nginx-cache
          emptyDir: { }
```

## Exercice 5 : Troubleshooting

### 5.1 Permission Denied

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: permission-pod
spec:
  containers:
    - name: permission-container
      image: busybox
      command: [ "sh", "-c", "touch /test.txt" ]
      securityContext:
        readOnlyRootFilesystem: true
```

Solutions à explorer :

1. Ajouter un volume writeable
2. Modifier les permissions
3. Changer le point de montage

### 5.2 Port Binding

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: port-pod
spec:
  containers:
    - name: port-container
      image: nginx
      ports:
        - containerPort: 80
      securityContext:
        capabilities:
          drop: [ "ALL" ]
```

## Points à Noter

1. Niveau Pod vs Conteneur :

- Les paramètres au niveau pod s'appliquent à tous les conteneurs
- Les paramètres au niveau conteneur peuvent surcharger ceux du pod
- Certains paramètres ne sont disponibles qu'au niveau conteneur (capabilities)

2. Bonnes Pratiques :

- Toujours utiliser runAsNonRoot
- Minimiser les capabilities
- Utiliser readOnlyRootFilesystem quand possible
- Définir des UIDs explicites

3. Debuggage :

```bash
# Vérifier le contexte de sécurité
kubectl describe pod [pod-name]

# Vérifier les permissions
kubectl exec [pod-name] -- ls -la /path

# Vérifier l'utilisateur
kubectl exec [pod-name] -- id

# Vérifier les capabilities
kubectl exec [pod-name] -- cat /proc/1/status | grep Cap
```

## Nettoyage

```bash
kubectl delete pod security-user-pod non-root-pod no-caps-pod specific-caps-pod readonly-pod permission-pod port-pod
kubectl delete deployment secure-app
```

## Exercice Bonus

1. Créer un pod qui :

- S'exécute en tant qu'utilisateur non-root
- A un système de fichiers en lecture seule
- Peut écrire dans des chemins spécifiques
- N'a que les capabilities minimales nécessaires

2. Implémenter une politique de sécurité pour :

- Forcer l'utilisation de conteneurs non-root
- Interdire les privilèges élevés
- Limiter les capabilities disponibles