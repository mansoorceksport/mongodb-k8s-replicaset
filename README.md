# MongoDB Replica Set on Kubernetes with Minikube

This guide helps you set up a MongoDB Replica Set (3-node) on Kubernetes using Minikube. You'll also learn how to test the replica set using MongoDB Compass from your local machine.

---

## 🚀 Prerequisites

* [Minikube](https://minikube.sigs.k8s.io/docs/start/) installed and running
* `kubectl` CLI installed
* MongoDB Compass installed on your local machine

---

## 📖 Why StatefulSet, Headless Service, and volumeClaimTemplates?

### 🔹 What is a StatefulSet?

A **StatefulSet** is a Kubernetes controller used to manage stateful applications. Unlike a Deployment, a StatefulSet:

* Provides **stable network identity** (e.g., `mongodb-0`, `mongodb-1`)
* Maintains **stable, persistent storage** via PVCs
* Ensures **ordered pod startup and termination**

MongoDB replica sets rely on predictable hostnames and consistent storage, which is why we use a StatefulSet instead of a regular Deployment.

### 🔸 What is a Headless Service?

A **Headless Service** in Kubernetes is a service with `clusterIP: None`. It:

* Doesn't load balance or proxy traffic
* Instead, returns the list of individual pod IPs directly via DNS

This is crucial for MongoDB replica sets, which need to know each other's hostnames and connect directly to peers.

### 🧠 Understanding `mongodb-0.mongodb.app-sandbox.svc.cluster.local:27017`

This is the **fully qualified domain name (FQDN)** of a MongoDB pod within the Kubernetes cluster. Let's break it down:

```
mongodb-0.mongodb.app-sandbox.svc.cluster.local:27017
```

| Part                | Meaning                                 |
| ------------------- | --------------------------------------- |
| `mongodb-0`         | Pod name (first pod in the StatefulSet) |
| `mongodb`           | Headless service name                   |
| `app-sandbox`       | Namespace                               |
| `svc.cluster.local` | Default Kubernetes service domain       |
| `:27017`            | MongoDB default port                    |

Together, this forms a stable DNS address that MongoDB can use to identify and connect to replica set members inside the cluster.

### 💾 What is `volumeClaimTemplates` and Why Is It Important?

In a StatefulSet, `volumeClaimTemplates` defines how **persistent volumes** are provisioned for each pod automatically. Here's why it matters:

* Each pod gets its **own dedicated PersistentVolumeClaim (PVC)** (e.g., `data-mongodb-0`, `data-mongodb-1`)
* The storage is **stable across restarts** — even if the pod is rescheduled to a different node
* MongoDB stores all data in `/data/db`, so without persistent volumes, you'd lose your data when pods restart or move

Without `volumeClaimTemplates`, you'd need to manually provision volumes for each pod, which defeats the purpose of using StatefulSets for scalable, dynamic storage management.

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
