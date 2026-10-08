# Project 3.2 — Resource Management and Horizontal Pod Autoscaling in Kubernetes

## 1. Project Overview

The objective of this project is to understand how Kubernetes manages application resources, maintains the desired number of application Pods, and provides automatic scalability using **Deployments, ReplicaSets, and Horizontal Pod Autoscaling (HPA)**.

A **ReplicaSet** ensures that the specified number of Pod replicas are running at any given time. A **Deployment** manages ReplicaSets and provides a controlled way to deploy, scale, update, and roll back applications.

In this project, a containerized application is deployed using a Kubernetes Deployment with defined **CPU and memory resource requests and limits**. A **Horizontal Pod Autoscaler (HPA)** is then configured to monitor CPU utilization and automatically adjust the number of application Pods based on workload.

Application load is simulated using a temporary load-generator Pod, allowing the scaling behavior of the HPA to be observed in real time.

By completing this project, learners will understand how Kubernetes controllers manage application lifecycle, how resource requests and limits influence workload scheduling, and how HPA provides automatic horizontal scaling based on resource utilization.

---

## 2. Architecture

```text
                         Kubernetes Cluster
                                │
                                ▼
                         Deployment
                         hpa-web-app
                                │
                    ┌───────────┴───────────┐
                    │                       │
                    ▼                       ▼
                 Pod 1                   Pod 2
                    │                       │
                    └───────────┬───────────┘
                                │
                                ▼
                         Kubernetes Service
                           hpa-web-app
                                │
                                ▼
                          Nginx Application

                         Metrics Server
                                │
                                ▼
                         CPU Utilization
                                │
                                ▼
                    Horizontal Pod Autoscaler
                           hpa-web-app
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
                Scale Up                 Scale Down
                    │                       │
                    └───────────┬───────────┘
                                ▼
                         Application Pods
                         Min: 2 / Max: 10

                    Temporary Load Generator
                           BusyBox Pod
                                │
                                ▼
                    Continuous HTTP Requests
                                │
                                ▼
                         hpa-web-app Service
```

---

## 3. Repository Structure

The Project 3.2 Kubernetes manifests are located under the `k8s` directory:

```text
K8s-projects/
│
└── k8s/
    ├── hpa-deployment.yaml
    ├── hpa2.yaml
    └── hpa-service.yaml
```

### Manifest Overview

| File | Purpose |
|---|---|
| `hpa-deployment.yaml` | Deploys the application and defines resource requests and limits |
| `hpa-service.yaml` | Provides a stable Service endpoint for the application |
| `hpa2.yaml` | Configures the Horizontal Pod Autoscaler |

---

## 4. Dependencies

The following components are required to implement and test this project:

| Dependency | Purpose |
|---|---|
| Kubernetes Cluster | Provides the environment for deploying and managing the application |
| `kubectl` | Command-line tool used to interact with the Kubernetes cluster |
| Docker / Container Runtime | Required for building and running container images |
| Docker Hub | Stores and provides access to the application container image |
| Metrics Server | Provides CPU and memory metrics required by HPA |
| Nginx Container Image | Application workload used for deployment |
| BusyBox Image | Used to generate continuous HTTP requests for load testing |
| Google Cloud Shell | Optional environment for executing Kubernetes commands when using GKE |

### Required Kubernetes Components

The following Kubernetes resources are used in this project:

- Deployment
- ReplicaSet
- Pods
- Service
- Horizontal Pod Autoscaler
- Metrics Server

> **Note:** A Kubernetes cluster can be created using platforms such as **Google Kubernetes Engine (GKE)**, Minikube, Kind, or another Kubernetes environment. The commands in this README use standard `kubectl` commands and can be adapted to the selected cluster.

---

# 5. Execution Steps

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

Navigate to the Kubernetes configuration directory:

```bash
cd k8s
```

Verify the Project 3.2 files:

```bash
ls
```

You should find:

```text
hpa-deployment.yaml
hpa2.yaml
hpa-service.yaml
```

---

## Step 2 — Deploy the Application

The application is deployed using a Kubernetes **Deployment**.

The Deployment manifest is:

```text
hpa-deployment.yaml
```

The Deployment is configured with:

