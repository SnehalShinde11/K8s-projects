# Project 3.1 — Deploying Scalable Applications using ReplicaSets and Deployments

## Project Overview

The objective of this project is to understand how Kubernetes ensures **high availability and scalability of applications** using **ReplicaSets and Deployments**.

A **ReplicaSet** ensures that a specified number of Pod replicas are running at any given time. **Deployments** build on top of ReplicaSets and allow developers to manage application updates, scaling, and rollbacks.

By completing this project, learners will deploy a scalable application and manage its lifecycle using Kubernetes controllers.

This project additionally demonstrates:

- Building and publishing a Docker image
- Creating and managing a Kubernetes ReplicaSet
- Self-healing using ReplicaSets
- Creating and managing a Kubernetes Deployment
- Scaling applications manually and automatically
- Performing rolling updates and rollbacks
- Exposing the application using a Kubernetes Service
- Configuring Horizontal Pod Autoscaling (HPA)
- Defining CPU and memory resource requests and limits
- Monitoring Pods, Nodes, and application logs

---

## Architecture

```text
                    Docker Hub
                        │
                        │
              kubernetes-web-app:1.0
                        │
                        ▼
                 Kubernetes / GKE
                        │
                        ▼
                 Deployment
            kubernetes-web-app
                        │
                        ▼
                  ReplicaSet
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
            Pod 1     Pod 2     Pod 3
              │         │         │
              └─────────┼─────────┘
                        │
                        ▼
               Kubernetes Service
          kubernetes-web-app-service
                        │
                        ▼
                  Application

          HPA monitors CPU utilization
                        │
                        ▼
             Scale Pods automatically
```

---

## Repository Structure

After cloning the repository, the Project 3.1 related files are organized as follows:

```text
K8s-projects/
│
├── Dockerfile
├── index.html
├── index-v1-backup.html
│
└── k8s/
    ├── replicaset.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── hpa.yaml
```

---

# Execution Steps

## Step 1 — Clone the Repository

Clone the project repository to the local machine or Cloud Shell environment:

```bash
git clone https://github.com/SnehalShinde11/K8s-projects.git
```

Navigate into the repository:

```bash
cd K8s-projects
```

Verify the repository contents:

```bash
ls
```

Navigate to the Kubernetes configuration directory when required:

```bash
cd k8s
```

---

## Step 2 — Prepare the Application Container Image

The application is packaged into a Docker image using Nginx.

The Dockerfile uses the Nginx Alpine image and copies the application `index.html` into the Nginx web root.

### Dockerfile

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Build the Docker Image

From the project root directory:

```bash
docker build -t snehalshinde11/kubernetes-web-app:1.0 .
```

### Push the Image to Docker Hub

Authenticate with Docker Hub if required:

```bash
docker login
```

Push the image:

```bash
docker push snehalshinde11/kubernetes-web-app:1.0
```

The image used by Kubernetes is:

```text
snehalshinde11/kubernetes-web-app:1.0
```

---

## Step 3 — Create a ReplicaSet

A ReplicaSet ensures that the desired number of Pods are running at all times.

The ReplicaSet configuration is stored in:

```text
k8s/replicaset.yaml
```

The ReplicaSet is configured with:

- Name: `kubernetes-web-app-rs`
- Desired replicas: `3`
- Container image: `snehalshinde11/kubernetes-web-app:1.0`
- Container port: `80`

Navigate to the Kubernetes configuration directory:

```bash
cd k8s
```

### Validate the Manifest

```bash
kubectl apply --dry-run=client -f replicaset.yaml
```

### Create the ReplicaSet

```bash
kubectl apply -f replicaset.yaml
```

### Verify ReplicaSet

```bash
kubectl get replicaset
```

or:

```bash
kubectl get rs
```

### Verify Pods

```bash
kubectl get pods
```

Display Pods along with their labels:

```bash
kubectl get pods --show-labels
```

### Describe the ReplicaSet

```bash
kubectl describe rs kubernetes-web-app-rs
```

### Display Pod Details

```bash
kubectl get pods -o wide
```

---

## Step 4 — Verify ReplicaSet Self-Healing

