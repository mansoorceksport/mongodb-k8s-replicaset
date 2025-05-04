# MongoDB Replica Set on Kubernetes with Minikube

This guide helps you set up a MongoDB Replica Set (3-node) on Kubernetes using Minikube. You'll also learn how to test the replica set using MongoDB Compass from your local machine.

---

## 🚀 Prerequisites

* [Minikube](https://minikube.sigs.k8s.io/docs/start/) installed and running
* `kubectl` CLI installed
* MongoDB Compass installed on your local machine

---

## 1. 📦 Create Namespace

```bash
kubectl create namespace app-sandbox
```

---

## 2. 🛠 Create Headless Service

```bash
kubectl create service clusterip mongodb \
  --tcp=27017:27017 \
  --clusterip=None \
  -n app-sandbox
```

---

## 3. 🏗 Deploy MongoDB StatefulSet (3 Pods)

Save this YAML as `mongodb-statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: app-sandbox
spec:
  serviceName: "mongodb"
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
          image: mongo:8.0
          command: ["mongod", "--replSet", "rs0", "--bind_ip_all"]
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: data
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 2Gi
```

Apply it:

```bash
kubectl apply -f mongodb-statefulset.yaml
```

Wait for the pods to be ready:

```bash
kubectl get pods -n app-sandbox -w
```

---

## 4. 🚀 Initialize the Replica Set

Save this YAML as `mongodb-init-job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mongodb-init
  namespace: app-sandbox
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: initiator
          image: mongo:5.0
          command: ["bash", "-c"]
          args:
            - |
              sleep 20;
              echo "Initiating replica set...";
              mongo --host mongodb-0.mongodb.app-sandbox.svc.cluster.local:27017 <<EOF
              rs.initiate({
                _id: "rs0",
                members: [
                  { _id: 0, host: "mongodb-0.mongodb.app-sandbox.svc.cluster.local:27017" },
                  { _id: 1, host: "mongodb-1.mongodb.app-sandbox.svc.cluster.local:27017" },
                  { _id: 2, host: "mongodb-2.mongodb.app-sandbox.svc.cluster.local:27017" }
                ]
              });
              EOF
```

Apply it:

```bash
kubectl apply -f mongodb-init-job.yaml
```

Check the logs:

```bash
kubectl logs -f job/mongodb-init -n app-sandbox
```

You should see `ok: 1` which means the replica set was initiated successfully.

---

## 5. 🔌 Port Forward to Your Local Machine

To access all replica set nodes locally:

```bash
kubectl port-forward -n app-sandbox pod/mongodb-0 27017:27017 &
kubectl port-forward -n app-sandbox pod/mongodb-1 27018:27017 &
kubectl port-forward -n app-sandbox pod/mongodb-2 27019:27017 &
```

---

## 6. 🧪 Test with MongoDB Compass

Open MongoDB Compass and connect using this connection string:

```
mongodb://localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0
```

* You should see the replica set members.
* `mongodb-0` should show as the **Primary**.
* `mongodb-1` and `mongodb-2` should be **Secondaries**.

---

## ✅ Success!

You now have a working MongoDB Replica Set on Kubernetes with Minikube — accessible from MongoDB Compass.

Happy hacking! 🎉
