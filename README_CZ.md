# Proxmox 3-node HA cluster a Ceph: zpráva o testech výpadků

**Jazykové verze:** [English](README.md) | [Deutsch](README_DE.md)

**Prostředí:** domácí laboratoř, 20.-21. srpna 2026. 3 uzly Proxmox VE (pve1, pve2, pve3), úložiště Ceph, zátěž: webová VM s nginx + PHP-FPM a 3-node MariaDB Galera cluster. Starý hardware (Xeon E5410, SAS/SATA HDD).
**Autor:** Radim Kulaviak (isol.cz)

## Topologie
| Uzel | IP | Ceph OSD | VM |
|---|---|---|---|
| pve1 | 192.168.0.200 | osd.0 (137 GiB) | VM 100, web1 (101), db1 (102) |
| pve2 | 192.168.0.201 | osd.1, osd.2 (314 GiB) | db2 (103) |
| pve3 | 192.168.0.202 | osd.3 (233 GiB) | db3 (104) |

- **Úložiště:** Ceph pool `vm-pool`, replikace 3, 683 GiB raw kapacity.
- **HA:** VM 100, 101, 102 spravuje HA manager Proxmoxu. db2 a db3 (103, 104) ne, jejich dostupnost řeší jen replikace Galera.

## Výsledky přehledně
| Test | Scénář | Výsledek |
|---|---|---|
| A | `systemctl stop mariadb` na db2 | Galera 3 → 2 uzly, služba běžela, návrat na 3 synchronizované uzly přes IST během pár sekund |
| B | odpojení sítě VM db2 (`link_down=1`) | Ceph a Proxmox HA beze změny, Galera 3 → 2, obnova přes IST |
| C | plánovaný reboot pve3 (CRM master) | nový CRM master zvolen během sekund, fyzický boot ~4,5 min, Ceph `HEALTH_WARN` bez ztráty dat |
| D | vytažený síťový kabel pve2 na ~10-15 min | na uzlu neběžela HA VM, kvórum MON 2/3 drželo, `HEALTH_OK` do několika minut po připojení |
| E | pve1 vypnutý >20 min, pak fencing pve3 během obnovy | HA VM se spustily na jiných uzlech, bez ztráty dat, plné `HEALTH_OK` zhruba 48 min od začátku |

## Test A: výpadek aplikace (zastavení MariaDB)
- **Scénář:** `systemctl stop mariadb` na `db2`, zbylé dva uzly pokračují.
- **Výsledek:** `wsrep_cluster_size` 3 → 2, zbylé uzly `Synced`. Testovací webová stránka ukázala selhání db2 a odpověď db1/db3. Proxmox HA se neúčastnil.
- **Obnova:** `systemctl start mariadb` spustil inkrementální přenos stavu (IST), během pár sekund zpět na 3 synchronizované uzly.

## Test B: izolace sítě VM
- **Scénář:** `qm set 103 --net0 virtio,bridge=vmbr0,link_down=1` na pve2 (jako vytažení kabelu VM, hostitel a síť Ceph zůstaly nedotčené).
- **Výsledek:** `ceph -s` beze změny, VM 103 zůstala `running`, Galera klesla na 2 uzly. Vrstvy jsou nezávislé, jde jen o problém na úrovni aplikace.
- **Obnova:** `link_down=0`, Galera IST, zpět na 3/3.
- **Vedlejší incident:** zapomněl jsem vrátit `link_down=0`, což později způsobilo `WSREP: failed to open gcomm backend connection` při startu MariaDB. Pomohl až úplný `qm stop` / `qm start`.

## Test C: plánovaný reboot jednoho uzlu (pve3, CRM master)
- **Výsledek:** CRM master se přesunul automaticky během sekund. Ceph ukázal `HEALTH_WARN` (snížená redundance), bez rebalance, protože výpadek byl kratší než 10minutový `mon_osd_down_out_interval`. VM nebyly dotčeny (VM 100 na uzlu neběžela).
- **Poznatek:** doba obnovy je tu dána fyzickým bootem starého hardwaru (~4,5 min), ne mechanismem Proxmox/Ceph.

