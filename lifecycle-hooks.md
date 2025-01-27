# Les Hooks Lifecycle : preStop et postStart

## Introduction

Les hooks lifecycle permettent d'exécuter du code à des moments spécifiques du cycle de vie d'un conteneur :
- `postStart` : exécuté immédiatement après la création du conteneur
- `preStop` : exécuté juste avant la terminaison du conteneur

## Le Hook postStart

### Caractéristiques
- Exécuté de manière asynchrone avec l'ENTRYPOINT du conteneur
- Pas de garantie d'ordre d'exécution avec l'ENTRYPOINT
- En cas d'échec, le conteneur est tué

### Cas d'usage typiques
- Initialisation d'une configuration
- Enregistrement du service
- Vérification de prérequis
- Configuration de l'environnement

### Exemple Simple
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: poststart-demo
spec:
  containers:
  - name: web
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from postStart > /usr/share/message"]
```

### Exemple Avancé avec Notification
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: poststart-notification
spec:
  containers:
  - name: app
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: 
          - /bin/sh
          - -c
          - |
            curl -X POST -H "Content-Type: application/json" \
            -d '{"text":"Container started successfully"}' \
            http://notification-service:8080/notify
```

## Le Hook preStop

### Caractéristiques
- Bloquant : le conteneur n'est pas terminé tant que le hook n'est pas terminé
- Inclus dans le délai de grâce de terminaison du pod
- Exécuté avant l'envoi du signal SIGTERM

### Cas d'usage typiques
- Arrêt gracieux des connexions
- Sauvegarde de l'état
- Notification de désenregistrement
- Nettoyage des ressources

### Exemple d'Arrêt Gracieux Nginx
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prestop-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "nginx -s quit && while killall -0 nginx; do sleep 1; done"]
```

### Exemple Complet avec postStart et preStop
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo
    image: nginx
    ports:
    - containerPort: 80
    lifecycle:
      postStart:
        exec:
          command: 
          - /bin/sh
          - -c
          - |
            echo "Service starting at $(date)" >> /var/log/startup.log
            curl -X POST http://monitoring:8080/service/start
      preStop:
        exec:
          command: 
          - /bin/sh
          - -c
          - |
            echo "Service stopping at $(date)" >> /var/log/shutdown.log
            curl -X POST http://monitoring:8080/service/stop
            sleep 5 # Permettre aux requêtes en cours de se terminer
```

## Méthodes d'Handlers

Les hooks supportent trois types d'handlers :

### 1. Exec
```yaml
lifecycle:
  postStart:
    exec:
      command: ["/bin/sh", "-c", "echo Hello"]
```

### 2. HTTP
```yaml
lifecycle:
  postStart:
    httpGet:
      path: /health
      port: 8080
      httpHeaders:
      - name: Custom-Header
        value: value
```

### 3. TCP Socket
```yaml
lifecycle:
  postStart:
    tcpSocket:
      port: 8080
```

## Bonnes Pratiques

### Pour postStart
1. Garder les opérations courtes et légères
2. Gérer les échecs potentiels
3. Ne pas bloquer le démarrage du conteneur
4. Logger les actions importantes

### Pour preStop
1. Prévoir un timeout approprié
2. S'assurer que le nettoyage est complet
3. Gérer les erreurs gracieusement
4. Coordonner avec le délai de grâce

## Exemple d'Application Web Complète
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
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
        ports:
        - containerPort: 80
        lifecycle:
          postStart:
            exec:
              command: 
              - /bin/sh
              - -c
              - |
                # Configuration initiale
                echo "Initializing at $(date)" >> /var/log/lifecycle.log
                # Vérification de la santé
                curl --retry 5 --retry-delay 2 http://localhost:80/health
                # Notification de démarrage
                curl -X POST http://monitor:8080/startup

          preStop:
            exec:
              command: 
              - /bin/sh
              - -c
              - |
                # Notification de l'arrêt imminent
                curl -X POST http://monitor:8080/prestop
                # Attendre que les connexions existantes se terminent
                sleep 10
                # Arrêt gracieux de nginx
                nginx -s quit
                # Attendre l'arrêt complet
                while killall -0 nginx; do sleep 1; done
                # Log final
                echo "Shutdown complete at $(date)" >> /var/log/lifecycle.log
```

## Points à Noter

1. Garanties de Livraison
- Les hooks sont exécutés "au moins une fois"
- Plusieurs exécutions sont possibles
- Prévoir une idempotence

2. Gestion des Erreurs
- Un échec de postStart tue le conteneur
- Un échec de preStop est loggé mais n'empêche pas la terminaison

3. Debuggage
- Utiliser `kubectl describe pod` pour voir les événements
- Vérifier les logs du conteneur
- Surveiller les timeouts

4. Limitations
- Pas d'arguments passés aux handlers
- Pas de retransmission en cas d'échec HTTP
- Temps d'exécution limité par le délai de grâce