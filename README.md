# Proxmox 3-Node HA Cluster & Ceph Test Report

**Language versions:** [Česky](README_CZ.md) | [Deutsch](README_DE.md)

**Environment (lab):** 3× Proxmox VE nodes (pve1, pve2, pve3), Ceph storage, workload: Apache web server + 3-node MariaDB Galera cluster.
**Author:** Radim Kulaviak ([isol.cz](https://www.isol.cz))

## Topology
- **pve1:** OSD.0, HA-managed VMs (web1, db1)
- **pve2:** OSD.1, OSD.2, VM db2
- **pve3:** OSD.3, VM db3
- **Storage:** Ceph pool `vm-pool`, replication size = 3

## Test scenarios and findings

### Test A: Application failure (MariaDB stop)
- **Scenario:** `systemctl stop mariadb` on `db2`.
- **Result:** `wsrep_cluster_size` dropped from 3 to 2; the remaining nodes stayed synced; the Proxmox layer was unaffected.
- **Recovery:** Starting MariaDB triggered an incremental state transfer (IST); the cluster was back to 3 synced nodes within seconds.

### Test B: Node failure and an unplanned second failure
- **Scenario:** `pve1` shut down for more than 20 minutes; during recovery `pve3` was unexpectedly fenced.
- **Result:** Proxmox HA, Ceph and Galera recovered without manual data repair. No data loss was observed in this lab.
- **Takeaway:** *Ceph/quorum redundancy is not the same as VM high availability.* VMs must be added to the HA manager explicitly (`ha-manager add vm:<ID> --state started`), otherwise they stay `stopped` after a host failure.

### Limits of this test
Three nodes, lab hardware, one failure sequence. It demonstrates the behaviour and my procedure, not a performance or capacity benchmark.

## Conceptual mapping to AWS (study notes, no production AWS experience)
| Proxmox / Ceph | AWS (conceptually comparable) |
|---|---|
| Ceph MON quorum | Multi-AZ failover decisions (e.g. RDS Multi-AZ) |
| Ceph replication / rebalance | EBS snapshots, cross-AZ replication |
| Proxmox HA manager | Auto Scaling group + health checks |
