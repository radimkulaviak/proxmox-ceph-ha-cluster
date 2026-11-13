# Proxmox-3-Node-HA-Cluster & Ceph: Ausfalltest-Bericht

**Sprachversionen:** [English](README.md) | [Česky](README_CZ.md)

**Umgebung:** Homelab, 20.-21. August 2026. 3 Proxmox-VE-Knoten (pve1, pve2, pve3), Ceph-Speicher, Workload: Web-VM mit nginx + PHP-FPM und ein 3-Node-MariaDB-Galera-Cluster. Alte Hardware (Xeon E5410, SAS/SATA-HDD).
**Autor:** Radim Kulaviak (isol.cz)

## Topologie
| Knoten | IP | Ceph-OSD | VMs |
|---|---|---|---|
| pve1 | 192.168.0.200 | osd.0 (137 GiB) | VM 100, web1 (101), db1 (102) |
| pve2 | 192.168.0.201 | osd.1, osd.2 (314 GiB) | db2 (103) |
| pve3 | 192.168.0.202 | osd.3 (233 GiB) | db3 (104) |

- **Speicher:** Ceph-Pool `vm-pool`, Replikation 3, 683 GiB Rohkapazität.
- **HA:** VM 100, 101, 102 werden vom Proxmox-HA-Manager verwaltet. db2 und db3 (103, 104) nicht, ihre Verfügbarkeit beruht nur auf der Galera-Replikation.

## Ergebnisse im Überblick
| Test | Szenario | Ergebnis |
|---|---|---|
| A | `systemctl stop mariadb` auf db2 | Galera 3 → 2 Knoten, Dienst lief weiter, per IST binnen Sekunden zurück auf 3 synchrone Knoten |
| B | Netzwerk-Link der VM db2 getrennt (`link_down=1`) | Ceph und Proxmox HA unberührt, Galera 3 → 2, Wiederherstellung per IST |
| C | geplanter Neustart von pve3 (CRM-Master) | neuer CRM-Master binnen Sekunden, physischer Boot ~4,5 Min, Ceph `HEALTH_WARN` ohne Datenverlust |
| D | Netzwerkkabel von pve2 für ~10-15 Min gezogen | keine HA-VM auf dem Knoten, MON-Quorum 2/3 blieb, `HEALTH_OK` binnen Minuten nach dem Wiederanschluss |
| E | pve1 über 20 Min heruntergefahren, danach Fencing von pve3 während der Wiederherstellung | HA-VMs starteten auf anderen Knoten, kein Datenverlust, volles `HEALTH_OK` etwa 48 Min nach Beginn |

## Test A: Anwendungsausfall (MariaDB-Stopp)
- **Szenario:** `systemctl stop mariadb` auf `db2`, die anderen zwei Knoten laufen weiter.
- **Ergebnis:** `wsrep_cluster_size` 3 → 2, die übrigen Knoten `Synced`. Eine Web-Testseite zeigte den Ausfall von db2 und Antworten von db1/db3. Proxmox HA war nicht beteiligt.
- **Wiederherstellung:** `systemctl start mariadb` löste einen inkrementellen Zustandstransfer (IST) aus, binnen Sekunden wieder 3 synchrone Knoten.

## Test B: Netzwerkisolation einer VM
- **Szenario:** `qm set 103 --net0 virtio,bridge=vmbr0,link_down=1` auf pve2 (wie das Ziehen des VM-Kabels, Host und Ceph-Netz blieben unberührt).
- **Ergebnis:** `ceph -s` unverändert, VM 103 blieb `running`, Galera fiel auf 2 Knoten. Die Schichten sind unabhängig, es ist nur ein Problem auf Anwendungsebene.
- **Wiederherstellung:** `link_down=0`, Galera IST, zurück auf 3/3.
- **Nebenvorfall:** Ich vergaß, `link_down=0` zurückzusetzen, was später beim Start von MariaDB `WSREP: failed to open gcomm backend connection` verursachte. Erst ein vollständiges `qm stop` / `qm start` half.

## Test C: geplanter Neustart eines Knotens (pve3, CRM-Master)
- **Ergebnis:** Der CRM-Master wechselte automatisch binnen Sekunden. Ceph zeigte `HEALTH_WARN` (reduzierte Redundanz), kein Rebalance, da der Ausfall kürzer als das 10-minütige `mon_osd_down_out_interval` war. Keine Auswirkung auf VMs (VM 100 lief nicht auf diesem Knoten).
- **Erkenntnis:** Die Wiederherstellungszeit wird hier vom physischen Boot der alten Hardware bestimmt (~4,5 Min), nicht vom Proxmox/Ceph-Mechanismus.

