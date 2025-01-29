# Les StatefulSets dans Kubernetes

## Exercice Pratique : Cluster MongoDB

### 1. Service Headless

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
```

### 2. StatefulSet MongoDB

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb-service
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:4.4
          command: [ "mongod", "--replSet", "rs0", "--bind_ip_all" ]
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: data
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 10Gi
```

### 3. Initialisation du Replica Set

```bash
kubectl exec -it mongodb-1 -ti -- mongo
```

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongodb-0.mongodb-service:27017" },
    { _id: 1, host: "mongodb-1.mongodb-service:27017" },
    { _id: 2, host: "mongodb-2.mongodb-service:27017" }
  ]
})
```

### 4. Commandes utiles

```bash
# Voir les StatefulSets
kubectl get statefulset

# Détails d'un StatefulSet
kubectl describe statefulset mongodb

# Voir les PVCs associés
kubectl get pvc

# DNS lookup pour un pod
kubectl exec -it mongodb-0 -- nslookup mongodb-0.mongodb-service

# Voir les logs d'un pod spécifique
kubectl logs mongodb-0

#######
# Mise à jour par phases

# Phase 1 : Test initial
kubectl patch statefulset mongodb -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'

# Phase 2 : Extension du déploiement
kubectl patch statefulset mongodb -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":1}}}}'

# Phase 3 : Déploiement complet
kubectl patch statefulset mongodb -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'

```