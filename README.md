# Proxmox 3-Node HA Cluster & Ceph: Failure Test Report

**Language versions:** [Česky](README_CZ.md) | [Deutsch](README_DE.md)

**Environment:** home lab, 20-21 August 2026. 3x Proxmox VE nodes (pve1, pve2, pve3), Ceph storage, workload: nginx + PHP-FPM web VM and a 3-node MariaDB Galera cluster. Old hardware (Xeon E5410, SAS/SATA HDD).
**Author:** Radim Kulaviak (isol.cz)

## Topology
| Node | IP | Ceph OSD | VMs |
|---|---|---|---|
| pve1 | 192.168.0.200 | osd.0 (137 GiB) | VM 100, web1 (101), db1 (102) |
| pve2 | 192.168.0.201 | osd.1, osd.2 (314 GiB) | db2 (103) |
| pve3 | 192.168.0.202 | osd.3 (233 GiB) | db3 (104) |

- **Storage:** Ceph pool `vm-pool`, replication size 3, 683 GiB raw capacity.
- **HA:** VMs 100, 101, 102 are managed by the Proxmox HA manager. db2 and db3 (103, 104) are not; their availability relies on Galera replication only.

## Results at a glance
| Test | Scenario | Result |
|---|---|---|
| A | `systemctl stop mariadb` on db2 | Galera 3 → 2 nodes, service kept running, back to 3 synced via IST within seconds |
| B | db2 VM network link set down (`link_down=1`) | Ceph and Proxmox HA unaffected, Galera 3 → 2, recovered via IST |
| C | planned reboot of pve3 (CRM master) | new CRM master elected within seconds, physical boot ~4.5 min, Ceph `HEALTH_WARN` without data loss |
| D | pve2 network cable pulled for ~10-15 min | no HA-managed VM on that node, MON quorum 2/3 kept, `HEALTH_OK` within minutes after reconnect |
| E | pve1 shut down for >20 min, then pve3 fenced during recovery | HA-managed VMs restarted on other nodes, no data loss, full `HEALTH_OK` about 48 min after the start |

## Test A: application failure (MariaDB stop)
- **Scenario:** `systemctl stop mariadb` on `db2`, the other two nodes continue.
- **Result:** `wsrep_cluster_size` 3 → 2, remaining nodes `Synced`. A web test page showed db2 failing and db1/db3 answering. Proxmox HA was not involved.
- **Recovery:** `systemctl start mariadb` triggered an incremental state transfer (IST), back to 3 synced nodes within seconds.

## Test B: VM network isolation
- **Scenario:** `qm set 103 --net0 virtio,bridge=vmbr0,link_down=1` on pve2 (equivalent to unplugging the VM's cable, the host and Ceph network stay untouched).
- **Result:** `ceph -s` unchanged, VM 103 stayed `running`, Galera dropped to 2 nodes. The layers are independent: this is an application-level problem only.
- **Recovery:** `link_down=0`, Galera IST, back to 3/3.
- **Side incident:** I forgot to restore `link_down=0`, which later caused `WSREP: failed to open gcomm backend connection` when starting MariaDB. A full `qm stop` / `qm start` fixed it.

## Test C: planned reboot of one node (pve3, CRM master)
- **Result:** the CRM master moved automatically within seconds. Ceph showed `HEALTH_WARN` (degraded redundancy), no rebalance because the outage was shorter than the 10-minute `mon_osd_down_out_interval`. No impact on VMs (VM 100 was not on that node).
- **Takeaway:** recovery time here is limited by the physical boot of old hardware (~4.5 min), not by the Proxmox/Ceph mechanism.

## Test D: unplanned node outage (pve2, cable)
- **Result:** no HA-managed VM ran on pve2, so no failover was needed. Ceph degraded but MON quorum 2/3 held. After reconnecting, `HEALTH_OK` returned automatically within a few minutes.

## Test E: node outage with HA-managed VMs plus a second failure
- **Scenario:** `shutdown -h now` on pve1 (VM 100, web1, db1, all HA-managed), outage over 20 minutes. During the return of pve1 I shut down pve3 by mistake, which caused a second fencing event.
- **Timeline** (approximate, from my notes, not exact log times):

| Time after shutdown | Event |
|---|---|
| ~1 min | MON quorum 2/3, osd.0 down, HA failover: vm:100 → pve2, vm:101 → pve3, vm:102 → pve2 |
| ~10 min | Ceph marks OSDs `out`, redundancy degraded by 33 % |
| ~36 min | pve1 powered on |
| ~38 min | unplanned fence of pve3, vm:101 recovered to pve1 |
| ~42 min | pve3 back online |
| ~48 min | `HEALTH_OK`, 33 PGs `active+clean`, HA stable |

- **Result:** both outages survived without data loss and without manual action on the HA/Ceph layer. db1 rejoined Galera automatically (`wsrep_cluster_size` back to 3).
- **Key takeaway:** *healthy Ceph and quorum do not make a VM highly available by themselves.* VM 104 (db3) was not in the HA manager, stayed `stopped` after pve3 recovered and needed a manual `qm start 104`. Add every critical VM with `ha-manager add vm:<ID> --state started`.

## Lessons for production designs
1. Add **all** critical VMs to the HA manager, not only the main ones.
2. Spread Ceph OSD capacity evenly. Here it was about 20 % / 46 % / 34 % (pve1 / pve2 / pve3), which complicates rebalancing when the heaviest node fails.
3. Recovery time on this old hardware is dominated by boot time (~4.5 min). I expect it to be much lower on modern hardware, but I have not measured that.
4. Test cascading failures (a second node failing during recovery), not only single outages.
5. Check that application services inside VMs are enabled at boot (`systemctl is-enabled`).

## Not tested yet
- A real physical disk failure (pulling a disk, not only stopping the OSD service)
- A node failure under simultaneous load
- A complete Ceph rebalance cycle after the OSDs were marked `out` (only the start was observed)

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
| `mon_osd_down_out_interval` (10 min) | Health check grace period |

*Scope: lab results on old hardware, no performance benchmarks. Times are approximate.*
