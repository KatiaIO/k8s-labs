# Secrets dans Kubernetes

## Objectifs
- Créer différents types de Secrets
- Utiliser les Secrets dans les pods
- Comprendre les bonnes pratiques de sécurité
- Gérer les Secrets de manière sécurisée

## Exercice 1 : Types de Secrets

### 1.1 Secret Opaque (générique)
```bash
# Création depuis la ligne de commande
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=t0p-Secret

# Équivalent YAML
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: t0p-Secret
```

### 1.2 Secret TLS
```bash
# Création depuis des fichiers
kubectl create secret tls tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key

# Équivalent YAML
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: base64encoded-cert-content
  tls.key: base64encoded-key-content
```

### 1.3 Secret Docker Registry
```bash
kubectl create secret docker-registry regcred \
  --docker-server=<your-registry-server> \
  --docker-username=<your-username> \
  --docker-password=<your-password> \
  --docker-email=<your-email>
```

## Exercice 2 : Utilisation des Secrets

### 2.1 Variables d'environnement
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

### 2.2 Montage en volume
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-vol-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
```

### 2.3 Pull Secret pour images privées
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-reg-pod
spec:
  containers:
  - name: private-app
    image: private-registry.io/app:v1
  imagePullSecrets:
  - name: regcred
```

## Exercice 3 : Cas Pratique - Application Web Sécurisée

1. Création des secrets nécessaires :
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: web-secrets
type: Opaque
stringData:
  api-key: "my-super-secret-key"
  api-token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  ssl.conf: |
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
```

2. Déploiement utilisant les secrets :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-web
  template:
    metadata:
      labels:
        app: secure-web
    spec:
      containers:
      - name: web
        image: nginx
        volumeMounts:
        - name: ssl-config
          mountPath: /etc/nginx/ssl.conf
          subPath: ssl.conf
        env:
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: web-secrets
              key: api-key
        - name: API_TOKEN
          valueFrom:
            secretKeyRef:
              name: web-secrets
              key: api-token
      volumes:
      - name: ssl-config
        secret:
          secretName: web-secrets
```

## Exercice 4 : Bonnes Pratiques de Sécurité

### 4.1 Limitation d'accès avec RBAC
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  resourceNames: ["web-secrets"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
subjects:
- kind: ServiceAccount
  name: web-service-account
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### 4.2 Rotation des secrets
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    metadata:
      annotations:
        checksum/secret: ${SECRET_CHECKSUM}  # À remplacer par un script
    spec:
      containers:
      # ... configuration du conteneur
```

## Points à Noter

1. Sécurité :
- Les secrets sont encodés en base64 (non chiffrés) par défaut
- Limiter l'accès avec RBAC
- Utiliser des outils de gestion de secrets externes pour la production
- Ne jamais commiter les secrets dans le code source

2. Bonnes pratiques :
- Utiliser stringData au lieu de data pour plus de lisibilité
- Toujours monter les secrets en readOnly
- Préférer les volumes aux variables d'environnement
- Implémenter la rotation des secrets

3. Limitations :
- Taille maximale de 1MB
- Les secrets sont stockés en clair dans etcd
- Les mises à jour ne sont pas propagées automatiquement aux variables d'environnement

## Vérification et Debug

```bash
# Voir les secrets
kubectl get secrets

# Décoder un secret
kubectl get secret db-credentials -o jsonpath='{.data.username}' | base64 --decode

# Vérifier le montage dans un pod
kubectl exec secret-vol-pod -- ls -l /etc/secrets

# Vérifier les variables d'environnement
kubectl exec secret-env-pod -- env
```

## Exercice Bonus : Gestion des Secrets en Production

1. Mettre en place une solution de gestion de secrets externe
   - Vault
   - AWS Secrets Manager
   - Azure Key Vault

2. Implémenter une stratégie de rotation automatique des secrets

3. Mettre en place un audit des accès aux secrets

## Nettoyage
```bash
kubectl delete secret db-credentials tls-secret regcred web-secrets
kubectl delete pod secret-env-pod secret-vol-pod private-reg-pod
kubectl delete deployment secure-web-app web-app
kubectl delete role secret-reader
kubectl delete rolebinding read-secrets
```
