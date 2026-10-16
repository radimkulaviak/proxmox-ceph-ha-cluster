# Proxmox 3-Knoten HA-Cluster & Ceph Testbericht

**Sprachversionen:** [English](README.md) | [Cesky](README_CZ.md)

**Umgebung:** 3x Proxmox VE Knoten (pve1, pve2, pve3), Ceph HA-Speicher, Workload: Apache Webserver + 3-Knoten MariaDB Galera Cluster.
**Autor:** Radim Kulaviak (isol.cz)

## Topologie und Architektur
- **pve1 (192.168.0.200):** OSD.0, hostet HA-verwaltete VMs (web1, db1).
- **pve2 (192.168.0.201):** OSD.1, OSD.2, hostet db2.
- **pve3 (192.168.0.202):** OSD.3, hostet db3.
- **Speicher:** Ceph Pool `vm-pool`, Replikation = 3.

---

## Wichtige Testszenarien und Erkenntnisse

### Test A: Anwendungsausfall (MariaDB Stopp)
- **Szenario:** `systemctl stop mariadb` auf Knoten `db2`.
- **Ergebnis:** Galera-Clustergroesse sank von 3 auf 2 (`wsrep_cluster_size`). Verbleibende Knoten blieben synchronisiert. Die Proxmox-Infrastrukturebene blieb unbeeintraechtigt.
- **Wiederherstellung:** Der Start von MariaDB loeste einen inkrementellen Zustandsstransfer (IST) aus, der innerhalb von Sekunden wieder zu einem synchronisierten 3-Knoten-Zustand fuehrte.

### Test B: Kaskadierender Infrastruktur- und Knotenausfall
- **Szenario:** Vollstaendiges Herunterfahren des Hauptknotens `pve1` (>20 Min) kombiniert mit einem sekundaeren Ausfall von `pve3` waehrend der Wiederherstellung.
- **Ergebnis:** Der Cluster fuehrte erfolgreich eine automatisierte Selbstheilung ueber Proxmox HA-, Ceph- und Galera-Ebenen ohne Datenverlust durch.
- **Kritische Erkenntnis:** *Infrastruktur-HA (Ceph/Quorum) entspricht nicht automatisch VM-HA.* Virtuelle Maschinen muessen explizit zum HA-Manager hinzugefuegt werden (`ha-manager add vm:<ID> --state started`), da sie sonst nach einem Host-Ausfall im Status `stopped` verbleiben.
