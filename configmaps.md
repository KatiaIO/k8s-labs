# ConfigMaps dans Kubernetes

## Objectifs
- Créer des ConfigMaps de différentes manières
- Utiliser les ConfigMaps dans les pods
- Gérer les mises à jour de configuration
- Comprendre les meilleures pratiques

## Exercice 1 : Création de ConfigMaps

### 1.1 Création depuis des littéraux
```bash
kubectl create configmap literal-config \
  --from-literal=DB_HOST=mysql \
  --from-literal=DB_PORT=3306
```

### 1.2 Création depuis un fichier
1. Créer un fichier `app.properties` :
```properties
app.name=MyApp
app.env=production
log.level=INFO
```

2. Créer le ConfigMap :
```bash
kubectl create configmap properties-config --from-file=app.properties
```

### 1.3 Création depuis un répertoire
1. Créer plusieurs fichiers de configuration :
```bash
mkdir config
echo "server.port=8080" > config/server.properties
echo "cache.ttl=3600" > config/cache.properties
```

2. Créer le ConfigMap :
```bash
kubectl create configmap dir-config --from-file=config/
```

### 1.4 Création depuis un YAML
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: yaml-config
data:
  database.conf: |
    host=postgresql
    port=5432
    name=myapp
  api.conf: |
    endpoint=/api/v1
    timeout=30
```

## Exercice 2 : Utilisation des ConfigMaps

### 2.1 Variables d'environnement
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: literal-config
          key: DB_HOST
    - name: DB_PORT
      valueFrom:
        configMapKeyRef:
          name: literal-config
          key: DB_PORT
```

### 2.2 Toutes les variables d'un ConfigMap
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envfrom-pod
spec:
  containers:
  - name: app
    image: nginx
    envFrom:
    - configMapRef:
        name: literal-config
```

### 2.3 Montage en volume
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: yaml-config
```

### 2.4 Montage d'un fichier spécifique
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: single-file-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app/database.conf
      subPath: database.conf
  volumes:
  - name: config-volume
    configMap:
      name: yaml-config
      items:
      - key: database.conf
        path: database.conf
```

## Exercice 3 : Cas Pratique - Application Web

1. Créer une configuration nginx :
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;
      
      location / {
        root /usr/share/nginx/html;
        index index.html;
      }
      
      location /api {
        proxy_pass http://backend-service:8080;
      }
    }
  
  index.html: |
    <!DOCTYPE html>
    <html>
    <body>
      <h1>Welcome to my configured nginx!</h1>
    </body>
    </html>
```

2. Déployer nginx avec cette configuration :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-configured
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/conf.d/default.conf
      subPath: nginx.conf
    - name: nginx-config
      mountPath: /usr/share/nginx/html/index.html
      subPath: index.html
  volumes:
  - name: nginx-config
    configMap:
      name: nginx-config
```

## Exercice 4 : Mise à jour des ConfigMaps

1. Mettre à jour un ConfigMap :
```bash
kubectl edit configmap nginx-config
```

2. Observer le comportement :
```bash
kubectl exec nginx-configured -- cat /etc/nginx/conf.d/default.conf
```

3. Forcer le rechargement (en pratique) :
```bash
kubectl delete pod nginx-configured
kubectl apply -f nginx-pod.yaml
```

## Points à Noter

1. Les mises à jour de ConfigMaps :
- Les volumes sont mis à jour automatiquement (avec délai)
- Les variables d'environnement ne sont PAS mises à jour
- Un redémarrage du pod est nécessaire pour les variables d'environnement

2. Bonnes pratiques :
- Versionnez vos ConfigMaps
- Utilisez des noms descriptifs
- Préférez les volumes pour les fichiers de configuration
- Utilisez les variables d'environnement pour les valeurs simples

3. Limites :
- Taille maximale de 1MB
- Ne pas utiliser pour les données sensibles (utiliser Secrets)
- Les mises à jour ne sont pas atomiques

## Nettoyage
```bash
kubectl delete configmap literal-config properties-config dir-config yaml-config nginx-config
kubectl delete pod env-pod envfrom-pod volume-pod single-file-pod nginx-configured
```

## Vérification et Debug

```bash
# Voir les ConfigMaps
kubectl get configmaps

# Voir le contenu d'un ConfigMap
kubectl describe configmap nginx-config

# Voir les variables d'environnement dans un pod
kubectl exec env-pod -- env

# Vérifier les fichiers montés
kubectl exec volume-pod -- ls -l /etc/config
```

## Exercice Bonus

Créer un déploiement qui :
1. Utilise plusieurs ConfigMaps
2. Monte certains fichiers en subPath
3. Combine variables d'environnement et volumes
4. Implémente une stratégie de rechargement de configuration