One of the main benefits of a ReplicaSet is **self-healing**.

If a Pod is deleted or fails, the ReplicaSet automatically creates a replacement Pod to maintain the desired replica count.

### Watch Pods

```bash
kubectl get pods -w
```

In another terminal, list the running Pods:

```bash
kubectl get pods
```

Delete one of the running Pods:

```bash
kubectl delete pod <pod-name>
```

The ReplicaSet detects that the number of running Pods has decreased and automatically creates a replacement.

Verify the Pods:

```bash
kubectl get pods
```

The desired state should return to:

```text
3 Pods
```

---

## Step 5 — Create a Deployment

A Deployment provides declarative management of Pods and ReplicaSets.

The Deployment configuration is stored in:

```text
k8s/deployment.yaml
```

The Deployment is configured with:

- Name: `kubernetes-web-app`
- Replicas: `3`
- Image: `snehalshinde11/kubernetes-web-app:1.0`
- Container port: `80`
- Strategy: `RollingUpdate`
- `maxUnavailable: 0`
- `maxSurge: 1`

### Validate the Deployment Manifest

```bash
kubectl apply --dry-run=client -f deployment.yaml
```

### Create the Deployment

```bash
kubectl apply -f deployment.yaml
```

### Verify Deployment

```bash
kubectl get deployments
```

```bash
kubectl get deployment kubernetes-web-app
```

### Verify ReplicaSets

```bash
kubectl get rs
```

### Verify Pods

```bash
kubectl get pods --show-labels
```

### Describe Deployment

```bash
kubectl describe deployment kubernetes-web-app
```

The Deployment creates and manages a ReplicaSet, which in turn manages the application Pods.

---

## Step 6 — Scale the Application

The Deployment can be scaled without modifying the Deployment manifest.

Check the current Deployment:

```bash
kubectl get deployment
```

Check the running Pods:

```bash
kubectl get pods
```

Scale the application from 3 to 5 replicas:

```bash
kubectl scale deployment kubernetes-web-app --replicas=5
```

Watch the Pods:

```bash
kubectl get pods -w
```

Verify the Deployment:

```bash
kubectl get deployment
```

The application should now have:

```text
5 Pods
```

---

## Step 7 — Perform a Rolling Update

A rolling update allows a new application version to be deployed without taking the application completely offline.

### Return to the Application Directory

```bash
cd ..
```

The command should place you in the `K8s-projects` project root.

### Backup the Existing Application

Create a backup of version 1:

```bash
cp index.html index-v1-backup.html
```

Modify `index.html` to create version 2 of the application.

For example, the updated application can display:

```text
Dashboard created for version two Snehal Shinde.
```

### Build Version 2

```bash
docker build -t snehalshinde11/kubernetes-web-app:2.0 .
```

### Push Version 2

```bash
docker push snehalshinde11/kubernetes-web-app:2.0
```

### Watch the Pods

From another terminal:

```bash
kubectl get pods -w
```

### Update the Deployment Image

```bash
kubectl set image deployment/kubernetes-web-app \
kubernetes-web-app=snehalshinde11/kubernetes-web-app:2.0
```

Kubernetes gradually replaces the old Pods with new Pods using the Deployment's `RollingUpdate` strategy.

### Verify ReplicaSets

```bash
kubectl get rs
```

### Verify Pods

```bash
kubectl get pods -o wide
```

### Verify the Image

```bash
kubectl describe deployment kubernetes-web-app | grep "Image:"
```

### Check Rollout History

```bash
kubectl rollout history deployment/kubernetes-web-app
```

### Access the Application on GKE

The application can be tested using Kubernetes port forwarding:

```bash
kubectl port-forward deployment/kubernetes-web-app 8080:80 --address=0.0.0.0
```

Then use:

**Google Cloud Shell Web Preview → Preview on port 8080**

The updated application should display:

```text
Dashboard created for version two Snehal Shinde.
```

---

## Step 8 — Rollback the Deployment

A failed image deployment can be simulated using an image tag that does not exist.

### Simulate a Failed Deployment

```bash
kubectl set image deployment/kubernetes-web-app \
kubernetes-web-app=snehalshinde11/kubernetes-web-app:3.0
```

