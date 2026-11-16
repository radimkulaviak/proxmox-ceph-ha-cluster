# Proxmox 3-Knoten-HA-Cluster & Ceph-Testbericht

**Sprachversionen:** [English](README.md) | [Česky](README_CZ.md)

**Umgebung (Labor):** 3× Proxmox VE (pve1, pve2, pve3), Ceph-Speicher, Workload: Apache + 3-Knoten MariaDB-Galera-Cluster.
**Autor:** Radim Kulaviak ([isol.cz](https://www.isol.cz))

## Topologie
- **pve1:** OSD.0, HA-verwaltete VMs (web1, db1)
- **pve2:** OSD.1, OSD.2, VM db2
- **pve3:** OSD.3, VM db3
- **Speicher:** Ceph-Pool `vm-pool`, Replikation = 3

## Testszenarien und Erkenntnisse

### Test A: Anwendungsausfall (MariaDB-Stopp)
- **Szenario:** `systemctl stop mariadb` auf `db2`.
- **Ergebnis:** `wsrep_cluster_size` sank von 3 auf 2; die übrigen Knoten blieben synchron; die Proxmox-Ebene blieb unberührt.
- **Wiederherstellung:** Der MariaDB-Start löste einen inkrementellen Zustandstransfer (IST) aus; der Cluster war innerhalb von Sekunden wieder bei 3 synchronen Knoten.

### Test B: Knotenausfall und ungeplanter zweiter Ausfall
- **Szenario:** `pve1` länger als 20 Minuten heruntergefahren; während der Wiederherstellung wurde `pve3` unerwartet gefenct.
- **Ergebnis:** Proxmox HA, Ceph und Galera erholten sich ohne manuelle Datenreparatur. In diesem Labor wurde kein Datenverlust beobachtet.
- **Erkenntnis:** *Ceph-/Quorum-Redundanz ist nicht gleich VM-Hochverfügbarkeit.* VMs müssen explizit zum HA-Manager hinzugefügt werden (`ha-manager add vm:<ID> --state started`), sonst bleiben sie nach einem Host-Ausfall `stopped`.

### Grenzen des Tests
Drei Knoten, Laborhardware, eine Ausfallsequenz. Der Test zeigt das Verhalten und mein Vorgehen, keinen Leistungs- oder Kapazitäts-Benchmark.
