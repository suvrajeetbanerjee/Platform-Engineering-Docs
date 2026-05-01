# Proxmox Virtual Environment (VE) Report: Architecture, Management, and Production Operations

## 1. The Proxmox Execution Layers: Hardware to User Space
Understanding the vertical execution flow of Proxmox VE is critical for platform engineers, especially when managing performance and isolating workloads for Kubernetes nodes.

* **Hardware Layer:** The physical silicon (CPU, RAM, Storage, NICs). Proxmox operates as a **Type-1 hypervisor** running directly on this bare metal, allowing it immediate control over hardware states and memory.
* **Kernel Space (Ring 0):** This is the privileged core of the system that has unrestricted hardware access. The customized Linux kernel manages the **Kernel-based Virtual Machine (KVM)** module to virtualize CPU and memory. This is where **namespaces** and **cgroups** reside.
    * **Namespaces** provide logical isolation (visibility of networks, processes, etc.).
    * **cgroups (Control Groups)** manage resource limitations, ensuring no single VM or container monopolizes CPU or RAM.
* **System Call Interface (SCI):** The gateway for unprivileged applications to request privileged kernel services.
* **Terminal Subsystem:** A complex subsystem spanning both user and kernel space. It combines a user-space **Terminal Emulator** (rendering UI/keys) and a kernel-space **Pseudo-Terminal (PTY)** driver.
* **User Space (Ring 3):** This is where Proxmox resides. Proxmox is fundamentally a suite of user-space tools and daemons that orchestrate kernel features. Key user-space components include:
    * `pvedaemon`: The REST API orchestrator.
    * `pmxcfs`: The Proxmox Cluster File System.
    * `QEMU`: The user-space process that emulates hardware for KVM virtual machines.

---

## 2. Proxmox Dashboard Components and Concepts
The web-based GUI abstracts the complex Linux backend into a structured, logical hierarchy. Understanding this hierarchy is essential for platform setups.

### Datacenter View (Global Governance)
The Datacenter level acts as the global control plane for the entire cluster. Configurations here replicate across all nodes in real-time.
* **Cluster & HA:** Manages node membership, **Corosync** health (the cluster communication engine), and High Availability (HA) groups and failover rules.
* **Storage:** Defines shared storage backends (like NFS, iSCSI, Ceph) that must be globally accessible to allow for live migrations of VMs between physical nodes.
* **SDN (Software-Defined Networking):** Centralizes complex network fabrics (VLANs, VXLANs, EVPN) to create isolated networks for multi-tenancy.
* **Firewall & Permissions:** Sets cluster-wide security rules, Role-Based Access Control (RBAC), and authentication realms (LDAP, Active Directory).

### Node View (Infrastructure Management)
The Node represents the physical server.
* **System & Network:** Configures physical NICs, Linux bridges, DNS, and NTP synchronization.
* **Disks & Ceph:** Manages local storage arrays, initializes ZFS pools, and oversees hyper-converged Ceph Object Storage Daemons (OSDs) attached to that specific server.

### Guest View (Workload Management)
"Guests" refer to the actual Virtual Machines (KVM) and Linux Containers (LXC).
* **Hardware & Options:** Allows you to define emulated devices (CPUs, RAM, storage controllers, network devices), boot order, and QEMU Guest Agent integrations.

---

## 3. Proxmox Crash Course: VM Provisioning Approach
For a platform engineer provisioning VMs for a Kubernetes cluster, the basic operations follow this flow:

* **How VMs are created:** In the GUI, you use the "Create VM" wizard, assign a unique VM ID, name, and allocate resources. **At scale**, engineers build "Golden Images" using **HashiCorp Packer**, install **cloud-init**, convert the VM to a template, and use **Terraform** (specifically the `bpg/proxmox` provider) to deploy fleets.
* **How Storage is attached:** Storage pools are defined at the Datacenter level. When creating a VM, you attach a virtual disk. 
    > **Best Practice:** Select **SCSI** as the bus and **VirtIO SCSI single** as the controller; enable **Discard** (for TRIM support) and **IO Thread** for maximum performance.
* **How an ISO is mounted:** Download your OS ISO image to a file-level storage pool. In the VM's Hardware tab, create a CD/DVD drive on an **IDE** bus and select the ISO.
* **How installation is done:** Start the VM and open the built-in **noVNC** or **SPICE** console. After installation, always install the `qemu-guest-agent` in the guest OS for graceful shutdowns and IP visibility.
* **How networking is attached:** VMs connect via virtual NICs (vNICs) attached to a Linux Bridge on the host (usually `vmbr0`). Traffic can be isolated via VLAN tags or SDN VNets.
* **How backups are created:** Handled by the `vzdump` utility or the **Proxmox Backup Server (PBS)**. PBS is recommended for production due to deduplication and "dirty bitmaps" for fast incremental backups.

