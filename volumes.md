# Exercice Pratique : Application Web avec Multiple Volumes

## Objectif

Créer une application web avec :

- Configuration externe (ConfigMap)
- Stockage persistant pour les uploads
- Volume partagé pour les logs
- Sidecar container pour le traitement des logs

## Étape 1 : Créer la ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;
      root /usr/share/nginx/html;
      location /uploads {
        alias /data/uploads;
      }
    }
```

## Étape 2 : Créer le PVC et le PVC

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: uploads-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 1Gi
```

## Étape 3 : Créer le Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
      volumeMounts:
        - name: config
          mountPath: /etc/nginx/conf.d
        - name: uploads
          mountPath: /data/uploads
        - name: logs
          mountPath: /var/log/nginx

    - name: log-processor
      image: busybox
      command: [ "/bin/sh", "-c", "tail -f /logs/access.log | grep POST >> /logs/uploads.log" ]
      volumeMounts:
        - name: logs
          mountPath: /logs

  volumes:
    - name: config
      configMap:
        name: webapp-config
        items:
          - key: nginx.conf
            path: default.conf
    - name: uploads
      persistentVolumeClaim:
        claimName: uploads-pvc
    - name: logs
      emptyDir: { }
```

## Tâches à Réaliser

### 1. Configuration et Déploiement

- Déployer tous les manifests
- Vérifier que les volumes sont correctement montés
- Valider la configuration nginx

### 2. Test des Uploads

```bash
# Port forward pour accéder à l'application
kubectl port-forward pod/webapp 8080:80

# Créer un fichier test
curl -X POST -F "file=@test.txt" http://localhost:8080/uploads/
```

### 3. Vérification des Logs

```bash
# Vérifier les logs nginx
kubectl exec webapp -c nginx -- cat /var/log/nginx/access.log

# Vérifier les logs traités
kubectl exec webapp -c log-processor -- cat /logs/uploads.log
```

### 4. Tests de Persistance

```bash
# Supprimer et recréer le pod
kubectl delete pod webapp
kubectl apply -f webapp.yaml

# Vérifier que les uploads sont toujours présents
kubectl exec webapp -c nginx -- ls /data/uploads
```

## Bonus

### 1. Ajout de Quotas

- Ajouter des limites de ressources pour les volumes
- Implémenter une rotation des logs

### 2. Monitoring

- Ajouter un conteneur pour collecter des métriques sur l'utilisation des volumes
- Mettre en place des alertes sur l'espace disque

### 3. Sécurité

- Configurer les permissions appropriées sur les volumes
- Mettre en place un chiffrement pour les données sensibles

## Résultats Attendus

- Les fichiers uploadés doivent persister après redémarrage du pod
- Les logs doivent être correctement traités par le sidecar
- La configuration nginx doit être appliquée correctement
- Les volumes doivent être montés avec les bonnes permissions

## Commandes Utiles pour le Debugging

```bash
# Vérifier les volumes montés
kubectl describe pod webapp

# Voir les events du pod
kubectl get events --sort-by=.metadata.creationTimestamp

# Vérifier le statut du PVC
kubectl get pvc uploads-pvc

# Explorer les montages dans les conteneurs
kubectl exec -it webapp -c nginx -- df -h
```

## Points d'Attention

1. Vérifier les permissions des fichiers
2. S'assurer que le storage class est disponible
3. Monitorer l'utilisation de l'espace disque
4. Valider le comportement lors des redémarrages