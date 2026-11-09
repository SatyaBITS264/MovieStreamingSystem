# Movie Streaming System - Deployment Guide

## 1. Overview

This guide describes how to build, deploy, and verify the Movie Streaming System on a local Kubernetes cluster using Minikube.

The application contains two FastAPI microservices:

- Movie Catalog Service
- Watch History Service

Kubernetes manages the deployments, services, health checks, and persistent storage.

## 2. Prerequisites

The following tools are required:

- Windows 11
- Docker Desktop
- Minikube
- kubectl
- PowerShell

Verify the installed tools:

```powershell
docker --version
minikube version
kubectl version --client


3. Start Minikube
Start the local Kubernetes cluster using Docker as the driver:
minikube start --driver=docker


Verify the cluster:
minikube status
kubectl get nodes
kubectl get pods -A


The Minikube node should show a Ready status.
4. Build Docker Images
Build the Movie Catalog image inside the Minikube Docker environment:
minikube image build -t movie-catalog-service:v1 C:\SEZG583\movie-catalog-service


Build the Watch History image:
minikube image build -t watch-history-service:v1 C:\SEZG583\watch-history-service


Verify that the images are available:
minikube image ls


The Kubernetes deployments use imagePullPolicy: Never, so the images must already exist inside Minikube.
5. Kubernetes Configuration
The Kubernetes manifests are stored in:

deployment/kubernetes/

The directory contains:
ManifestPurpose
catalog-pvc.yamlPersistent storage for Movie Catalog
history-pvc.yamlPersistent storage for Watch History
movie-catalog.yamlMovie Catalog Deployment and Service
watch-history.yamlWatch History Deployment and Service

Persistent Volume Claims
Two PersistentVolumeClaims are used:
- movie-catalog-pvc
- watch-history-pvc
Each claim requests 1 GiB of storage using the standard storage class and ReadWriteOnce access mode.

The databases are stored at:
- Movie Catalog: /data/movies.db
- Watch History: /data/history.db
Kubernetes Deployments
Movie Catalog:
- Deployment: movie-catalog
- Container port: 8001
- Image: movie-catalog-service:v1
- Health endpoint: /health
Watch History:
- Deployment: watch-history
- Container port: 8002
- Image: watch-history-service:v1
- Health endpoint: /health
- Catalog URL: http://movie-catalog-service:8001
Kubernetes Services
ServiceService PortNodePort
movie-catalog-service800130081
watch-history-service800230082

6. Deploy the Application
Run these commands from the project root directory:
kubectl apply -f .\deployment\kubernetes\catalog-pvc.yaml
kubectl apply -f .\deployment\kubernetes\history-pvc.yaml
kubectl apply -f .\deployment\kubernetes\movie-catalog.yaml
kubectl apply -f .\deployment\kubernetes\watch-history.yaml


These commands apply the persistent storage claims and the two microservice configurations.
7. Verify the Deployment
Check the deployments:
kubectl get deployments


Check the pods:
kubectl get pods -o wide


Check the services:
kubectl get services


Check persistent volume claims:
kubectl get pvc


Check deployment details:
kubectl describe deployment movie-catalog
kubectl describe deployment watch-history


Both application pods should reach Running status and show Ready containers.

Both PVCs should reach Bound status.
8. Access the Application
NodePort access
Retrieve the Minikube IP:
minikube ip


The services are exposed through these NodePorts:
- Movie Catalog: http://<MINIKUBE-IP>:30081
- Watch History: http://<MINIKUBE-IP>:30082
Swagger UI using port forwarding
Open a separate PowerShell window for each service.

Movie Catalog:
kubectl port-forward service/movie-catalog-service 8004:8001


Open:

http://127.0.0.1:8004/docs

Watch History:
kubectl port-forward service/watch-history-service 8003:8002


Open:

http://127.0.0.1:8003/docs

Keep the port-forward PowerShell windows running while using Swagger UI.
9. Kubernetes Dashboard
Start the Kubernetes Dashboard:
minikube dashboard


The dashboard provides a graphical view of Kubernetes resources, including:
- Deployments
- Pods
- Services
- PersistentVolumeClaims
- Cluster resources
Capture screenshots of the running application resources for project evidence.
10. Persistent Storage Verification
The application databases are stored on persistent volumes mounted at /data.

To inspect the claims:
kubectl get pvc


To inspect the volume mounts:
kubectl describe pod <POD-NAME>


To verify persistence, create a record, replace the relevant application pod, wait for the replacement pod to become Ready, and retrieve the record again.

The record should remain available because the database is stored on the persistent volume rather than only inside the container filesystem.
11. Useful Troubleshooting Commands
View application logs:
kubectl logs deployment/movie-catalog
kubectl logs deployment/watch-history


Check pod details:
kubectl describe pods


Check recent cluster events:
kubectl get events --sort-by=.metadata.creationTimestamp


Check service endpoints:
kubectl get endpoints


12. Architecture Summary
The Movie Catalog Service provides movie information.

The Watch History Service stores user watch records and validates each movie ID by calling Movie Catalog through the Kubernetes service name.

Both services use separate SQLite databases mounted on persistent volumes.

Kubernetes provides service discovery, deployment management, health monitoring, and persistent storage for the application.