## Test D: ungeplanter Knotenausfall (pve2, Kabel)
- **Ergebnis:** Auf pve2 lief keine HA-VM, ein Failover war nicht nötig. Ceph war degradiert, aber das MON-Quorum 2/3 hielt. Nach dem Wiederanschluss kehrte `HEALTH_OK` von selbst binnen weniger Minuten zurück.

## Test E: Knotenausfall mit HA-VMs und ein zweiter Ausfall
- **Szenario:** `shutdown -h now` auf pve1 (VM 100, web1, db1, alle unter HA), Ausfall über 20 Minuten. Beim Wiederanlauf von pve1 fuhr ich pve3 versehentlich herunter, was ein zweites Fencing auslöste.
- **Zeitverlauf** (ungefähr, aus meinen Notizen, keine exakten Logzeiten):

| Zeit nach Shutdown | Ereignis |
|---|---|
| ~1 Min | MON-Quorum 2/3, osd.0 down, HA-Failover: vm:100 → pve2, vm:101 → pve3, vm:102 → pve2 |
| ~10 Min | Ceph markiert OSDs als `out`, Redundanz um 33 % reduziert |
| ~36 Min | pve1 eingeschaltet |
| ~38 Min | ungeplantes Fencing von pve3, vm:101 auf pve1 wiederhergestellt |
| ~42 Min | pve3 wieder online |
| ~48 Min | `HEALTH_OK`, 33 PGs `active+clean`, HA stabil |

- **Ergebnis:** Beide Ausfälle ohne Datenverlust und ohne manuellen Eingriff in die HA/Ceph-Schicht überstanden. db1 trat Galera automatisch wieder bei (`wsrep_cluster_size` zurück auf 3).
- **Wichtigste Erkenntnis:** *Ein gesundes Ceph und Quorum machen eine VM nicht automatisch hochverfügbar.* VM 104 (db3) war nicht im HA-Manager, blieb nach der Rückkehr von pve3 `stopped` und brauchte ein manuelles `qm start 104`. Jede kritische VM mit `ha-manager add vm:<ID> --state started` hinzufügen.

## Lehren für Produktionsdesigns
1. **Alle** kritischen VMs in den HA-Manager aufnehmen, nicht nur die wichtigsten.
2. Die Ceph-OSD-Kapazität gleichmäßig verteilen. Hier waren es etwa 20 % / 46 % / 34 % (pve1 / pve2 / pve3), was das Rebalancing beim Ausfall des größten Knotens erschwert.
3. Die Wiederherstellungszeit auf dieser alten Hardware wird vom Boot bestimmt (~4,5 Min). Auf moderner Hardware erwarte ich deutlich weniger, habe es aber nicht gemessen.
4. Kaskadierende Ausfälle testen (ein zweiter Knoten fällt während der Wiederherstellung aus), nicht nur einzelne Ausfälle.
5. Prüfen, dass Anwendungsdienste in den VMs beim Boot starten (`systemctl is-enabled`).

## Noch nicht getestet
- Ein echter physischer Plattenausfall (Platte ziehen, nicht nur den OSD-Dienst stoppen)
- Knotenausfall unter gleichzeitiger Last
- Ein vollständiger Ceph-Rebalance-Zyklus nach dem Markieren der OSDs als `out` (nur der Beginn wurde beobachtet)

## Befehle zur Zustandsprüfung
```bash
ceph -s
pvecm status
ha-manager status
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

## Konzeptionelle Parallelen zu AWS (Lernnotizen für SAA-C03)
Nur Analogien, die Dienste sind nicht gleichwertig.

| Proxmox / Ceph | Ähnliche AWS-Idee |
|---|---|
| Ceph-MON-Quorum (toleriert Ausfall von 1 von 3) | Multi-AZ-Failover (z. B. RDS) |
| Ceph-OSD-Replikation und Rebalancing | Replizierter Speicher über AZs |
| Proxmox-HA-Manager (CRM/LRM) | Auto-Scaling-Gruppe mit Health Checks |
| `mon_osd_down_out_interval` (10 Min) | Karenzzeit des Health Checks |

*Umfang: Laborergebnisse auf alter Hardware, keine Performance-Benchmarks. Zeiten sind ungefähr.*
