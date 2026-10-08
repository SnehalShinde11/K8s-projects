# Project 3.3 - Kubernetes Deployment with PV, ConfigMap, Secrets, and Services

## 1. Project Overview

The objective of this project is to implement a complete Kubernetes application deployment using core Kubernetes resources.

Modern Kubernetes applications rely on multiple resources such as **Deployments, Services, Persistent Volumes, ConfigMaps, and Secrets** to run properly. These components help manage application configuration, storage, scaling, and communication.

By completing this project, learners will deploy a containerized application that uses persistent storage, external configuration, and secure credentials within a Kubernetes cluster.

In this project, **PostgreSQL 17** is used as the containerized application because it requires persistent storage for database data and configuration values and credentials during runtime.

The application is deployed using a Kubernetes **Deployment**, with:

- Persistent storage using a Persistent Volume Claim (PVC)
- GKE dynamic Persistent Volume (PV) provisioning
- ConfigMap for non-sensitive configuration
- Secret for sensitive database credentials
- PostgreSQL 17 container image
- ClusterIP Service for internal database access

The project also demonstrates an important GKE-specific storage consideration. Instead of manually creating a `hostPath` Persistent Volume, the GKE-compatible implementation uses the `standard-rwo` StorageClass to dynamically provision the Persistent Disk.

The final architecture is:

```text
PVC
 │
 ▼
GKE standard-rwo StorageClass
 │
 ▼
Dynamically Provisioned PV
 │
 ▼
PostgreSQL Deployment
 │
 ├── ConfigMap
 ├── Secret
 └── Persistent Storage
 │
 ▼
ClusterIP Service
 │
 ▼
PostgreSQL :5432
```

---

# 2. Application Used

For this project, PostgreSQL 17 is used as the containerized application.

Docker image:

```text
postgres:17
```

PostgreSQL requires persistent storage because database data must survive Pod recreation.

The application also requires:

- Database name
- Database username
- Database password
- Persistent data directory

These values are provided using Kubernetes ConfigMap, Secret, and PVC resources.

---

# 3. Repository Structure

The Project 3.3 directory has already been created as part of the project setup.

Navigate to the existing Kubernetes directory:

```text
kubernetes-web-app/
│
└── project-3.3-postgres/
    │
    └── k8s/
        ├── pv-hostpath.yaml
        ├── pvc-hostpath.yaml
        ├── pvc.yaml
        ├── error-deployment.yaml
        ├── deployment.yaml
        ├── configmap.yaml
        ├── secret.yaml
        └── service.yaml
```

### Manifest Overview

| File | Purpose |
|---|---|
| `pv-hostpath.yaml` | Preserved hostPath PV used during the initial storage approach |
| `pvc-hostpath.yaml` | Preserved PVC associated with the initial hostPath approach |
| `pvc.yaml` | GKE-compatible PVC using dynamic provisioning |
| `error-deployment.yaml` | Preserved Deployment showing the PostgreSQL `lost+found` error scenario |
| `deployment.yaml` | Final PostgreSQL Deployment using `PGDATA` subdirectory |
| `configmap.yaml` | Stores non-sensitive PostgreSQL configuration |
| `secret.yaml` | Stores sensitive PostgreSQL credentials |
| `service.yaml` | Creates the internal ClusterIP Service |

---

# 4. Dependencies

The following components are required to implement and test this project:

| Dependency | Purpose |
|---|---|
| Google Kubernetes Engine (GKE) | Kubernetes cluster used for deployment |
| `kubectl` | Command-line tool for interacting with Kubernetes |
| PostgreSQL 17 | Database application used in the project |
| Docker Hub | Source of the publicly available `postgres:17` image |
| GKE `standard-rwo` StorageClass | Dynamically provisions persistent storage |
| Persistent Disk CSI Driver | Provides persistent storage for GKE workloads |
| Kubernetes ConfigMap | Stores non-sensitive application configuration |
| Kubernetes Secret | Stores sensitive database credentials |

### Kubernetes Resources Used

- Persistent Volume
- Persistent Volume Claim
- StorageClass
- Deployment
- ReplicaSet
- Pod
- ConfigMap
- Secret
- Service

> **Important:** For GKE, the final implementation does not manually create a `hostPath` Persistent Volume. The PVC uses the GKE `standard-rwo` StorageClass, which dynamically provisions the required Persistent Disk-backed PV.

---

# 5. Execution Steps

## Step 1 — Prepare the Application Container

For this project, PostgreSQL 17 is used as the containerized application.

