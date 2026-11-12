# Proxmox 3-Node HA Cluster & Ceph: Failure Test Report

**Language versions:** [Česky](README_CZ.md) | [Deutsch](README_DE.md)

**Environment:** home lab. 3x Proxmox VE nodes (pve1, pve2, pve3), Ceph storage, workload: Apache web server + 3-node MariaDB Galera cluster.
**Author:** Radim Kulaviak (isol.cz)

## Topology
- **pve1 (192.168.0.200):** OSD.0, HA-managed VMs (web1, db1)
- **pve2 (192.168.0.201):** OSD.1, OSD.2, VM db2
- **pve3 (192.168.0.202):** OSD.3, VM db3
- **Storage:** Ceph pool `vm-pool`, replication size 3

## Test A: application failure (MariaDB stop)
- **Scenario:** `systemctl stop mariadb` on `db2`.
- **Result:** Galera cluster size dropped from 3 to 2 (`wsrep_cluster_size`), the remaining nodes stayed synced, Proxmox was not affected.
- **Recovery:** starting MariaDB triggered an incremental state transfer (IST) and the cluster returned to 3 synced nodes.

## Test B: cascading node failures
- **Scenario:** `pve1` shut down for more than 20 minutes; `pve3` then failed (fenced) during recovery.
- **Result:** after the nodes came back, Proxmox HA, Ceph and Galera returned to a healthy state.
- **Key takeaway:** *healthy Ceph and quorum do not make a VM highly available by themselves.* A VM must be added to the HA manager (`ha-manager add vm:<ID> --state started`), otherwise it stays `stopped` after a host failure.

## Commands used to check state
```bash
ceph -s
pvecm status
ha-manager status
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

## Conceptual parallels to AWS (study notes for SAA-C03)
Analogies only, the services are not equivalent.

| Proxmox / Ceph | Similar AWS idea |
|---|---|
| Ceph MON quorum (tolerates 1 of 3 failures) | Multi-AZ failover (e.g. RDS) |
| Ceph OSD replication and rebalancing | Replicated storage across AZs |
| Proxmox HA manager (CRM/LRM) | Auto Scaling group with health checks |

*Scope: lab results, no performance benchmarks.*
