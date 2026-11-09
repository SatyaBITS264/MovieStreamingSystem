# Movie Streaming System - Architecture

## 1. Architecture Overview

The Movie Streaming System is a microservices-based application implemented using Python FastAPI and deployed on a local Kubernetes cluster using Minikube.

The application consists of two independently deployable services:

1. Movie Catalog Service
2. Watch History Service

Each service maintains its own SQLite database and persistent storage through Kubernetes PersistentVolumeClaims.

## 2. Architecture Diagram

```mermaid
flowchart TB
    Client["Client / API Consumer"]

    subgraph K8S["Minikube Kubernetes Cluster"]
        CatalogService["Movie Catalog Service - NodePort 30081"]
        HistoryService["Watch History Service - NodePort 30082"]

        subgraph CatalogPod["Movie Catalog Pod"]
            CatalogAPI["FastAPI - Port 8001"]
            CatalogDB[("SQLite - /data/movies.db")]
        end

        subgraph HistoryPod["Watch History Pod"]
            HistoryAPI["FastAPI - Port 8002"]
            HistoryDB[("SQLite - /data/history.db")]
        end

        CatalogPVC[("Movie Catalog PVC - 1Gi")]
        HistoryPVC[("Watch History PVC - 1Gi")]
    end

    Client --> CatalogService
    Client --> HistoryService
    CatalogService --> CatalogAPI
    HistoryService --> HistoryAPI
    HistoryAPI -->|"HTTP movie validation"| CatalogService
    CatalogAPI --> CatalogDB
    HistoryAPI --> HistoryDB
    CatalogDB --- CatalogPVC
    HistoryDB --- HistoryPVC


3. Technology Stack
LayerTechnology
Programming LanguagePython
API FrameworkFastAPI
DatabaseSQLite
ContainerizationDocker
Container OrchestrationKubernetes
Local Kubernetes EnvironmentMinikube
Source ControlGitHub
API DocumentationSwagger UI / OpenAPI
Persistent StorageKubernetes PVC

4. Service Responsibilities
Movie Catalog Service
- Manages movie catalogue information.
- Provides movie creation and retrieval APIs.
- Stores movie information in SQLite.
- Uses persistent storage mounted at /data.
- Runs on container port 8001.
Watch History Service
- Records user movie-watching history.
- Retrieves watch history by user ID.
- Validates movie IDs through the Movie Catalog Service.
- Stores watch history in SQLite.
- Uses persistent storage mounted at /data.
- Runs on container port 8002.
5. Service Communication
The Watch History Service communicates with the Movie Catalog Service through the Kubernetes internal DNS name:

http://movie-catalog-service:8001

Before recording a movie in watch history, the Watch History Service validates that the movie exists in the Movie Catalog.
6. Persistent Storage
Each microservice uses a separate Kubernetes PersistentVolumeClaim.
ServicePVCMount PathDatabase
Movie Catalogmovie-catalog-pvc/datamovies.db
Watch Historywatch-history-pvc/datahistory.db

Both PVCs request 1Gi of storage and use ReadWriteOnce access mode.
7. Deployment and Access
ServiceContainer PortNodePort
Movie Catalog800130081
Watch History800230082

Both services include readiness and liveness probes using their /health endpoints.

The deployments use locally available Minikube images with imagePullPolicy: Never.