The image is publicly available:

```text
postgres:17
```

The Project 3.3 directory has already been created.

Navigate to the existing project directory:

```bash
cd ~/kubernetes-web-app/project-3.3-postgres/k8s
```

Verify the current directory:

```bash
pwd
```

Verify the existing project files:

```bash
ls
```

The Kubernetes manifests for Project 3.3 will be maintained inside this existing directory.

> **Note:** Do not create another `project-3.3-postgres/k8s` directory. Continue using the existing directory and modify or add the required Kubernetes manifest files there.

---

# 6. Task 2 — Create Persistent Volume

The initial approach used a manually created `hostPath` Persistent Volume.

The original file was:

```text
pv.yaml
```

After testing the initial approach, it was renamed and preserved as:

```text
pv-hostpath.yaml
```

### Why the hostPath approach was changed

A `hostPath` Persistent Volume is generally intended for local or single-node Kubernetes environments.

For a GKE cluster, manually creating a `hostPath` PV is not the appropriate storage approach.

GKE provides dynamic provisioning through StorageClasses.

Therefore, the final implementation uses:

```text
PVC
  │
  ▼
standard-rwo StorageClass
  │
  ▼
GKE Persistent Disk
  │
  ▼
Dynamically Provisioned PV
```

The original hostPath file is retained as:

```text
pv-hostpath.yaml
```

This allows the initial implementation and the GKE-compatible implementation to be clearly distinguished.

---

# 7. Task 3 — Create Persistent Volume Claim

For GKE, a manually created `pv.yaml` is not required.

Instead, the application uses a PVC that requests storage from the GKE `standard-rwo` StorageClass.

The GKE-compatible PVC is stored in:

```text
pvc.yaml
```

### Verify the StorageClass

```bash
kubectl get storageclass standard-rwo
```

You can also list all StorageClasses:

```bash
kubectl get storageclass
```

### Apply the PVC

```bash
kubectl apply -f pvc.yaml
```

Verify the PVC:

```bash
kubectl get pvc
```

You can also verify both PV and PVC:

```bash
kubectl get pv,pvc
```

### PVC Pending State

Initially, the PVC may show:

```text
Pending
```

This is expected when using the GKE `standard-rwo` StorageClass because it uses:

```text
VolumeBindingMode: WaitForFirstConsumer
```

This means the Persistent Disk is provisioned when a Pod that uses the PVC is scheduled.

The expected flow is:

```text
PVC Created
     │
     ▼
PVC Pending
     │
     ▼
Pod Using PVC Is Scheduled
     │
     ▼
GKE Provisions Persistent Disk
     │
     ▼
PV Created
     │
     ▼
PVC Bound
```

---

# 8. Task 4 — Create ConfigMap

A ConfigMap is used to store non-sensitive configuration values.

For the PostgreSQL application, configuration such as the database name and username can be provided through the ConfigMap.

The configuration file is:

```text
configmap.yaml
```

Apply the ConfigMap:

```bash
kubectl apply -f configmap.yaml
```

Verify the ConfigMap:

```bash
kubectl get configmap postgres-config
```

To view the complete ConfigMap:

```bash
kubectl describe configmap postgres-config
```

### ConfigMap Purpose

The ConfigMap provides configuration separately from the container image.

This allows application configuration to be changed without rebuilding the PostgreSQL container image.

---

# 9. Task 5 — Create Kubernetes Secret

Sensitive information should not be stored directly in a ConfigMap.

A Kubernetes Secret is used for sensitive values such as:

- Database password
- Authentication credentials
- API keys
- Authentication tokens

For this project, the PostgreSQL credentials are stored using:

```text
secret.yaml
```

Apply the Secret:

```bash
kubectl apply -f secret.yaml
```

Verify the Secret:

```bash
kubectl get secret postgres-secret
```

You can inspect the Secret metadata using:

```bash
kubectl describe secret postgres-secret
```

> **Note:** Kubernetes Secrets are designed for sensitive configuration, but they are not automatically encrypted in every Kubernetes setup. Access to Secrets should therefore be controlled using appropriate Kubernetes permissions.

---

# 10. Task 6 — Deploy Application Using Deployment

The PostgreSQL application is deployed using:

```text
deployment.yaml
```

The Deployment:

- Uses the `postgres:17` image
- Uses the PostgreSQL PVC
- Loads configuration from the ConfigMap
- Loads credentials from the Secret
- Mounts persistent storage
- Uses a PostgreSQL data subdirectory

### Apply the Deployment