## Test D: neplánovaný výpadek uzlu (pve2, kabel)
- **Výsledek:** na pve2 neběžela žádná VM pod HA, failover nebyl potřeba. Ceph byl degradovaný, ale kvórum MON 2/3 drželo. Po připojení se `HEALTH_OK` vrátil sám během několika minut.

## Test E: výpadek uzlu s HA VM a druhý výpadek
- **Scénář:** `shutdown -h now` na pve1 (VM 100, web1, db1, všechny pod HA), výpadek přes 20 minut. Při návratu pve1 jsem omylem vypnul pve3, což vyvolalo druhý fencing.
- **Časová osa** (přibližně, z mých poznámek, ne přesné časy z logů):

| Čas od vypnutí | Událost |
|---|---|
| ~1 min | kvórum MON 2/3, osd.0 down, HA failover: vm:100 → pve2, vm:101 → pve3, vm:102 → pve2 |
| ~10 min | Ceph označí OSD jako `out`, redundance snížena o 33 % |
| ~36 min | pve1 zapnut |
| ~38 min | neplánovaný fence pve3, vm:101 obnovena na pve1 |
| ~42 min | pve3 zpět online |
| ~48 min | `HEALTH_OK`, 33 PG `active+clean`, HA stabilní |

- **Výsledek:** oba výpadky přežity bez ztráty dat a bez ručního zásahu do vrstvy HA/Ceph. db1 se sama připojila zpět do Galery (`wsrep_cluster_size` zpět na 3).
- **Hlavní poznatek:** *zdravý Ceph a kvórum samy o sobě nedělají VM vysoce dostupnou.* VM 104 (db3) nebyla v HA manageru, po návratu pve3 zůstala `stopped` a vyžadovala ruční `qm start 104`. Každou kritickou VM přidej příkazem `ha-manager add vm:<ID> --state started`.

## Poučení pro produkční návrh
1. Do HA manageru přidej **všechny** kritické VM, ne jen hlavní.
2. Rozlož kapacitu OSD v Ceph rovnoměrně. Zde to bylo zhruba 20 % / 46 % / 34 % (pve1 / pve2 / pve3), což komplikuje rebalance při výpadku nejtěžšího uzlu.
3. Doba obnovy na tomto starém hardwaru je dána hlavně bootem (~4,5 min). Čekám, že na moderním hardwaru bude výrazně nižší, ale neměřil jsem to.
4. Testuj kaskádové výpadky (druhý uzel padne během obnovy), ne jen jednotlivé výpadky.
5. Ověř, že aplikační služby uvnitř VM se spouštějí po startu (`systemctl is-enabled`).

## Zatím netestováno
- Skutečné selhání fyzického disku (vytažení disku, ne jen zastavení služby OSD)
- Výpadek uzlu při současné zátěži
- Kompletní cyklus rebalance v Ceph po označení OSD jako `out` (pozorován jen začátek)

## Příkazy ke kontrole stavu
```bash
ceph -s
pvecm status
ha-manager status
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

## Konceptuální paralely k AWS (poznámky ke studiu SAA-C03)
Jde jen o analogie, služby nejsou rovnocenné.

| Proxmox / Ceph | Podobná myšlenka v AWS |
|---|---|
| Kvórum Ceph MON (snese výpadek 1 ze 3) | Multi-AZ failover (např. RDS) |
| Replikace a rebalance Ceph OSD | Replikované úložiště napříč AZ |
| HA manager Proxmoxu (CRM/LRM) | Auto Scaling group s health checky |
| `mon_osd_down_out_interval` (10 min) | Odklad health checku |

*Rozsah: výsledky z laboratoře na starém hardwaru, žádné výkonnostní benchmarky. Časy jsou přibližné.*