Since version `3.0` does not exist, the new Pods cannot successfully start.

Check the Pods:

```bash
kubectl get pods
```

Check the ReplicaSets:

```bash
kubectl get rs
```

### Restore the Stable Version

Set the Deployment back to version `1.0`:

```bash
kubectl set image deployment/kubernetes-web-app \
kubernetes-web-app=snehalshinde11/kubernetes-web-app:1.0
```

Check rollout status:

```bash
kubectl rollout status deployment/kubernetes-web-app
```

Verify ReplicaSets:

```bash
kubectl get rs
```

The Deployment should return to the stable application image.

---

## Step 9 — Expose the Application Using a Kubernetes Service

A Kubernetes Service provides a stable network endpoint for accessing Pods managed by the Deployment.

The Service configuration is stored in:

```text
k8s/service.yaml
```

The Service is named:

```text
kubernetes-web-app-service
```

### Create the Service

```bash
vi service.yaml
```

Apply the Service manifest:

```bash
kubectl apply -f service.yaml
```

### Verify the Service

```bash
kubectl get service
```

or:

```bash
kubectl get svc
```

### Verify Service Endpoints

```bash
kubectl get endpoints kubernetes-web-app-service
```

The endpoints should show the IP addresses of the Pods selected by the Service.

### Service Flow

```text
Client
  │
  ▼
kubernetes-web-app-service
  │
  ├── Pod 1
  ├── Pod 2
  ├── Pod 3
  └── Pod 4 / Pod 5
```

The Service provides a stable abstraction even when individual Pods are created, deleted, or replaced.

---

## Step 10 — Enable Auto-Scaling Using Horizontal Pod Autoscaler

The **Horizontal Pod Autoscaler (HPA)** automatically adjusts the number of Pods based on resource utilization.

For CPU-based scaling, Kubernetes calculates CPU utilization against the CPU **request** configured for the container.

### Verify Metrics Server

Before creating the HPA, verify that Kubernetes metrics are available:

```bash
kubectl top pods
```

If CPU and memory values are returned for the Pods, the cluster is ready for CPU-based HPA.

Node metrics can also be checked:

```bash
kubectl top nodes
```

### Create the HPA

The HPA configuration is stored in:

```text
k8s/hpa.yaml
```

Create or edit the manifest:

```bash
vi hpa.yaml
```

Apply the HPA:

```bash
kubectl apply -f hpa.yaml
```

### Verify HPA

```bash
kubectl get hpa
```

Describe the HPA:

```bash
kubectl describe hpa kubernetes-web-app-hpa
```

The HPA monitors the Deployment and can automatically increase or decrease the number of Pods based on the configured CPU utilization target.

### HPA Flow

```text
              Metrics Server
                    │
                    ▼
             CPU Utilization
                    │
                    ▼
       Horizontal Pod Autoscaler
                    │
             ┌──────┴──────┐
             ▼             ▼
       Scale Up         Scale Down
             │             │
             └──────┬──────┘
                    ▼
             Kubernetes Pods
```

---

## Step 11 — Add Resource Requests and Limits

Resource requests and limits control how much CPU and memory a Pod requests and the maximum resources it can consume.

The Deployment includes:

```yaml
resources:
  requests:
    cpu: "50m"
    memory: "32Mi"
  limits:
    cpu: "200m"
    memory: "128Mi"
```

### Resource Configuration

| Resource | Request | Limit |
|---|---:|---:|
| CPU | `50m` | `200m` |
| Memory | `32Mi` | `128Mi` |

### Why Requests Matter for HPA

The CPU utilization used by the HPA is calculated relative to the configured CPU request.

For this application:

```text
CPU Request = 50m
```

Therefore, the HPA uses the `50m` CPU request as the baseline when calculating CPU utilization.

Resource requests also help Kubernetes make scheduling decisions, while limits define the maximum amount of CPU and memory the container can consume.

---

## Step 12 — Monitor Pods Using kubectl and Logs

Kubernetes provides several commands for monitoring the health, resource usage, and logs of application Pods.

### List Pods

```bash
kubectl get pods
```

### Display Detailed Pod Information

