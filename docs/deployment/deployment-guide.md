# Movie Streaming System - Deployment Guide

## 1. Overview

This guide describes how to build, deploy, and verify the Movie Streaming System on a local Kubernetes cluster using Minikube.

The application contains two FastAPI microservices:

- **Movie Catalog Service** – manages movie information.
- **Watch History Service** – records user watch history and validates movie IDs through the Movie Catalog Service.

Kubernetes manages deployments, service discovery, health checks, autoscaling, and persistent storage.

## 2. Prerequisites

The following tools are required:

- Windows 11
- Docker Desktop
- Minikube
- kubectl
- PowerShell
- Git
- Postman (for API testing)

Verify the installed tools:

```powershell
docker --version
minikube version
kubectl version --client
git --version
```

## 3. Start Minikube

Start the local Kubernetes cluster using Docker as the driver:

```powershell
minikube start --driver=docker
```

Verify the cluster:

```powershell
minikube status
kubectl get nodes
kubectl get pods -A
```

The Minikube node should show a Ready status.

## 4. Build Docker Images

Build the Movie Catalog image inside the Minikube Docker environment:

```powershell
minikube image build -t movie-catalog-service:v1 C:\SEZG583\movie-catalog-service
```

Build the Watch History image:

```powershell
minikube image build -t watch-history-service:v1 C:\SEZG583\watch-history-service
```

Verify that the images are available:

```powershell
minikube image ls
```

The Kubernetes deployments use `imagePullPolicy: Never`, so the images must already exist inside Minikube.

## 5. Kubernetes Configuration

The Kubernetes manifests are stored in:

```text
deployment/kubernetes/
```

| Manifest | Purpose |
|---|---|
| catalog-pvc.yaml | Persistent storage for Movie Catalog |
| history-pvc.yaml | Persistent storage for Watch History |
| movie-catalog.yaml | Movie Catalog Deployment and Service |
| watch-history.yaml | Watch History Deployment and Service |

### 5.1 Persistent Volume Claims

Two PersistentVolumeClaims are used:

- `movie-catalog-pvc`
- `watch-history-pvc`

Each claim requests 1 GiB of storage using the standard storage class and ReadWriteOnce access mode.

The databases are stored at:

- Movie Catalog: `/data/movies.db`
- Watch History: `/data/history.db`

### 5.2 Kubernetes Deployments

**Movie Catalog**

- Deployment: `movie-catalog`
- Container port: `8001`
- Image: `movie-catalog-service:v1`
- Health endpoint: `/health`

**Watch History**

- Deployment: `watch-history`
- Container port: `8002`
- Image: `watch-history-service:v1`
- Health endpoint: `/health`
- Catalog URL: `http://movie-catalog-service:8001`

Both deployments define CPU and memory resource requests and limits to support resource management and Horizontal Pod Autoscaling.

### 5.3 Kubernetes Services

| Service | Service Port | NodePort |
|---|---:|---:|
| movie-catalog-service | 8001 | 30081 |
| watch-history-service | 8002 | 30082 |

## 6. Deploy the Application

Run these commands from the project root directory:

```powershell
kubectl apply -f .\deployment\kubernetes\catalog-pvc.yaml
kubectl apply -f .\deployment\kubernetes\history-pvc.yaml
kubectl apply -f .\deployment\kubernetes\movie-catalog.yaml
kubectl apply -f .\deployment\kubernetes\watch-history.yaml
```

These commands apply the persistent storage claims and the two microservice configurations.

## 7. Verify the Deployment

Check the deployments:

```powershell
kubectl get deployments
```

Check the pods:

```powershell
kubectl get pods -o wide
```

Check the services:

```powershell
kubectl get services
```

Check persistent volume claims:

```powershell
kubectl get pvc
```

Check deployment details:

```powershell
kubectl describe deployment movie-catalog
kubectl describe deployment watch-history
```

Both application deployments should have available replicas, and both PVCs should reach Bound status.

## 8. Access the Application

### 8.1 NodePort Access

Retrieve the Minikube IP:

```powershell
minikube ip
```

The services are exposed through these NodePorts:

- Movie Catalog: `http://<MINIKUBE-IP>:30081`
- Watch History: `http://<MINIKUBE-IP>:30082`

### 8.2 Swagger UI Using Port Forwarding

Open a separate PowerShell window for each service.

**Movie Catalog:**

```powershell
kubectl port-forward service/movie-catalog-service 8004:8001
```

Open:

`http://127.0.0.1:8004/docs`

**Watch History:**

```powershell
kubectl port-forward service/watch-history-service 8003:8002
```

Open:

`http://127.0.0.1:8003/docs`

Keep the port-forward PowerShell windows running while using Swagger UI.

## 9. Kubernetes Dashboard

Start the Kubernetes Dashboard:

```powershell
minikube dashboard
```

The dashboard provides a graphical view of Kubernetes resources, including:

- Deployments
- Pods
- Services
- PersistentVolumeClaims
- Cluster resources

The dashboard can be used to inspect the application resources and capture project evidence.

## 10. Persistent Storage Verification

The application databases are stored on persistent volumes mounted at `/data`.

Inspect the claims:

```powershell
kubectl get pvc
```

Inspect the volume mounts:

```powershell
kubectl describe pod <POD-NAME>
```

To verify persistence:

1. Create a movie or watch-history record.
2. Record the returned data.
3. Delete only the relevant application Pod.
4. Wait for its Deployment to create a replacement Pod.
5. Retrieve the record again through the API.

