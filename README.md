# BERDL System Documentation

This directory contains documentation for the BERDL purpose-built data lakehouse system.

All source code repositories are located in the [BERDataLakehouse GitHub Organization](https://github.com/BERDataLakehouse).

## System Architecture

BERDL utilizes a microservices architecture to provide a secure, scalable, and interactive data analysis environment. The core components include dynamic notebook spawning, secure credential management, and an MCP (Model Context Protocol) server for AI-assisted data operations.

```mermaid
graph TD
    %% Use subgraphs to organize hierarchically
    
    subgraph Users ["User Layer"]
        User([User])
    end

    subgraph Entry ["Platform Entry"]
        JH[BERDL JupyterHub]
        NB[Spark Notebook]
    end

    subgraph Core ["Core Services"]
        direction TB
        MMS[MinIO Manager Service]
        SCM[Spark Cluster Manager]
        MCP[Datalake MCP Server]
    end

    subgraph Compute ["Compute Layer"]
        DYNC[Dynamic Spark Cluster]
        SM[Shared Static Cluster]
    end

    subgraph Data ["Data & Metadata"]
        S3[MinIO Storage]
        HM[Hive Metastore]
        Disk[(Storage)]
    end

    %% Interactions
    
    %% User Entry Flow
    User -->|"Browser (Login & UI)"| JH
    User -->|Direct API| MCP
    
    %% JupyterHub Internal Flow
    JH -->|Proxies UI| NB
    JH -->|Init Policy| MMS
    JH -->|Trigger Create| SCM
    
    %% Service Logic
    SCM -->|Spawns| DYNC
    NB -->|Uses| DYNC
    
    %% Notebook Interactions
    NB -->|Auth| MMS
    NB -->|Query| MCP
    
    %% MCP Logic
    MCP -->|Direct/Fallback| SM
    MCP -->|Via Hub| DYNC
    
    %% Data Access
    NB -->|S3| S3
    NB -->|Meta| HM
    DYNC -->|Process| S3
    SM -->|Process| S3
    S3 -.-> Disk

    %% Styling
    classDef service fill:#f9f,stroke:#333,stroke-width:2px;
    classDef storage fill:#ff9,stroke:#333,stroke-width:2px;
    classDef compute fill:#cce6ff,stroke:#333,stroke-width:2px;
    
    class JH,NB,MMS,SCM,MCP service;
    class S3,HM,Disk storage;
    class DYNC,SM compute;
```

## Container Dependency Architecture

The following diagram illustrates the build hierarchy and base image dependencies for the BERDL services.

```mermaid
graph TD
    %% Base Images
    JQ[quay.io/jupyter/pyspark-notebook]
    PUB_JH[jupyterhub/jupyterhub]
    PY313[python:3.13-slim]
    PY311[python:3.11-slim]

    %% Internal Base
    subgraph Foundation
        id1(spark_notebook_base)
    end

    %% Services
    subgraph Services
        NB[spark_notebook]
        MCP[datalake-mcp-server]
        MMS[minio_manager_service]
        SCM[spark_cluster_manager]
        JH[BERDL_JupyterHub]
    end

    %% Dynamic Compute
    subgraph DynamicCompute ["Dynamic Compute"]
        DYNC["Dynamic Spark Cluster (kube_spark_manager_image)"]
    end

    %% Relations
    JQ -->|FROM| id1
    
    id1 -->|FROM| NB
    id1 -->|FROM| MCP
    
    NB -->|FROM| DYNC
    
    PUB_JH -->|FROM| JH
    PY313 -->|FROM| MMS
    PY311 -->|FROM| SCM
    
    %% Styling
    classDef external fill:#eee,stroke:#333,stroke-dasharray: 5 5;
    classDef internal fill:#cce6ff,stroke:#333,stroke-width:2px;
    classDef service fill:#f9f,stroke:#333,stroke-width:2px;
    classDef compute fill:#ffcc00,stroke:#333,stroke-width:2px;

    class JQ,PUB_JH,PY313,PY311 external;
    class id1 internal;
    class NB,MCP,MMS,SCM,JH service;
    class DYNC compute;
```



## Python Dependency Architecture

The following diagram illustrates the internal Python package dependencies.

```mermaid
graph TD
    %% Clients
    SMC[cdm-spark-manager-client]
    MMSC[minio-manager-service-client]
    MCPC[datalake-mcp-server-client]
    
    %% Base Package
    subgraph Base ["spark_notebook_base"]
        PNB[berdl-notebook-python-base]
    end
    
    %% Service Implementations
    subgraph NotebookUtils ["spark_notebook"]
        NU[berdl_notebook_utils]
    end
    
    subgraph MCPServer ["datalake-mcp-server"]
        MCP[datalake-mcp-server]
    end
    
    subgraph JupyterHub ["BERDL_JupyterHub"]
        JH[berdl-jupyterhub]
    end

    %% Dependencies
    PNB -->|Dep| SMC
    PNB -->|Dep| MMSC
    PNB -->|Dep| MCPC
    
    NU -->|Dep| PNB
    MCP -->|Dep| NU
    
    JH -->|Dep| SMC
    
    %% Styling
    classDef client fill:#ffedea,stroke:#cc0000,stroke-width:1px;
    classDef pkg fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    
    class SMC,MMSC,MCPC client;
    class PNB,NU,MCP,JH pkg;
```

## Core Components

| Service | Description | Documentation | Repository |
|---------|-------------|---------------|------------|
| **JupyterHub** | Manages user sessions and spawns individual notebook servers. | [BERDL JupyterHub](./services/berdl-jupyterhub.md) | [Repo](https://github.com/BERDataLakehouse/BERDL_JupyterHub) |
| **Spark Notebook** | User's personal workspace with Spark pre-configured. | [Spark Notebook](./services/spark_notebook.md) | [Repo](https://github.com/BERDataLakehouse/spark_notebook) |
| **Spark Notebook Base** | Foundational Docker image with PySpark and common dependencies. | [Spark Notebook Base](./services/spark_notebook_base.md) | [Repo](https://github.com/BERDataLakehouse/spark_notebook_base) |
| **Datalake MCP Server** | FastAPI Data API with MCP layer for AI interactions and direct queries. | [Datalake MCP Service](./services/datalake-mcp-service.md) | [Repo](https://github.com/BERDataLakehouse/datalake-mcp-server) |
| **MinIO Manager Service** | Handles dynamic credentials and IAM policies for secure data access. | [MinIO Manager Service](./services/minio-manager-service.md) | [Repo](https://github.com/BERDataLakehouse/minio_manager_service) |
| **Spark Cluster Manager** | API for managing dynamic, personal Spark clusters on K8s (Primary for Users). | [Spark Cluster Manager](./services/spark-cluster-manager.md) | [Repo](https://github.com/BERDataLakehouse/spark_cluster_manager) |
| **Hive Metastore** | Stores metadata for Delta Lake tables. | [Hive Metastore](./services/hive-metastore.md) | [Repo](https://github.com/BERDataLakehouse/hive_metastore) |
| **Spark Cluster** | Spark master/worker image for static and dynamic clusters. | [Spark Cluster](./services/spark-cluster.md) | [Repo](https://github.com/BERDataLakehouse/kube_spark_manager_image) |