- Deployment name: `hpa-web-app`
- Initial replicas: `2`
- Container name: `hpa-web-app`
- Container image: `snehalshinde11/kubernetes-web-app:2.0`
- Container port: `80`
- Image pull policy: `Always`

### Apply the Deployment

```bash
kubectl apply -f hpa-deployment.yaml
```

### Verify the Deployment

```bash
kubectl get deployment hpa-web-app
```

### Verify Application Pods

```bash
kubectl get pods -l app=hpa-web-app
```

Initially, two Pods should be created because the Deployment specifies:

```text
replicas: 2
```

---

## Step 3 — Define Resource Requests and Limits

Kubernetes uses resource requests and limits to manage CPU and memory allocation for containers.

The application Deployment contains the required resource configuration.

### Resource Configuration

| Resource | Request | Limit |
|---|---:|---:|
| CPU | `100m` | `500m` |
| Memory | `64Mi` | `128Mi` |

### Resource Requests

A resource request represents the amount of CPU or memory that Kubernetes considers necessary for scheduling the Pod.

For this application:

```text
CPU Request    = 100m
Memory Request = 64Mi
```

Kubernetes uses these requested resources when deciding where the Pod can be scheduled.

### Resource Limits

A resource limit defines the maximum amount of CPU or memory that the container is allowed to consume.

For this application:

```text
CPU Limit    = 500m
Memory Limit = 128Mi
```

The resource limits help prevent the application from consuming unlimited cluster resources.

---

## Step 4 — Verify Resource Allocation

After deploying the application, verify that the Pods are running with the configured resource settings.

### List Application Pods

```bash
kubectl get pods -l app=hpa-web-app
```

### Display Pod Names

```bash
kubectl get pods -l app=hpa-web-app -o name
```

### Describe a Pod

Replace `<POD_NAME>` with the name of one of the running Pods:

```bash
kubectl describe pod <POD_NAME>
```

In the output, locate the **Requests** and **Limits** section.

You should see values similar to:

```text
Requests:
  cpu:     100m
  memory:  64Mi

Limits:
  cpu:     500m
  memory:  128Mi
```

This confirms that the Pod is running with the configured resource requirements.

### Why Resource Allocation Matters

Resource requests help Kubernetes determine whether a worker node has enough available resources to schedule the Pod.

```text
Pod Request
    │
    ▼
Kubernetes Scheduler
    │
    ▼
Check available node resources
    │
    ▼
Select suitable worker node
    │
    ▼
Schedule Pod
```

---

## Step 5 — Create the Kubernetes Service

The load generator needs a stable endpoint through which it can send requests to the application.

The Service configuration is stored in:

```text
hpa-service.yaml
```

The Service is named:

```text
hpa-web-app
```

It selects the application Pods using the label:

```text
app: hpa-web-app
```

### Apply the Service

```bash
kubectl apply -f hpa-service.yaml
```

### Verify the Service

```bash
kubectl get service hpa-web-app
```

or:

```bash
kubectl get svc hpa-web-app
```

The Service provides a stable DNS name:

```text
hpa-web-app
```

The load generator can therefore send requests to:

```text
http://hpa-web-app
```

---

## Step 6 — Verify Metrics Server

Before configuring the Horizontal Pod Autoscaler, verify that Kubernetes resource metrics are available.

Run:

```bash
kubectl top pods
```

You can also check node metrics:

```bash
kubectl top nodes
```

If CPU and memory values are displayed, the Metrics Server is providing the required metrics.

Example:

```text
NAME                         CPU(cores)   MEMORY(bytes)
hpa-web-app-xxxxx            2m           8Mi
hpa-web-app-yyyyy            2m           8Mi
```

The exact values will vary depending on the workload.

Metrics Server is required because the HPA uses resource utilization metrics to make scaling decisions.

---

## Step 7 — Configure Horizontal Pod Autoscaler

The Horizontal Pod Autoscaler configuration is stored in:

```text
hpa2.yaml
```

The HPA uses the Kubernetes `autoscaling/v2` API.

The HPA is configured with:

- HPA name: `hpa-web-app`
- Target Deployment: `hpa-web-app`
- Minimum replicas: `2`
- Maximum replicas: `10`
- Resource metric: CPU
- Target average CPU utilization: `50%`

### Apply the HPA

```bash
kubectl apply -f hpa2.yaml
```

