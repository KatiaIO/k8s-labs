# Introduction à Helm

## Objectifs

- Comprendre les concepts de base de Helm
- Utiliser des charts existants
- Créer un chart simple
- Gérer les releases

## Exercice 1 : Utilisation de Base

1. Ajouter un repository populaire :

```bash
# Ajouter le repo bitnami
helm repo add bitnami https://charts.bitnami.com/bitnami

# Mettre à jour les repos
helm repo update
```

2. Chercher un chart :

```bash
# Rechercher nginx
helm search repo nginx
```

3. Installer un chart :

```bash
# Installer nginx
helm install my-nginx bitnami/nginx

# Vérifier l'installation
kubectl get all
```

## Exercice 2 : Personnalisation de Values

1. Examiner les values disponibles :

```bash
# Voir les values par défaut
helm show values bitnami/nginx > values.yaml
```

2. Installer avec des values personnalisées :

```bash
# Créer un fichier custom-values.yaml
cat <<EOF > custom-values.yaml
replicaCount: 2
service:
  type: NodePort
EOF

# Installer avec les values personnalisées
helm install my-release bitnami/nginx -f custom-values.yaml
```

## Exercice 3 : Créer un Chart Simple

1. Créer un nouveau chart :

```bash
# Créer la structure du chart
helm create my-app
```

2. Modifier le chart pour une application web simple :

Éditer `my-app/values.yaml` :

```yaml
replicaCount: 1

image:
  repository: nginx
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
```

Éditer `my-app/templates/deployment.yaml` (garder uniquement l'essentiel) :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 80
```

3. Installer le chart :

```bash
# Vérifier la syntaxe
helm lint my-app

# Installer le chart
helm install my-release my-app

# Vérifier l'installation
kubectl get all
```

## Exercice 4 : Gestion des Releases

```bash
# Lister les releases
helm list

# Voir le statut d'une release
helm status my-release

# Mettre à jour une release
helm upgrade my-release my-app --set replicaCount=3

# Rollback si nécessaire
helm rollback my-release 1

# Désinstaller une release
helm uninstall my-release
```

## Exercice 5 : Debug et Test

```bash
# Voir ce qui sera installé
helm template my-app

# Vérifier l'installation
helm test my-release

# Voir l'historique des releases
helm history my-release
```

## Nettoyage

```bash
# Supprimer toutes les releases
helm uninstall my-nginx my-release

# Supprimer les repos ajoutés
helm repo remove bitnami
```

## Points Clés à Retenir

1. Concepts de Base :

- Chart : Package Helm
- Release : Instance d'un chart installé
- Repository : Collection de charts
- Values : Configuration paramétrable

2. Commandes Essentielles :

- helm install
- helm upgrade
- helm rollback
- helm uninstall

3. Bonnes Pratiques :

- Toujours vérifier les values par défaut
- Utiliser des fichiers values pour la configuration
- Versionner les charts personnalisés
- Tester avant déploiement en production