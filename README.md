# Kubernetes PoC - Running a NestJS App with PostgreSQL

This repository contains a **Kubernetes Proof of Concept (PoC)** for running a **NestJS-based logging API** connected to a **PostgreSQL database** inside a **K3d (K3s in Docker) cluster**.

---

## 📌 What is Kubernetes?

Kubernetes (K8s) is an open-source container orchestration system that automates **deployments, scaling, and management of containerized applications**. It helps manage clusters of nodes and ensures applications run efficiently and reliably.

In this PoC, Kubernetes is used to:

- Deploy a **NestJS API** inside a Kubernetes **Deployment**.
- Manage a **PostgreSQL database** within a **Kubernetes cluster**.
- Expose the API to the host machine using **LoadBalancer** and **NodePort**.
- Store environment variables using **ConfigMaps**.
- Apply **automated database migrations** using a **Kubernetes Job**.

---

## 🔹 Key Kubernetes Components Used

### 1️⃣ Namespace

Namespaces allow organizing Kubernetes resources into separate environments. In this project, we define a `logger-k8s` namespace for isolating resources.

### 2️⃣ Deployments

A **Deployment** defines how the API and database containers should run. It manages the lifecycle, ensures availability, and enables rolling updates.

- `api/deployment.yaml` → Deploys the NestJS logger API.
- `database/deployment.yaml` → Deploys the PostgreSQL database.

### 3️⃣ Services

A **Service** provides stable network access to a set of pods. We use:

- **ClusterIP** for the **database** (only accessible inside the cluster).
- **LoadBalancer** for the **API** (exposes the service externally).

### 4️⃣ Persistent Volume

A **Persistent Volume Claim (PVC)** ensures that PostgreSQL retains data across pod restarts.

### 5️⃣ ConfigMap

A **ConfigMap** stores environment variables required by the API and database. This must be applied **before** deploying the services.

---

## ⚙️ Configuring Environment Variables

Before deploying the API and database, ensure that the necessary environment variables are set using a **ConfigMap**. This allows Kubernetes to inject environment variables into the containers dynamically.

```sh
cp api/configmap.yaml.example api/configmap.yaml
```

Then edit the file to include real values.

```yaml
DATABASE_URL: Url to the Database
POSTGRES_DB: Database name
POSTGRES_USER: Database user name
POSTGRES_PASSWORD: Database password
POSTGRES_HOST: Name of the host of the DB, ex. database-logger
POSTGRES_PORT: Database port
POSTGRES_SCHEMA: Database schema, usually 'public'
PORT: Logger port
```

This step is crucial to ensure that **environment variables** are injected into the database and API containers before they start.

---

## 🚀 Steps to Run the `kubernetes-poc` Locally

### **1️⃣ Start the Kubernetes Cluster (K3d)**

We use **K3d**, a lightweight Kubernetes distribution running in Docker:

```sh
k3d cluster create kubernetes-poc-cluster --port "8000:8000@loadbalancer"
```

### **2️⃣ Apply Kubernetes Configuration**

Run the following commands **in order**:

```sh
kubectl apply -f namespace.yaml
kubectl apply -f database/pvc.yaml
kubectl apply -f api/configmap.yaml
kubectl apply -f database/deployment.yaml
kubectl apply -f database/service.yaml
kubectl apply -f api/migrate-job.yaml
kubectl wait --for=condition=complete job/prisma-migrate-job -n logger-k8s
kubectl apply -f api/deployment.yaml
kubectl apply -f api/service.yaml
```

### **3️⃣ Verify Running Pods and Services**

```sh
kubectl get pods -n logger-k8s
kubectl get svc -n logger-k8s
```

You should see the API and database running as separate pods.

### **3️⃣ Verify the ConfigMap**

Ensure that the ConfigMap is correctly applied:

```sh
kubectl get configmap logger-config -n logger-k8s -o yaml
```

### **5️⃣ Check Migration Logs (Optional)**

Once the migration job is complete, you can inspect its logs with:

```sh
kubectl logs job/prisma-migrate-job -n logger-k8s
```

---

## 📡 Accessing the API

### **1️⃣ Logs**

```sh
curl http://localhost:8000/logs
```

### **2️⃣ Loadbalancing check**

```sh
curl http://localhost:8000
```

Example output:

```
curl http://localhost:8000
Hello from instance: logger-f5685948d-9v8mb%
```

```
curl http://localhost:8000
Hello from instance: logger-f5685948d-ct5rg%
```

---

## 🔀 Next Steps: Add Ingress (NGINX)

For better traffic routing, consider adding **NGINX Ingress**:

```sh
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
```

This allows exposing the API via a cleaner URL (`http://logger.local`).

---

## 🎯 Conclusion

This PoC demonstrates **how to deploy a NestJS API with PostgreSQL inside Kubernetes using K3d**. It includes **Deployments, Services, Persistent Volumes, and Load Balancing**.

For future improvements: ✅ Add **Ingress** for cleaner routing.\
✅ Implement **Auto-scaling** based on CPU/memory usage.\
✅ Add **Secrets** instead of ConfigMaps for sensitive data.

Enjoy experimenting with Kubernetes! 🚀
