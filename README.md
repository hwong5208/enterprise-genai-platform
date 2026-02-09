# Enterprise GenAI Platform - Architecture & Design Strategy

## 🚀 Mission: Prototype to Product
**Objective:** Transform an existing local GenAI prototype into a robust, scalable, and secure cloud-native platform for the animation industry.

This document outlines the architectural strategy to "bridge GenAI with production pipelines," ensuring high availability, strict security for IP, and cost-effective scaling for GPU workloads.

---

## 🏗️ Architecture Overview

The system is designed as a cloud-native solution on AWS, leveraging **EKS** for orchestration and **Spot Instances** for cost-optimization of GPU workloads.

```mermaid
graph TD
    %% --- Styles ---
    classDef client fill:#E3F2FD,stroke:#0D47A1,stroke-width:2px,color:#000
    classDef aws fill:#FFF3E0,stroke:#E65100,stroke-width:2px,stroke-dasharray: 5 5,color:#000
    classDef control fill:#FFFDE7,stroke:#F57F17,stroke-width:2px,color:#000
    classDef compute fill:#E8F5E9,stroke:#1B5E20,stroke-width:2px,color:#000
    classDef storage fill:#F3E5F5,stroke:#4A148C,stroke-width:2px,color:#000
    classDef monitor fill:#FFEBEE,stroke:#C62828,stroke-width:2px,color:#000

    %% --- Nodes ---
    Client(["Artist Workstation<br/>(Maya / Blender Plugin)"]):::client

    subgraph AWS_Cloud ["AWS Cloud Environment"]
        style AWS_Cloud fill:#FAFAFA,stroke:#616161,stroke-width:2px,color:#000
        
        %% Entry & Auth
        Gateway["API Gateway<br/>(Throttling & Security)"]:::aws
        Auth["Cognito / Auth0<br/>(Identity Provider)"]:::aws
        
        %% Orchestration Layer
        subgraph EKS_Cluster ["Amazon EKS Cluster (Control Plane)"]
            style EKS_Cluster fill:#E0F7FA,stroke:#006064,stroke-width:2px,color:#000
            
            subgraph Services ["Core Services (Fargate)"]
                API_Svc["API Service<br/>(FastAPI / Python)"]:::control
                Karpenter["Karpenter<br/>(Node Autoscaler)"]:::control
            end
            
            subgraph Observability ["Observability Stack"]
                Prometheus["Prometheus<br/>(Metrics)"]:::monitor
                Grafana["Grafana<br/>(Dashboards)"]:::monitor
            end
            
            Queue["Amazon SQS<br/>(Job Queue)"]:::aws
        end

        %% Compute Layer
        subgraph Compute_Plane ["Compute Plane (GPU Spot Instances)"]
            style Compute_Plane fill:#FFFFFF,stroke:#2E7D32,stroke-width:2px,stroke-dasharray: 5 5,color:#000
            Worker1["ComfyUI Worker 1<br/>(g4dn.xlarge)"]:::compute
            WorkerN["...Scale to N..."]:::compute
        end

        %% Data Layer
        subgraph Data_Layer ["Data & Storage"]
            style Data_Layer fill:#FFFFFF,stroke:#7B1FA2,stroke-width:2px,stroke-dasharray: 5 5,color:#000
            EFS[("Amazon EFS<br/>Shared Models")]:::storage
            S3[("Amazon S3<br/>Generated Assets")]:::storage
            QLDB[("Amazon QLDB<br/>Immutable Ledger")]:::storage
            RDS[("Amazon RDS<br/>User Metadata")]:::storage
        end
    end

    %% --- Connections ---
    Client ==>|REST API| Gateway
    Gateway ==> API_Svc
    API_Svc -.->|1. Verify Token| Auth
    API_Svc -->|2. Push Job| Queue
    
    Karpenter -.->|3. Watch Queue Depth| Queue
    Karpenter ==>|4. Provision Nodes| Compute_Plane
    
    Compute_Plane -->|5. Poll Job| Queue
    Compute_Plane ==>|6. Load Models (Read-Only)| EFS
    Compute_Plane -->|7. Save Assets (Write-Once)| S3
    Compute_Plane -->|8. Audit Log (Chain-of-Title)| QLDB
    
    %% Monitoring
    Compute_Plane -.->|GPU Metrics| Prometheus
    API_Svc -.->|Request Metrics| Prometheus
    Prometheus ==>|Visualize| Grafana
```

---

## 🔑 Key Technical Decisions

### 1. Compute & Scalability (Karpenter + Spot)
*   **The Challenge:** GenAI workloads are bursty and GPU/VRAM intensive. Static provisioning is too expensive; standard autoscalers are too slow.
*   **The Solution:** 
    *   **Karpenter:** Directly provisions the "right-sized" node based on pending pod requirements (e.g., requesting a `g5.2xlarge` specifically for a heavy SDXL job).
    *   **Spot Instances:** We use Spot instances for workers to reduce compute costs by up to 90%.
    *   **Defensive Strategy:** If Spot capacity is unavailable, Karpenter falls back to On-Demand instances to ensure SLAs are met.

### 2. Auditability & Chain-of-Title (QLDB)
*   **The Challenge:** Enterprise clients require cryptographic proof of asset provenance for copyright compliance.
*   **The Solution:** **Amazon QLDB (Quantum Ledger Database)**.
    *   Every generated image transaction is recorded in an immutable, cryptographically verifiable ledger.
    *   Records include: *Prompt used, Model hash, User ID, Timestamp, and Input Image hash*.
    *   This provides a "Chain-of-Title" that cannot be altered, satisfying legal requirements for professional production.

### 3. Storage Strategy (Tiered Access)
*   **Models (Read-Heavy):** stored on **Amazon EFS**. Mounted as "Read-Only" across all workers. This prevents the "thundering herd" bottleneck of downloading 5GB models from S3 every time a new node spins up.
*   **Assets (Write-Once, Read-Many):** Generated images are pushed immediately to **Amazon S3** with strict lifecycle policies.
*   **Isolation:** S3 paths are prefixed with Tenant IDs (`s3://bucket/{tenant_id}/...`) to ensure strict data segregation.

---

## 🛠️ Migration Strategy: From Local to Cloud

### Phase 1: Containerization (The "Golden Image")
*   Convert the local Python/ComfyUI environment into a production-ready **Docker image**.
*   **Optimization:** Strip unnecessary development tools; use multi-stage builds to keep image size small for faster pull times.
*   **Dependencies:** Pin all Python versions and CUDA libraries to ensure reproducibility.

### 2. Infrastructure as Code (IaC)
*   Define the entire AWS environment (EKS, VPC, RDS, S3) using **Terraform** or **AWS CDK**.
*   **Goal:** This ensures that "Staging" and "Production" environments are identical, reducing "it works on my machine" issues.

### 3. Job Queue Implementation
*   Decouple the API from the Workers using **Amazon SQS**.
*   **Why?** If a worker crashes (Spot interruption or OOM), the message remains in the queue (via visibility timeout) and is picked up by the next available worker, ensuring zero dropped jobs.

---

## 🛡️ Security & Multi-Tenancy
*   **Identity:** Integration with **Auth0 / Amazon Cognito** for OIDC-compliant authentication.
*   **Network Isolation:** Workers run in private subnets with no public internet access.
*   **Tenancy:** Strict logical isolation at the database (Row-Level Security) and Storage (S3 Prefix) levels.
