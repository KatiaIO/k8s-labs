# Exercice Pratique : Init Containers

## Objectif
Créer un pod avec un init container qui vérifie si un service est disponible avant de démarrer le conteneur principal.

## Scénario
Vous avez une application web (nginx) qui nécessite qu'un service backend soit disponible avant de démarrer. L'init container va vérifier la disponibilité du service backend avant de permettre le démarrage du conteneur principal.

## Étapes

1. Créer d'abord un service backend (pour simuler la dépendance) :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  selector:
    app: backend
```

2. Créer le pod avec l'init container :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  initContainers:
  - name: init-service-check
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup backend-service; do echo "Waiting for backend service"; sleep 2; done;']
  containers:
  - name: web-app
    image: nginx
    ports:
    - containerPort: 80
```

## Instructions

1. Sauvegarder le premier YAML dans un fichier `backend-service.yaml`
2. Sauvegarder le second YAML dans un fichier `web-pod.yaml`
3. Appliquer les configurations :
```bash
kubectl apply -f backend-service.yaml
kubectl apply -f web-pod.yaml
```

4. Observer le comportement :
```bash
kubectl get pod web-pod
kubectl describe pod web-pod
```

## Comportement Attendu

- L'init container va démarrer et chercher le service backend-service
- Comme le service existe mais n'a pas de endpoints (pas de pods backend), l'init container va quand même réussir (car nslookup trouve le service)
- Le conteneur principal (nginx) démarrera une fois que l'init container aura terminé avec succès

## Pour Aller Plus Loin

Modifiez l'exercice en :
1. Ajoutant une vérification plus stricte dans l'init container (par exemple, vérifier que le service répond sur son port)
2. Créant un vrai backend pour le service
3. Ajoutant plusieurs init containers en chaîne

## Nettoyage

```bash
kubectl delete pod web-pod
kubectl delete service backend-service
```

## Points Clés à Retenir

- Les init containers s'exécutent en séquence avant le conteneur principal
- Ils doivent se terminer avec succès pour que le pod démarre
- Ils sont utiles pour la vérification des dépendances
- Une fois terminés, ils ne redémarrent pas