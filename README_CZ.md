# Proxmox 3-Uzlovy HA Cluster & Ceph Testovaci Zprava

**Jazykove verze:** [English](README.md) | [Deutsch](README_DE.md)

**Prostredi:** 3x Proxmox VE uzly (pve1, pve2, pve3), Ceph HA uloziste, zatez: Apache web server + 3-uzlovy MariaDB Galera Cluster.
**Autor:** Radim Kulaviak (isol.cz)

## Topologie a Architektura
- **pve1 (192.168.0.200):** OSD.0, hostuje HA spravovane VM (web1, db1).
- **pve2 (192.168.0.201):** OSD.1, OSD.2, hostuje db2.
- **pve3 (192.168.0.202):** OSD.3, hostuje db3.
- **Uloziste:** Ceph pool `vm-pool`, replikace = 3.

---

## Klicove testovaci scenare a zjesteni

### Test A: Vypadek aplikacni vrstvy (Zastaveni MariaDB)
- **Scenar:** `systemctl stop mariadb` na uzlu `db2`.
- **Vysledek:** Velikost Galera clusteru klesla z 3 na 2 (`wsrep_cluster_size`). Zbyvajici uzly zustaly plne synchronizovany. Infrastrukturni vrstva Proxmox ostala nedotcena.
- **Obnova:** Start MariaDB spustil inkrementalni prenos stavu (IST) a cluster se behem par sekund vratil do 3-uzloveho synchronizovaneho stavu.

### Test B: Kaskadovy vypadek infrastruktury a uzlu
- **Scenar:** Kompleutni vypnuti hlavniho uzlu `pve1` (>20 min) v kombinaci s sekundarnim vypadkem/fencingem `pve3` behem obnovy.
- **Vysledek:** Cluster uspesne provadel automaticke samo-hojeni napric vrstvami Proxmox HA, Ceph a Galera bez ztraty dat.
- **Klicove produkcni pouceni:** *Infrastrukturni HA (Ceph/Quorum) neznamena automaticky VM HA.* Virtualni stroje musi byt explicitne pridany do HA manageru (`ha-manager add vm:<ID> --state started`), jinak po vypadku hostitele a obnove zustanou ve stavu `stopped`.
