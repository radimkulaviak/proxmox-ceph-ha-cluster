# Proxmox 3-node HA cluster a Ceph: zpráva o testech výpadků

**Jazykové verze:** [English](README.md) | [Deutsch](README_DE.md)

**Prostředí:** domácí laboratoř. 3 uzly Proxmox VE (pve1, pve2, pve3), úložiště Ceph, zátěž: webový server Apache + 3-node MariaDB Galera cluster.
**Autor:** Radim Kulaviak (isol.cz)

## Topologie
- **pve1 (192.168.0.200):** OSD.0, VM pod správou HA (web1, db1)
- **pve2 (192.168.0.201):** OSD.1, OSD.2, VM db2
- **pve3 (192.168.0.202):** OSD.3, VM db3
- **Úložiště:** Ceph pool `vm-pool`, replikace 3

## Test A: výpadek aplikace (zastavení MariaDB)
- **Scénář:** `systemctl stop mariadb` na `db2`.
- **Výsledek:** velikost Galera clusteru klesla ze 3 na 2 (`wsrep_cluster_size`), ostatní uzly zůstaly synchronizované, Proxmox nebyl ovlivněn.
- **Obnova:** po spuštění MariaDB proběhl inkrementální přenos stavu (IST) a cluster se vrátil na 3 synchronizované uzly.

## Test B: kaskádové výpadky uzlů
- **Scénář:** `pve1` vypnut více než 20 minut; během obnovy pak vypadl (fencing) `pve3`.
- **Výsledek:** po návratu uzlů se Proxmox HA, Ceph a Galera vrátily do zdravého stavu.
- **Hlavní poznatek:** *zdravý Ceph a kvórum samy o sobě nezajistí vysokou dostupnost VM.* VM musí být přidaná do HA manageru (`ha-manager add vm:<ID> --state started`), jinak po výpadku hostitele zůstane ve stavu `stopped`.

## Příkazy pro kontrolu stavu
```bash
ceph -s
pvecm status
ha-manager status
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

## Koncepční paralely s AWS (poznámky ke zkoušce SAA-C03)
Jen analogie, služby si neodpovídají 1:1.

| Proxmox / Ceph | Podobná myšlenka v AWS |
|---|---|
| Kvórum Ceph MON (snese výpadek 1 ze 3) | Multi-AZ failover (např. RDS) |
| Replikace a rebalancing Ceph OSD | Replikované úložiště napříč AZ |
| Proxmox HA manager (CRM/LRM) | Auto Scaling group s kontrolou stavu |

*Rozsah: výsledky z laboratoře, bez výkonnostních testů.*
