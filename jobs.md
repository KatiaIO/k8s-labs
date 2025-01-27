# Jobs dans Kubernetes

## Objectifs du Lab

- Créer et gérer différents types de Jobs
- Comprendre les paramètres des Jobs
- Gérer les Jobs parallèles
- Manipuler les CronJobs

## Prérequis

- Accès à un cluster Kubernetes
- kubectl configuré
- Namespace dédié pour les exercices

## Exercice 1 : Job Simple

1. Créer un Job basique qui affiche la date :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: print-date
spec:
  template:
    spec:
      containers:
        - name: print-date
          image: busybox
          command: [ "sh", "-c", "date; echo Job terminé!" ]
      restartPolicy: Never
```

2. Appliquer et vérifier :

```bash
kubectl apply -f print-date-job.yaml
kubectl get jobs
kubectl get pods
kubectl logs -l job-name=print-date
```

## Exercice 2 : Job avec Retry

1. Créer un Job qui simule des échecs :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: retry-job
spec:
  backoffLimit: 4
  template:
    spec:
      containers:
        - name: retry-container
          image: busybox
          command:
            - sh
            - -c
            - "if [ $RANDOM -gt 20000 ]; then echo 'Succès!'; else echo 'Échec!' && exit 1; fi"
      restartPolicy: Never
```

2. Observer le comportement :

```bash
kubectl get jobs retry-job -w
kubectl describe job retry-job
```

## Exercice 3 : Job Parallèle

1. Créer un Job qui s'exécute en parallèle :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 5
  parallelism: 2
  template:
    spec:
      containers:
        - name: worker
          image: busybox
          command:
            - sh
            - -c
            - "echo Processing item $RANDOM; sleep 5"
      restartPolicy: Never
```

2. Observer l'exécution parallèle :

```bash
kubectl get pods -w
kubectl get jobs parallel-job
```

## Exercice 4 : CronJob

1. Créer un CronJob qui s'exécute toutes les minutes :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/1 * * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: hello
              image: busybox
              command:
                - sh
                - -c
                - "echo 'Exécution à' $(date)"
          restartPolicy: Never
```

2. Observer les exécutions :

```bash
kubectl get cronjobs
kubectl get jobs
kubectl get pods
```

## Exercice 5 : Job de Traitement de Données

1. Créer un Job qui simule le traitement de données :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor
spec:
  completions: 3
  parallelism: 2
  template:
    spec:
      containers:
        - name: processor
          image: python:3.9-alpine
          command:
            - python
            - -c
            - |
              import time
              import random
              import os

              job_id = os.environ.get('JOB_COMPLETION_INDEX', '0')
              print(f"Starting processing for batch {job_id}")

              # Simuler un traitement
              processing_time = random.randint(3, 8)
              time.sleep(processing_time)

              print(f"Batch {job_id} completed in {processing_time} seconds")
      restartPolicy: Never
```

2. Suivre l'avancement :

```bash
kubectl get jobs data-processor -w
kubectl logs -l job-name=data-processor
```

## Nettoyage

Supprimer toutes les ressources créées :

```bash
kubectl delete job print-date
kubectl delete job retry-job
kubectl delete job parallel-job
kubectl delete job data-processor
kubectl delete cronjob hello-cron
```

## Points Clés à Retenir

1. Les Jobs sont utiles pour les tâches ponctuelles
2. backoffLimit contrôle le nombre de tentatives
3. completions et parallelism permettent de gérer des tâches parallèles
4. Les CronJobs permettent d'automatiser l'exécution périodique
5. Toujours définir une politique de rétention pour les jobs terminés

## Exercices Bonus

1. Modifier le CronJob pour qu'il garde plus d'historique
2. Créer un Job avec un timeout (activeDeadlineSeconds)
3. Créer un Job qui utilise des variables d'environnement
4. Implémenter un Job avec une logique de traitement par lots

## Troubleshooting

En cas de problèmes :

```bash
kubectl describe job <nom-du-job>
kubectl get events
kubectl logs <pod-name>
```