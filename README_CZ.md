# Proxmox 3-uzlový HA cluster a testovací zpráva Ceph

**Jazykové verze:** [English](README.md) | [Deutsch](README_DE.md)

**Prostředí (lab):** 3× Proxmox VE (pve1, pve2, pve3), úložiště Ceph, zátěž: Apache + 3-uzlový MariaDB Galera cluster.
**Autor:** Radim Kulaviak ([isol.cz](https://www.isol.cz))

## Topologie
- **pve1:** OSD.0, HA spravované VM (web1, db1)
- **pve2:** OSD.1, OSD.2, VM db2
- **pve3:** OSD.3, VM db3
- **Úložiště:** Ceph pool `vm-pool`, replikace = 3

## Testovací scénáře a zjištění

### Test A: Výpadek aplikační vrstvy (zastavení MariaDB)
- **Scénář:** `systemctl stop mariadb` na `db2`.
- **Výsledek:** `wsrep_cluster_size` klesla ze 3 na 2; zbylé uzly zůstaly synchronizované; vrstva Proxmox nebyla dotčena.
- **Obnova:** Start MariaDB spustil inkrementální přenos stavu (IST); cluster se během několika sekund vrátil na 3 synchronizované uzly.

### Test B: Výpadek uzlu a neplánovaný druhý výpadek
- **Scénář:** `pve1` vypnut déle než 20 minut; během obnovy byl neočekávaně fencován `pve3`.
- **Výsledek:** Proxmox HA, Ceph a Galera se obnovily bez ručních zásahů do dat. V tomto labu nebyla pozorována ztráta dat.
- **Poučení:** *Redundance Ceph/kvora není totéž co vysoká dostupnost VM.* VM je nutné explicitně přidat do HA manageru (`ha-manager add vm:<ID> --state started`), jinak po výpadku hostitele zůstanou ve stavu `stopped`.

### Limity testu
Tři uzly, laboratorní hardware, jedna sekvence výpadků. Test ukazuje chování a můj postup, nejde o benchmark výkonu ani kapacity.