---

## 4. Critical "Gotchas" in Production Environments

* **Corosync Network Latency (The "Hanging Cluster"):** If cluster heartbeat traffic shares a network with heavy storage or backup traffic, latency may spike above 5ms. This causes nodes to lose quorum and "fence" (forcefully reboot). 
    * **Solution:** Always use a dedicated, physically separate network for Corosync.
* **ZFS ARC Memory Starvation:** By default, the ZFS Adaptive Replacement Cache (ARC) can consume up to 50% of host RAM, potentially starving VMs.
    * **Solution:** Manually limit ZFS ARC size via `/etc/modprobe.d/zfs.conf`.
* **Template Cloning & Machine-ID Conflicts:** If you do not truncate the `/etc/machine-id` file in your template, cloned VMs may fight for the same DHCP IP address.
* **Thin Provisioning Expansion (Discard/TRIM):** On thin-provisioned storage (LVM-Thin/ZFS), you must check the **Discard** box and enable **QEMU Guest Agent** so deleted files inside the guest release space on the physical storage.
* **IOMMU and Hardware Passthrough Failures:** Passing through GPUs/NICs requires enabling **VT-d / AMD-Vi** in BIOS and adding `intel_iommu=on` or `amd_iommu=on` to the Linux kernel command line.
* **Kubernetes Storage with CSI:** To automate storage, use the `proxmox-csi-plugin`. This dynamically creates virtual disks for Kubernetes **PersistentVolumeClaims (PVC)**.



---

## Implementation

### 1. Environment Decisions and Cleanup
- Dropped the earlier Linode Kubernetes Engine (LKE) cluster to regain **full control with kubeadm** and align with on‑prem style architecture.
- Decided to treat Linode VMs as **bare‑metal equivalents** for the lab, matching the Host360 Proxmox design conceptually (control planes + workers, HAProxy later).

### 2. Bastion / Admin Host Setup
- Created a dedicated **bastion VM** (`admin-platform`) on Linode:
  - OS: Ubuntu (24.04 from output).
  - Network: Public Internet (no cloud firewall yet; hardening planned later).
- Clarified the role of the bastion:
  - Single **operations entry point**: laptop → bastion → all cluster nodes.
  - All K8s and infra commands will be executed from the bastion, not from Windows terminals.

### 3. Staging Kubernetes Node Layout (VM Provisioning)
- Provisioned **6 Linode VMs** for the staging cluster:
  - Control plane nodes:
    - `stg-cp1` – "xxx.xxx.xxx"
    - `stg-cp2` – "xxx.xxx.xxx"
    - `stg-cp3` – "xxx.xxx.xxx"
  - Worker nodes:
    - `stg-worker1` – "xxx.xxx.xxx"
    - `stg-worker2` – "xxx.xxx.xxx"
    - `stg-worker3` – "xxx.xxx.xxx"
- Chose **Public Internet** networking for all nodes to keep the initial setup simple; VLAN/VPC hardening deferred to a later phase.

### 4. SSH Key-Based Access via Bastion
- On `admin-platform`:
  - Generated an **Ed25519 SSH keypair** (`/root/.ssh/id_ed25519` and `.pub`).
  - Used `ssh-copy-id` to install the public key on all 6 staging nodes.
- Verified passwordless SSH:
  - Confirmed `ssh root@<node-ip>` works **without password** for:
    - `stg-cp1`, `stg-cp2`, `stg-cp3`
    - `stg-worker1`, `stg-worker2`, `stg-worker3`
- Wrote and executed a small helper script (`pblky2nds.sh`) on the bastion to automate `ssh-copy-id` against all node IPs.

### 5. Phase 1 Progress (Cluster Setup Roadmap)
- Completed:
  - **Phase 1 – Step 0: Bastion & SSH setup**
    - Bastion host created and reachable.
    - Key‑based SSH configured from bastion to all future K8s nodes.
    - Set hostnames (`stg-cp1..3`, `stg-worker1..3`).
    - Disabled swap and configure required `sysctl` settings for Kubernetes networking.
- Next steps :
  - **Phase 1 – Step 2: Install containerd + kubeadm/kubelet/kubectl**
    - Prepare all 6 nodes for kubeadm `init`/`join`
