# Proxmox-3-Node-HA-Cluster & Ceph: Ausfalltest-Bericht

**Sprachversionen:** [English](README.md) | [Česky](README_CZ.md)

**Umgebung:** Homelab. 3 Proxmox-VE-Knoten (pve1, pve2, pve3), Ceph-Speicher, Workload: Apache-Webserver + 3-Node-MariaDB-Galera-Cluster.
**Autor:** Radim Kulaviak (isol.cz)

## Topologie
- **pve1 (192.168.0.200):** OSD.0, HA-verwaltete VMs (web1, db1)
- **pve2 (192.168.0.201):** OSD.1, OSD.2, VM db2
- **pve3 (192.168.0.202):** OSD.3, VM db3
- **Speicher:** Ceph-Pool `vm-pool`, Replikation 3

## Test A: Anwendungsausfall (MariaDB gestoppt)
- **Szenario:** `systemctl stop mariadb` auf `db2`.
- **Ergebnis:** Die Galera-Clustergröße fiel von 3 auf 2 (`wsrep_cluster_size`), die übrigen Knoten blieben synchron, Proxmox war nicht betroffen.
- **Wiederherstellung:** Nach dem Start von MariaDB lief ein inkrementeller State Transfer (IST), der Cluster war wieder mit 3 synchronen Knoten verfügbar.

## Test B: Kaskadierende Knotenausfälle
- **Szenario:** `pve1` länger als 20 Minuten heruntergefahren; während der Wiederherstellung fiel `pve3` aus (Fencing).
- **Ergebnis:** Nach der Rückkehr der Knoten befanden sich Proxmox HA, Ceph und Galera wieder in einem gesunden Zustand.
- **Kernaussage:** *Ein gesunder Ceph-Cluster mit Quorum macht eine VM nicht automatisch hochverfügbar.* Die VM muss dem HA-Manager hinzugefügt werden (`ha-manager add vm:<ID> --state started`), sonst bleibt sie nach einem Hostausfall im Zustand `stopped`.

## Befehle zur Zustandsprüfung
```bash
ceph -s
pvecm status
ha-manager status
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

## Konzeptionelle Parallelen zu AWS (Lernnotizen zu SAA-C03)
Nur Analogien, die Dienste sind nicht gleichwertig.

| Proxmox / Ceph | Ähnliche Idee in AWS |
|---|---|
| Ceph-MON-Quorum (toleriert 1 von 3 Ausfällen) | Multi-AZ-Failover (z. B. RDS) |
| Ceph-OSD-Replikation und Rebalancing | Replizierter Speicher über AZs |
| Proxmox-HA-Manager (CRM/LRM) | Auto-Scaling-Gruppe mit Health Checks |

*Umfang: Laborergebnisse, keine Performance-Benchmarks.*
