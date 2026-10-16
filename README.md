# Proxmox 3-Node HA Cluster & Ceph Test Report

**Language versions:** [Cesky](README_CZ.md) | [Deutsch](README_DE.md)

**Environment:** 3x Proxmox VE nodes (pve1, pve2, pve3), Ceph HA storage, workload: Apache web server + 3-node MariaDB Galera Cluster.
**Author:** Radim Kulaviak (isol.cz)

## Topology & Architecture
- **pve1 (192.168.0.200):** OSD.0, hosting HA-managed VMs (web1, db1).
- **pve2 (192.168.0.201):** OSD.1, OSD.2, hosting db2.
- **pve3 (192.168.0.202):** OSD.3, hosting db3.
- **Storage:** Ceph pool `vm-pool`, replication size = 3.

---

## Key Test Scenarios & Findings

### Test A: Application Failure (MariaDB Stop)
- **Scenario:** `systemctl stop mariadb` on node `db2`.
- **Result:** Galera cluster size dropped from 3 to 2 (`wsrep_cluster_size`). Remaining nodes stayed fully synced. Proxmox infrastructure layer remained unaffected.
- **Recovery:** Starting MariaDB triggered an incremental state transfer (IST), returning to a 3-node synced state within seconds.

### Test B: Cascading Infrastructure & Node Failures
- **Scenario:** Full shutdown of primary node `pve1` (>20 min) combined with an accidental secondary failure/fencing of `pve3` during recovery.
- **Result:** The cluster successfully executed automated self-healing across Proxmox HA, Ceph, and Galera layers without data loss.
- **Critical Production Takeaway:** *Infrastructure HA (Ceph/Quorum) does not automatically equal VM HA.* Virtual machines must be explicitly added to the HA manager (`ha-manager add vm:<ID> --state started`), otherwise they will remain in a `stopped` state after a host failure and recovery.

---

## AWS SAA-C03 Architectural Parallels
| Proxmox / Ceph Concept | AWS Equivalent |
|---|---|
| Ceph MON Quorum (survives 1/3 failure) | RDS Multi-AZ Failover |
| Ceph OSD Replication & Rebalance | EBS Snapshots & Cross-AZ Replication |
| Proxmox HA Manager (CRM/LRM) | Auto Scaling Group (ASG) & Health Checks |
| `mon_osd_down_out_interval` (10 min) | ASG Health Check Grace Period |
