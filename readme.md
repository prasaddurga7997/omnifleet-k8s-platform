<div align="center">

# 🚢 OmniFleet: Enterprise Multi-Tier Kubernetes Platform

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.30+-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![KinD](https://img.shields.io/badge/Cluster-KinD-blue?style=for-the-badge&logo=docker&logoColor=white)](https://kind.sigs.k8s.io/)
[![CKA Aligned](https://img.shields.io/badge/Curriculum-CKA_Exam_Prep-orange?style=for-the-badge&logo=linuxfoundation&logoColor=white)](https://www.cncf.io/certification/cka/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)

<p align="center">
  A production-grade, declarative Kubernetes platform simulating an enterprise microservices architecture with strict workload isolation, deterministic startup sequencing, and full-cluster observability.
</p>

</div>

---

## 📌 Project Overview

**OmniFleet** is designed to demonstrate key production Kubernetes administration patterns aligned with the Certified Kubernetes Administrator (CKA) curriculum:

* **Strict Namespace Isolation:** Production workloads are separated into `omni-prod`, keeping system agents isolated in `kube-system`.
* **Dedicated Data Tier Isolation:** The database tier (`redis-db`) is isolated on `cka-practise-worker2` using a dedicated node taint (`workload=database:NoSchedule`), tolerations, and hard node affinity (`tier=db`).
* **Hardware-Aware Scheduling:** Compute-intensive backend microservices are pinned to specific worker capacity (`size=Large`) via node affinity.
* **Service Startup Sequencing:** Backend replicas use an `initContainer` running `busybox` and `nc` to probe `redis-db-svc:6379` before launching application containers, preventing startup race conditions.
* **Decoupled Dynamic Configuration:** Configuration profiles are decoupled using two ConfigMap strategies: bulk environment injection (`envFrom`) and granular key mapping (`valueFrom`).
* **Edge & Internal Networking:** Exposes internal microservices via stable `ClusterIP` services and routes ingress traffic through an external `NodePort` service.
* **Full-Cluster Telemetry DaemonSet:** Collects node-level health metrics across all nodes—including the tainted database worker and the control plane—leveraging the Downward API.
* **Stateful Storage Layer:** Implement `PersistentVolume` (PV) and `PersistentVolumeClaim` (PVC) with `hostPath` binding on `worker2` for Redis data persistence.
---

## 🏗️ Architecture Diagram

```mermaid
graph TD
    subgraph Host ["Local Environment"]
        Client[HTTP Client / Browser]
        PF[kubectl port-forward :30080 -> 80]
        Client -->|http://localhost:30080| PF
    end

    subgraph Cluster ["Kubernetes Cluster (KinD: cka-practise)"]
        subgraph KubeSystem ["Namespace: kube-system"]
            DS1[cluster-logger Pod<br/>control-plane]
            DS2[cluster-logger Pod<br/>worker]
            DS3[cluster-logger Pod<br/>worker2]
        end

        subgraph OmniProd ["Namespace: omni-prod"]
            subgraph NodeWorker ["Node: cka-practise-worker (size=Large)"]
                F_SVC[Service: omni-frontend-svc<br/>Type: NodePort :30080]
                F_POD1[omni-frontend-1]
                F_POD2[omni-frontend-2]
                
                B_SVC[Service: omni-backend-svc<br/>Type: ClusterIP :80]
                B_POD1[omni-backend-1<br/>Init: wait-for-db]
                B_POD2[omni-backend-2<br/>Init: wait-for-db]
                B_POD3[omni-backend-3<br/>Init: wait-for-db]
                
                CM1[(ConfigMap: backend-config)]
                CM2[(ConfigMap: db-routing)]
            end

            subgraph NodeWorker2 ["Node: cka-practise-worker2 (workload=database:NoSchedule)"]
                DB_SVC[Service: redis-db-svc<br/>Type: ClusterIP :6379]
                DB_POD[Pod: redis-db<br/>Redis Alpine]
                DB_POD --> PVC[(PVC: redis-pvc<br/>500Mi RWO)]
                PVC --> PV[(PV: redis-pv<br/>hostPath /mnt/data/redis)]
            end
        end
    end

    PF --> F_SVC
    F_SVC --> F_POD1 & F_POD2
    F_POD1 & F_POD2 -.->|Internal HTTP Calls| B_SVC
    B_SVC --> B_POD1 & B_POD2 & B_POD3
    
    CM1 -.->|envFrom| B_POD1 & B_POD2 & B_POD3
    CM2 -.->|valueFrom DB_HOST| B_POD1 & B_POD2 & B_POD3
    
    B_POD1 & B_POD2 & B_POD3 -.->|InitContainer TCP Check :6379| DB_SVC
    DB_SVC --> DB_POD