### Verify the HPA

```bash
kubectl get hpa
```

Or:

```bash
kubectl get hpa hpa-web-app
```

The HPA should show information similar to:

```text
NAME          REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS
hpa-web-app   Deployment/hpa-web-app   5%/50%    2         10        2
```

The exact CPU percentage depends on the current workload.

### Describe the HPA

```bash
kubectl describe hpa hpa-web-app
```

---

## Step 8 — Understand HPA CPU Calculation

The HPA target is configured as:

```text
averageUtilization: 50
```

The HPA calculates CPU utilization relative to the CPU request of the Pods.

For this application:

```text
CPU Request = 100m
```

Therefore:

```text
50% of 100m = 50m
```

Conceptually, if the average CPU usage approaches or exceeds the configured target, the HPA can increase the number of Pods.

The configured scaling boundaries are:

```text
Minimum Pods = 2
Maximum Pods = 10
```

Therefore, the HPA will maintain the number of application Pods within this range.

---

## Step 9 — Generate Application Load

A normal HTTP request against Nginx may not generate enough CPU usage to trigger the HPA.

Therefore, a temporary **BusyBox load-generator Pod** is used to continuously send requests to the application Service.

### Create the Load Generator

Run:

```bash
kubectl run load-generator \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://hpa-web-app; done"
```

This creates a temporary Pod named:

```text
load-generator
```

The Pod continuously executes HTTP requests against:

```text
http://hpa-web-app
```

### Verify the Load Generator

```bash
kubectl get pod load-generator
```

You should see the Pod in the `Running` state.

### Check Load Generator Logs

```bash
kubectl logs load-generator
```

Depending on the BusyBox command and output handling, the logs may contain little or no visible output because the request output is redirected.

The important point is that the Pod continuously generates HTTP traffic toward the Service.

---

## Step 10 — Monitor HPA Scaling Behavior

Open **three terminals** in Google Cloud Shell to observe the scaling process.

### Terminal 1 — Monitor HPA

Run:

```bash
kubectl get hpa hpa-web-app -w
```

Initially, you may see something similar to:

```text
NAME          REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS
hpa-web-app   Deployment/hpa-web-app   5%/50%    2         10        2
```

After CPU usage increases, the HPA may show a higher utilization and increase the replica count.

For example:

```text
hpa-web-app   Deployment/hpa-web-app   75%/50%   2   10   4
```

The exact values and number of replicas depend on the cluster and workload.

### Terminal 2 — Monitor Pods

Run:

```bash
kubectl get pods -l app=hpa-web-app -w
```

When the HPA scales up, additional Pods will appear.

You may initially see:

```text
hpa-web-app-xxxxx   1/1   Running
hpa-web-app-yyyyy   1/1   Running
```

During scaling, additional Pods may temporarily appear as:

```text
hpa-web-app-zzzzz   0/1   Pending
```

After scheduling and startup:

```text
hpa-web-app-xxxxx   1/1   Running
hpa-web-app-yyyyy   1/1   Running
hpa-web-app-zzzzz   1/1   Running
```

### Terminal 3 — Monitor Resource Usage

Run:

```bash
kubectl top pods
```

You can periodically execute the command to observe CPU and memory consumption.

You can also monitor the nodes:

```bash
kubectl top nodes
```

This helps correlate:

```text
Application Load
      │
      ▼
CPU Usage Increases
      │
      ▼
Metrics Server
      │
      ▼
HPA Detects Higher Utilization
      │
      ▼
Deployment Scales Pods
```

---

## Step 11 — Observe Scale Down Behavior

After observing the scale-up behavior, stop the load generator.

Delete the temporary load-generator Pod:

```bash
kubectl delete pod load-generator
```

Verify that it has been removed:

```bash
kubectl get pods
```

Continue watching the HPA:

```bash
kubectl get hpa hpa-web-app -w
```

And monitor the application Pods:

```bash
kubectl get pods -l app=hpa-web-app -w
```

As CPU utilization decreases, the HPA can gradually reduce the number of Pods.

The application will not scale below the configured minimum:

```text
minReplicas: 2
```

Therefore, after the workload decreases, the Deployment should eventually stabilize around the configured minimum replica count.

---

## Step 12 — Verify the Final Application State

