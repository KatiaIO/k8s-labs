# Secrets dans Kubernetes

## Objectifs

- Créer différents types de Secrets
- Utiliser les Secrets dans les pods
- Comprendre les bonnes pratiques de sécurité
- Gérer les Secrets de manière sécurisée

## Exercice 1 : Création et utilisation simple

### 1.1 Création de secrets depuis des fichiers

```bash
# Créer les fichiers de credentials
echo -n "admin" > username.txt
echo -n "password123" > password.txt

# Créer le secret
kubectl create secret generic db-credentials \
  --from-file=username=username.txt \
  --from-file=password=password.txt

# Vérifier le secret
kubectl get secret db-credentials -o yaml
kubectl get secret db-credentials -o jsonpath='{.data.username}' | base64 --decode
```

### 1.2 Création via YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials-yaml
type: Opaque
stringData:
  username: admin
  password: t0p-Secret
```

### 1.3 Secret TLS

```bash
# Génération d'un certificat auto-signé pour test
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=myapp.com"

# Création du secret TLS
kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key
  
kubectl describe secrets tls-secret
```

## Exercice 2 : Application complète avec Secrets

### 2.1 Création des Secrets pour l'application

```yaml
# db-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secrets
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: rootpass
  MYSQL_DATABASE: myapp
  MYSQL_USER: appuser
  MYSQL_PASSWORD: apppass
```

### 2.2 Déploiement de la base de données

```yaml
# mysql-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:5.7
          envFrom:
            - secretRef:
                name: mysql-secrets

```

```bash
# Se connecter à MySQL en utilisant les credentials définis dans le secret
kubectl exec -it deployment/mysql -- mysql -u appuser -papppass myapp -e "SELECT 1;"
```

### 2.3 Déploiement de l'application web

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx
          volumeMounts:
            - name: secrets
              mountPath: /etc/secrets
              readOnly: true
      volumes:
        - name: secrets
          secret:
            secretName: mysql-secrets
```

## Exercice 3 : Debug et troubleshooting

### 3.1 Vérification des montages

```bash
# Vérifier les volumes montés
kubectl exec [pod-name] -- ls -l /etc/secrets

# Vérifier les variables d'environnement
kubectl exec [pod-name] -- env | grep MYSQL
```

## Exercice 4 : Bonnes pratiques

### 4.1 Configuration RBAC

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
  - apiGroups: [ "" ]
    resources: [ "secrets" ]
    verbs: [ "get" ]
    resourceNames: [ "mysql-secrets" ]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-secret-reader
subjects:
  - kind: ServiceAccount
    name: web-app
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## Questions pour la réflexion

1. Pourquoi utiliser un Secret plutôt qu'un ConfigMap pour les credentials ?
2. Quels sont les risques de sécurité liés aux Secrets dans Kubernetes ?
3. Comment améliorer la sécurité des Secrets dans un environnement de production ?
4. Dans quels cas utiliser les variables d'environnement vs les volumes pour les Secrets ?

## Points à noter

1. Sécurité :
    - Les secrets sont encodés en base64 (non chiffrés) par défaut
    - Limiter l'accès avec RBAC
    - Ne jamais commiter les secrets dans le code source
    - Envisager des solutions externes (Vault, AWS Secrets Manager) pour la production

2. Bonnes pratiques :
    - Utiliser stringData au lieu de data pour plus de lisibilité
    - Toujours monter les secrets en readOnly
    - Préférer les volumes aux variables d'environnement
    - Implémenter la rotation des secrets

3. Limitations :
    - Taille maximale de 1MB
    - Les secrets sont stockés en clair dans etcd
    - Les mises à jour ne sont pas propagées automatiquement aux variables d'environnement

## Nettoyage

```bash
# Supprimer les secrets
kubectl delete secret mysql-secrets db-credentials tls-secret

# Supprimer les déploiements
kubectl delete deployment mysql web-app

# Supprimer les configurations RBAC
kubectl delete role secret-reader
kubectl delete rolebinding app-secret-reader

# Supprimer les fichiers temporaires
rm username.txt password.txt tls.crt tls.key
```