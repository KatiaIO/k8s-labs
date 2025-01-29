# Lab : SecurityContext dans Kubernetes

## Objectifs

- Comprendre et configurer le SecurityContext
- Appliquer les bonnes pratiques de sécurité
- Déboguer les problèmes de permission courants

## Exercice 1 : Pod avec user et group spécifiques

Créez un fichier `security-user-pod.yaml` :

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
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      emptyDir: { }
```

Vérifications à effectuer :

```bash
# Déployer le pod
kubectl apply -f security-user-pod.yaml

# Vérifier l'utilisateur et les groupes (Résultat attendu : uid=1000 gid=3000 groups=2000)
kubectl exec security-user-pod -- id

# Vérifier les permissions des fichiers créés (Résultat attendu : groupe=2000)
kubectl exec security-user-pod -- touch /data/test
kubectl exec security-user-pod -- ls -l /data/test
```

## Exercice 2 : Capabilities Linux

### 2.1 Drop All Capabilities

Créez un fichier `no-caps-pod.yaml` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-caps-pod
spec:
  containers:
    - name: no-caps-container
      image: busybox
      command: [ "sh", "-c", "sleep 3600" ]
      securityContext:
        capabilities:
          drop: [ "ALL" ]
```

Vérifications à effectuer :

```bash
# Déployer le pod
kubectl apply -f no-caps-pod.yaml

# Vérifier les capabilities (Résultat attendu : pas de capabilities)
kubectl exec no-caps-pod -- cat /proc/1/status | grep Cap
kubectl exec no-caps-pod -- ping 8.8.8.8  # Doit échouer
kubectl exec no-caps-pod -- nc -l -p 80   # Doit échouer
```

### 2.2 Ajout de capabilities spécifiques

Créez un fichier `specific-caps-pod.yaml` :

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

Vérifications à effectuer :

```bash
# Déployer le pod
kubectl apply -f specific-caps-pod.yaml

# Vérifier que NET_BIND_SERVICE fonctionne
kubectl exec specific-caps-pod -- nc -l -p 80  # Doit fonctionner
```

## Exercice 3 : Système de fichiers en lecture seule

Créez un fichier `readonly-pod.yaml` :

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

Vérifications à effectuer :

```bash
# Déployer le pod
kubectl apply -f readonly-pod.yaml

# Tester l'écriture dans différents répertoires
kubectl exec readonly-pod -- touch /test.txt  # Doit échouer
kubectl exec readonly-pod -- touch /tmp/test.txt  # Doit réussir
kubectl exec readonly-pod -- touch /var/cache/nginx/test.txt  # Doit réussir
```
