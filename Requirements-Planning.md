## Requirements planning

This baseline design avoids "magic numbers" by tying every resource value to a specific technical requirement for on-prem Proxmox environments.

### Core Assumptions
*   **Networking:** On-prem Proxmox setup; no cloud-native Load Balancer (LB).
*   **Tooling:** All CI/CD and Observability tools (Jenkins, ArgoCD, Quay, Prometheus, Grafana, etc.) run **inside** Kubernetes.
*   **Databases:** MariaDB Galera and MongoDB run **outside** Kubernetes on dedicated VMs for maximum durability.

---

## 1. Final VM Plan: Production
The production environment is built for High Availability (HA) and performance.

### 1.1 Production VM Table
| S. No. | Server Type | Purpose | vCPU | RAM (GB) | Boot (GB) | Data (GB) | Qty | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1** | HAProxy LB | External LB + TLS | 2 | 4 | 80 | - | 2 | Active/Active or A/P with keepalived VIP. |
| **2** | K8s Control Plane | etcd + API Server HA | 4 | 8 | 80 | - | 3 | Dedicated nodes for cluster stability. |
| **3** | K8s Worker Node | Platform & App Workloads | 8 | 32 | 160 | Shared | 4 | Runs Jenkins, Quay, Prometheus, etc. |
| **4** | MariaDB Galera | Multi-master DB HA | 8 | 16 | 80 | 350 | 3 | 3-node cluster for quorum. |
| **5** | MongoDB Replica | NoSQL HA | 4 | 32 | 80 | 1024 | 3 | Primary + 2 Secondaries for read scaling. |

### 1.2 The Technical Justification
*   **HAProxy (2x):** Essential for on-prem ingress where cloud LBs don't exist. Two nodes prevent a single point of failure (SPOF) for all incoming traffic.
*   **K8s Control Plane (3x):** The 3-node requirement is non-negotiable for **etcd quorum**. 4 vCPU/8 GB ensures the API server isn't choked by observability metrics or CI/CD requests.
*   **K8s Workers (4x):** Sized at 8 vCPU/32 GB to support heavy-duty pods (like Jenkins agents and Elasticsearch) while maintaining a manageable blast radius.
*   **MariaDB Galera (3x):** Requires an odd number of nodes (minimum 3) to prevent **split-brain** scenarios. High vCPU count handles the overhead of synchronous replication.
*   **MongoDB (3x):** High RAM (32 GB) is critical to keep the "working set" (indexes and hot data) in memory, preventing disk I/O bottlenecks.



---

## 2. Final VM Plan: Staging
Staging mirrors production architecture but reduces the resource footprint to save costs.

### 2.1 Staging VM Table
| S. No. | Server Type | Purpose | vCPU | RAM (GB) | Boot (GB) | Data (GB) | Qty | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1** | HAProxy LB | Edge LB | 2 | 4 | 80 | - | 1–2 | 1 for cost; 2 for mirroring prod HA. |
| **2** | K8s Control Plane | Control Plane | 4 | 8 | 80 | - | 3 | Keep topology identical to Production. |
| **3** | K8s Worker Node | Tools & Testing | 4 | 16 | 160 | Shared | 3 | Reduced size; lower traffic load. |
| **4** | MariaDB (Single) | Database | 4 | 16 | 80 | 200 | 1 | Single node is acceptable for testing. |
| **5** | MongoDB (Single) | NoSQL | 4 | 16 | 80 | 300 | 1 | Trade-off: Lower HA for lower cost. |

### 2.2 Why This Matters
*   **Consistency:** Keeping the Control Plane at 3 nodes ensures that cluster upgrades and management scripts behave exactly like they do in Production.
*   **Trade-offs:** We use single-node databases here because staging is for **functional testing**, not HA benchmarking. If failover testing is required, it should be done on a temporary, dedicated cluster.

---

## 3. Final VM Plan: Development
Focused on the absolute minimum resources required to facilitate a developer workflow.

### 3.1 Development VM Table
| S. No. | Server Type | vCPU | RAM (GB) | Boot (GB) | Data (GB) | Qty | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1** | HAProxy LB | 2 | 4 | 80 | - | 1 | Single LB. |
| **2** | K8s Control Plane | 2 | 4 | 80 | - | 1 | Non-HA is acceptable for Dev. |
| **3** | K8s Worker Node | 4 | 8 | 160 | Shared | 2 | Test scheduling and DaemonSets. |
| **4** | MariaDB (Single) | 2 | 8 | 80 | 100 | 1 | Small datasets only. |
| **5** | MongoDB (Single) | 2 | 8 | 80 | 200 | 1 | Minimal memory footprint. |

---

## 4. Shared Design Assumptions
> **Note:** These assumptions should be presented to infrastructure stakeholders to clarify the operational scope.

*   **Internal Tooling:** All "Platform Tools" (ArgoCD, Prometheus, etc.) are containerized and reside in K8s namespaces (e.g., `platform-tools`).
*   **Storage:** A shared backend (Ceph, NFS, or iSCSI) is utilized via K8s **StorageClasses** for worker nodes.
*   **Database Security:** External DBs are placed in private subnets with access restricted to the K8s node IP range.

---

## 5. Executive Summary: "The Elevator Pitch"
> "We sized the control plane according to Kubernetes best practices, using three dedicated nodes to maintain etcd quorum and isolate cluster management from application noise. Worker nodes are medium-sized (8 vCPU/32 GB) to support heavy-duty tools like Jenkins and Quay while ensuring we can drain nodes for maintenance without overcommitting resources. For our databases, we utilize a three-node external cluster pattern for both MariaDB and MongoDB to guarantee high availability and automatic failover, which is critical for stateful data durability outside the ephemeral container environment."