The record should remain available because the database is stored on the persistent volume rather than only inside the container filesystem.

This verifies data persistence across application Pod replacement in the Minikube environment.

## 11. Horizontal Pod Autoscaling (HPA)

Horizontal Pod Autoscaling automatically adjusts the number of application Pods based on observed resource utilization.

### 11.1 Enable Metrics Server

Enable the Minikube Metrics Server add-on:

```powershell
minikube addons enable metrics-server
```

Verify that resource metrics are available:

```powershell
kubectl top nodes
kubectl top pods
```

### 11.2 Configure Resource Requests and Limits

Both application deployments use the following container resource configuration:

| Resource | Request | Limit |
|---|---:|---:|
| CPU | 100m | 500m |
| Memory | 128Mi | 512Mi |

CPU requests provide the baseline used by HPA to calculate CPU utilization.

### 11.3 Create HPA Resources

Create an HPA for Movie Catalog:

```powershell
kubectl autoscale deployment movie-catalog --cpu-percent=50 --min=1 --max=5
```

Create an HPA for Watch History:

```powershell
kubectl autoscale deployment watch-history --cpu=50% --min=1 --max=5
```

The configuration uses:

- Target CPU utilization: 50%
- Minimum replicas: 1
- Maximum replicas: 5

### 11.4 Verify HPA

Check the autoscaler status:

```powershell
kubectl get hpa
```

Monitor Pod counts:

```powershell
kubectl get pods
```

Monitor CPU utilization:

```powershell
kubectl top pods
```

During load testing, increased CPU utilization can cause HPA to increase the number of application replicas, subject to the configured maximum and available cluster resources.

After the load is removed, HPA can reduce replicas according to its scaling behavior and stabilization settings.

## 12. CI/CD Pipeline

Both microservices have independent GitHub Actions CI/CD workflows.

### 12.1 Movie Catalog CI/CD

Repository:

`https://github.com/SatyaBITS264/movie-catalog-service`

Workflow:

`.github/workflows/ci-cd.yml`

The workflow is triggered by:

- Push to the main branch
- Pull requests targeting the main branch

Pipeline stages:

1. Checkout source code.
2. Set up Python 3.12.
3. Install application and development dependencies.
4. Run automated tests using pytest.
5. Set up Docker Buildx.
6. Build the Docker image.

The Movie Catalog test suite contains 6 automated tests.

### 12.2 Watch History CI/CD

Repository:

`https://github.com/SatyaBITS264/watch-history-service`

Workflow:

`.github/workflows/ci-cd.yml`

The workflow uses the same push and pull-request triggers.

Pipeline stages:

1. Checkout source code.
2. Set up Python 3.12.
3. Install application and development dependencies.
4. Run automated tests using pytest.
5. Set up Docker Buildx.
6. Build the Docker image.

The Watch History test suite contains 7 automated tests, including tests for successful catalog validation, invalid movie IDs, and catalog service connection failures.

### 12.3 CI/CD Verification

The GitHub Actions workflow runs automated tests and verifies that the Docker image can be built.

Both service workflows have completed successfully.

**Pipeline scope:** Docker image builds use `push: false`. The current workflows do not publish images to a container registry or automatically deploy the application to Kubernetes. Kubernetes deployment is performed separately using the deployment manifests.

## 13. API Testing Using Postman

Postman is used to validate the REST APIs exposed by both microservices.

The API tests cover:

- Movie Catalog health check.
- Create a movie.
- Retrieve all movies.
- Retrieve a movie by ID.
- Verify the response for an invalid movie ID.
- Watch History health check.
- Create and retrieve watch-history records.

Expected HTTP response codes include:

| Operation | Expected Status |
|---|---:|
| Health check | 200 |
| Create movie | 201 |
| Retrieve movies | 200 |
| Retrieve movie by ID | 200 |
| Invalid movie ID | 404 |
| Create watch-history record | 201 |
| Retrieve watch history | 200 |

Postman screenshots are maintained in:

`evidence/postman/`

## 14. Evidence and Screenshots

Project evidence is maintained in the `evidence/` directory.

| Directory | Evidence |
|---|---|
| `evidence/kubernetes/` | Kubernetes deployment and resource screenshots |
| `evidence/persistence/` | Data before and after Pod replacement |
| `evidence/postman/` | API testing screenshots |
| `evidence/hpa/` | HPA load and replica scaling screenshots |
| `evidence/ci_cd/` | GitHub Actions automated test evidence |

## 15. Useful Troubleshooting Commands

View application logs:

```powershell
kubectl logs deployment/movie-catalog
kubectl logs deployment/watch-history
```

Check Pod details:

```powershell
kubectl describe pods
```

Check recent cluster events:

```powershell
kubectl get events --sort-by=.metadata.creationTimestamp
```

Check service endpoints:

```powershell
kubectl get endpoints
```

Check HPA status:

```powershell
kubectl describe hpa
```

Check resource metrics:

```powershell
kubectl top nodes
kubectl top pods
```

## 16. Architecture Summary

The Movie Catalog Service provides movie information and supports movie creation and retrieval.

The Watch History Service stores user watch records and validates each movie ID by calling Movie Catalog through the Kubernetes service name.

Both services use separate SQLite databases mounted on persistent volumes.

Kubernetes provides service discovery, deployment management, health monitoring, persistent storage, and horizontal scaling.

GitHub Actions provides automated testing and Docker image build verification for both microservices.