```bash
kubectl apply -f deployment.yaml
```

Verify the Deployment:

```bash
kubectl get deployments
```

Verify the PostgreSQL Pods:

```bash
kubectl get pods -l app=postgres
```

Watch the Pod:

```bash
kubectl get pods -l app=postgres -w
```

Verify storage:

```bash
kubectl get pv,pvc
```

Check PostgreSQL logs:

```bash
kubectl logs -l app=postgres
```

---

## 10.1 PostgreSQL `lost+found` Error

During the initial deployment, PostgreSQL may fail with an error similar to:

```text
initdb: error: directory "/var/lib/postgresql/data" exists but is not empty
initdb: detail: It contains a lost+found directory
```

### Why this happens

A GKE Persistent Disk filesystem can contain a `lost+found` directory at the root of the mounted filesystem.

PostgreSQL expects its database data directory to be empty during initialization.

Therefore, mounting the PVC directly at:

```text
/var/lib/postgresql/data
```

can cause PostgreSQL initialization to fail.

---

## 10.2 Error Showcase Deployment

The original Deployment that produced the error is preserved as:

```text
error-deployment.yaml
```

This file documents the initial implementation and the PostgreSQL initialization issue.

It is not used as the final Deployment.

---

## 10.3 Final PostgreSQL Data Directory

The final Deployment uses a subdirectory inside the mounted PVC:

```text
/var/lib/postgresql/data/pgdata
```

PostgreSQL is instructed to use this directory through:

```text
PGDATA
```

The storage flow becomes:

```text
GKE Persistent Disk
        │
        ▼
/var/lib/postgresql/data
        │
        ├── lost+found
        │
        └── pgdata
              │
              ▼
       PostgreSQL Database
```

This avoids the `lost+found` initialization problem because PostgreSQL uses the empty `pgdata` subdirectory instead of the root of the mounted filesystem.

---

# 11. Task 7 — Expose Application Using Service

PostgreSQL is a database and only needs to be accessible from inside the Kubernetes cluster.

Therefore, a **ClusterIP Service** is used.

The Service configuration is stored in:

```text
service.yaml
```

Apply the Service:

```bash
kubectl apply -f service.yaml
```

Verify the Service:

```bash
kubectl get service
```

Verify the PostgreSQL Service:

```bash
kubectl get service postgres-service
```

Check the Service endpoints:

```bash
kubectl get endpoints postgres-service
```

The PostgreSQL Service uses:

```text
Port: 5432
Type: ClusterIP
```

The Service provides a stable internal endpoint for applications that need to communicate with PostgreSQL.

---

# 12. Connect to PostgreSQL

First, identify the PostgreSQL Pod:

```bash
kubectl get pods -l app=postgres
```

Example:

```text
postgres-66cdd8c79-6585j
```

Connect to PostgreSQL from inside the Pod:

```bash
kubectl exec -it postgres-66cdd8c79-6585j -- psql -U snehal -d snehaldb
```

Once connected, verify the current database:

```sql
SELECT current_database();
```

Check the PostgreSQL connection:

```sql
\conninfo
```

This confirms that PostgreSQL is running and that the specified database can be accessed.

---

# 13. Task 8 — Verify Application Deployment

The final application should be verified from multiple perspectives.

## Verify PostgreSQL Pod

```bash
kubectl get pods -l app=postgres
```

The Pod should be in the:

```text
Running
```

state.

---

## Verify Persistent Storage

Check both PV and PVC:

```bash
kubectl get pv,pvc
```

The PVC should eventually show:

```text
Bound
```

and a dynamically provisioned PV should be associated with it.

---

## Verify ConfigMap

```bash
kubectl get configmap postgres-config
```

---

## Verify Secret

```bash
kubectl get secret postgres-secret
```

---

## Verify Service

```bash
kubectl get service postgres-service
```

---

## Verify Service Endpoints

```bash
kubectl get endpoints postgres-service
```

The Service should have an endpoint corresponding to the PostgreSQL Pod.

---

## Connect to PostgreSQL

Use:

```bash
kubectl exec -it <pod-name> -- psql -U snehal -d snehaldb
```

For example:

```bash
kubectl exec -it postgres-66cdd8c79-6585j -- psql -U snehal -d snehaldb
```

---

# 14. Test Persistent Storage

The most important test is to confirm that database data survives Pod deletion.

Connect to PostgreSQL:

```bash
kubectl exec -it <pod-name> -- psql -U snehal -d snehaldb
```

For example:

```bash
kubectl exec -it postgres-66cdd8c79-6585j -- psql -U snehal -d snehaldb
```

---

## 14.1 Create a Table

Inside PostgreSQL, create an `employees` table:

```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    role VARCHAR(100)
);
```

---

## 14.2 Insert Data

Insert a sample record:

```sql
INSERT INTO employees (name, role)
VALUES ('Snehal', 'DevOps Engineer');
```

---

## 14.3 Retrieve the Data

Run:

```sql
SELECT * FROM employees;
```

The inserted record should be displayed.

Example:

```text
 id |  name  |       role
----+--------+-------------------
  1 | Snehal | DevOps Engineer
```

---

# 15. Test Data Persistence After Pod Deletion

Now verify that the database data survives Pod recreation.

First, identify the current PostgreSQL Pod:

```bash
kubectl get pods -l app=postgres
```

Delete the current Pod:

```bash
kubectl delete pod <pod-name>
```

For example:

```bash
kubectl delete pod postgres-66cdd8c79-6585j
```

Because the Pod is managed by a Deployment, Kubernetes should automatically create a replacement Pod.

Watch the replacement Pod:

```bash
kubectl get pods -l app=postgres -w
```

Wait until the new Pod reaches:

```text
Running
```

---

# 16. Verify Data Survived Pod Recreation

Get the new PostgreSQL Pod name:

```bash
kubectl get pods -l app=postgres
```

Connect to PostgreSQL using the new Pod:

```bash
kubectl exec -it <new-pod-name> -- psql -U snehal -d snehaldb
```

For example:

```bash
kubectl exec -it postgres-66cdd8c79-kplmf -- psql -U snehal -d snehaldb
```

Once connected, run:

```sql
SELECT * FROM employees;
```

The previously inserted record should still be present.

This confirms that the database data is stored on persistent storage rather than only inside the Pod's temporary filesystem.

---

# 17. Final Verification

Run the following commands to verify the complete application:

### Pods

```bash
kubectl get pods -l app=postgres
```

### Deployment

```bash
kubectl get deployment postgres
```

### Persistent Volume

```bash
kubectl get pv
```

### Persistent Volume Claim

```bash
kubectl get pvc
```

### StorageClass

```bash
kubectl get storageclass standard-rwo
```

### ConfigMap

```bash
kubectl get configmap postgres-config
```

### Secret

```bash
kubectl get secret postgres-secret
```

### Service

```bash
kubectl get service postgres-service
```

### Service Endpoints

```bash
kubectl get endpoints postgres-service
```

### PostgreSQL Logs

```bash
kubectl logs -l app=postgres
```

---

# 18. Kubernetes Resource Flow

```text
                         Kubernetes Cluster
                                │
                                ▼
                        PostgreSQL Deployment
                                │
                                ▼
                              Pod
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
          ConfigMap          Secret             PVC
              │                 │                 │
              │                 │                 ▼
              │                 │        standard-rwo StorageClass
              │                 │                 │
              │                 │                 ▼
              │                 │       Dynamic Persistent Disk
              │                 │                 │
              │                 │                 ▼
              │                 │                PV
              │                 │                 │
              └─────────────────┴─────────────────┘
                                │
                                ▼
                         PostgreSQL Database
                                │
                                ▼
                      ClusterIP Service
                       postgres-service
                              :5432
```

---

# 19. Key Kubernetes Concepts Demonstrated

### Persistent Volume

A Persistent Volume represents storage available to workloads in the Kubernetes cluster.

### Persistent Volume Claim

A PVC is a request for persistent storage made by an application.

### Dynamic Provisioning

GKE can automatically provision persistent storage when a PVC requests storage through the appropriate StorageClass.

### StorageClass

The `standard-rwo` StorageClass provides GKE-compatible dynamic provisioning for the PVC.

### ConfigMap

ConfigMap stores non-sensitive configuration separately from the application container.

### Secret

Secret stores sensitive information such as database credentials.

### Deployment

Deployment manages the PostgreSQL application Pod and ensures that the desired workload remains available.

### Service

The ClusterIP Service provides a stable internal endpoint for accessing PostgreSQL.

### Persistent Data

Persistent storage allows database data to survive PostgreSQL Pod deletion and recreation.

---

# Project Details

| Detail | Information |
|---|---|
| Name | Snehal Shinde |
| Project | Project 3.3 |
| Assignment | Kubernetes Deployment with PV, ConfigMap, Secrets, and Services |
| Docker Hub Image | `postgres:17` |