After the load test is complete, verify all Kubernetes resources.

### Verify Deployment

```bash
kubectl get deployment hpa-web-app
```

### Verify Pods

```bash
kubectl get pods -l app=hpa-web-app
```

### Verify Service

```bash
kubectl get service hpa-web-app
```

### Verify HPA

```bash
kubectl get hpa hpa-web-app
```

### Verify Resource Usage

```bash
kubectl top pods
```

### Verify Nodes

```bash
kubectl top nodes
```

### Describe HPA

```bash
kubectl describe hpa hpa-web-app
```

---

# 6. Useful Kubernetes Commands

### Apply a Manifest

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

### List Application Pods

```bash
kubectl get pods -l app=hpa-web-app
```

### Describe a Pod

```bash
kubectl describe pod <pod-name>
```

### List Services

```bash
kubectl get svc
```

### List HPA Resources

```bash
kubectl get hpa
```

### Describe HPA

```bash
kubectl describe hpa hpa-web-app
```

### Monitor Pod Metrics

```bash
kubectl top pods
```

### Monitor Node Metrics

```bash
kubectl top nodes
```

### Watch HPA

```bash
kubectl get hpa hpa-web-app -w
```

### Watch Pods

```bash
kubectl get pods -l app=hpa-web-app -w
```

### Create Load Generator

```bash
kubectl run load-generator \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://hpa-web-app; done"
```

### Delete Load Generator

```bash
kubectl delete pod load-generator
```

---

# 7. Key Kubernetes Concepts Demonstrated

### Resource Requests

Resource requests define the amount of CPU and memory that Kubernetes uses when scheduling Pods.

For this application:

```text
CPU    = 100m
Memory = 64Mi
```

### Resource Limits

Resource limits define the maximum CPU and memory that the container can consume.

For this application:

```text
CPU    = 500m
Memory = 128Mi
```

### Horizontal Pod Autoscaler

HPA automatically adjusts the number of application Pods based on resource utilization.

### Metrics Server

Metrics Server provides CPU and memory utilization metrics used by commands such as:

```bash
kubectl top pods
kubectl top nodes
```

and by the Horizontal Pod Autoscaler.

### Kubernetes Service

The Service provides a stable endpoint for accessing the application Pods.

In this project:

```text
Service Name = hpa-web-app
```

### Load Generator

A temporary BusyBox Pod is used to continuously generate HTTP requests and create application workload.

### Minimum and Maximum Replicas

The HPA configuration defines:

```text
Minimum = 2 Pods
Maximum = 10 Pods
```

This prevents the HPA from scaling below or above the configured boundaries.

---

# 8. Scaling Flow

```text
                 User / Load Generator
                         │
                         ▼
                  hpa-web-app
                     Service
                         │
                         ▼
                  Application Pods
                         │
                         ▼
                    CPU Usage
                         │
                         ▼
                  Metrics Server
                         │
                         ▼
              Horizontal Pod Autoscaler
                         │
             ┌───────────┴───────────┐
             │                       │
        High CPU Load           Low CPU Load
             │                       │
             ▼                       ▼
         Scale Up                 Scale Down
             │                       │
             └───────────┬───────────┘
                         ▼
                  2 to 10 Pods
```

---

# 9. Project Outcome

This project demonstrates how Kubernetes can manage application resources and automatically scale workloads based on CPU utilization.

The following capabilities were implemented:

- Kubernetes Deployment
- ReplicaSet-based Pod management
- Containerized Nginx application
- CPU resource requests
- Memory resource requests
- CPU resource limits
- Memory resource limits
- Kubernetes Service
- Metrics Server verification
- Horizontal Pod Autoscaler
- Minimum and maximum replica configuration
- CPU-based autoscaling
- Load generation using BusyBox
- Real-time HPA monitoring
- Pod scaling observation
- Scale-down behavior after load removal

The final setup demonstrates an application that can dynamically adjust its number of Pods based on workload while maintaining defined resource boundaries.

---

# Project Details

| Detail | Information |
|---|---|
| Name | Snehal Shinde |
| Project | Project 3.2 |
| Assignment | Resource Management and Horizontal Pod Autoscaling in Kubernetes |
| Docker Hub Repository | `snehalshinde11/kubernetes-web-app` |
