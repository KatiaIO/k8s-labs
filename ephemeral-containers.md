# Les Conteneurs Éphémères dans Kubernetes

## Exercice Pratique

### Objectif

Utiliser les conteneurs éphémères pour déboguer une application Java distroless.

### Étapes

1. **Déployer l'application**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: java-app
spec:
  containers:
    - name: app
      image: nginx
```

2. **Tâches de Débogage**

- Inspecter les processus Java
- Vérifier les connexions réseau
- Examiner les fichiers logs
- Tester la connectivité réseau

3. **Commandes Utiles**

```bash
# Ajouter un debugger
kubectl debug -it java-app --image=busybox --target=app

# Vérifier les processus
ps aux

# Examiner le réseau
netstat -tuln

# Inspecter les fichiers
ls -la /proc/1/root
```