```bash
kubectl get pods -o wide
```

This displays additional information such as:

- Pod IP
- Node
- Pod status
- Restart count

### Watch Pods in Real Time

```bash
kubectl get pods -w
```

This is useful while:

- Scaling the Deployment
- Performing rolling updates
- Testing self-healing
- Monitoring HPA activity

### Monitor Pod CPU and Memory

```bash
kubectl top pods
```

### Monitor Node CPU and Memory

```bash
kubectl top nodes
```

### View Pod Logs

First list the Pods:

```bash
kubectl get pods
```

Then view the logs of a specific Pod:

```bash
kubectl logs <pod-name>
```

For continuously streaming logs:

```bash
kubectl logs -f <pod-name>
```

The `-f` option follows the log output in real time.

### Useful Monitoring Commands

```bash
kubectl get pods
kubectl get pods -o wide
kubectl get pods -w
kubectl top pods
kubectl top nodes
kubectl logs <pod-name>
kubectl logs -f <pod-name>
```

---

# Useful Kubernetes Commands

### Deploy Resources

```bash
kubectl apply -f <file>.yaml
```

### Validate a Manifest

```bash
kubectl apply --dry-run=client -f <file>.yaml
```

### List Deployments

```bash
kubectl get deployments
```

### List ReplicaSets

```bash
kubectl get rs
```

### List Pods

```bash
kubectl get pods
```

### List Services

```bash
kubectl get svc
```

### List HPA

```bash
kubectl get hpa
```

### Describe a Resource

```bash
kubectl describe <resource> <name>
```

### Scale a Deployment

```bash
kubectl scale deployment <deployment-name> --replicas=<number>
```

### Update a Container Image

```bash
kubectl set image deployment/<deployment-name> \
<container-name>=<image>:<tag>
```

### Check Rollout Status

```bash
kubectl rollout status deployment/<deployment-name>
```

### Check Rollout History

```bash
kubectl rollout history deployment/<deployment-name>
```

### Port Forward

```bash
kubectl port-forward deployment/<deployment-name> 8080:80 --address=0.0.0.0
```

---

# Key Kubernetes Concepts Demonstrated

### ReplicaSet

Maintains a specified number of running Pods and provides self-healing when Pods fail or are deleted.

### Deployment

Provides declarative application management and controls ReplicaSets, rolling updates, and application versions.

### Rolling Update

Gradually replaces old application Pods with new Pods while maintaining application availability.

### Rollback

Allows the application to return to a previously stable version after an unsuccessful deployment.

### Kubernetes Service

Provides a stable network endpoint for a group of Pods selected using labels.

### Horizontal Pod Autoscaler

Automatically adjusts the number of Pods based on resource utilization such as CPU.

### Resource Requests

Define the amount of CPU and memory required by a container for scheduling and resource calculations.

### Resource Limits

Define the maximum CPU and memory that a container can consume.

### Metrics Server

Provides resource utilization metrics such as CPU and memory that can be consumed by commands such as `kubectl top` and by the HPA.

### Pod Monitoring

`kubectl` provides commands for monitoring Pod status, resource usage, placement, and application logs.

---

# Project Outcome

This project demonstrates an end-to-end Kubernetes deployment workflow for a containerized Nginx web application on GKE.

The following capabilities are implemented:

- Docker image creation
- Docker Hub image publishing
- Kubernetes ReplicaSet deployment
- ReplicaSet self-healing
- Kubernetes Deployment
- Manual application scaling
- Rolling updates
- Deployment rollback
- Kubernetes Service exposure
- Horizontal Pod Autoscaling
- CPU and memory resource requests and limits
- Pod and Node resource monitoring
- Application log monitoring
- GKE application access using port forwarding

The final setup provides a scalable and manageable Kubernetes application with **self-healing, controlled deployments, service discovery, automatic scaling, resource management, and monitoring capabilities**.

---

## Project Details

| Detail | Information |
|---|---|
| Name | Snehal Shinde |
| Project | Project 3.1 |
| Assignment | Deploying Scalable Applications using ReplicaSets and Deployments |
| Docker Hub Repository | `snehalshinde11/kubernetes-web-app` |
