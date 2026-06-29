# Infrastruttura IT — Casa GEEKOM

**Indice documentazione** | [README](README.md) | [Home Assistant](01_home_assistant.md) | [PDC Hitachi](02_pdc_hitachi.md) | [Fotovoltaico](03_fotovoltaico.md) | [Tesla/Wallbox](04_tesla_wallbox.md) | [Energia](05_energia.md) | [Clima](06_clima_temperature.md) | [Operazioni](07_operazioni.md) | [Riferimenti](08_riferimenti.md)

---

**Aggiornato:** 2026-06-23

---

## Indice

- [Cluster Proxmox](#cluster-proxmox)
- [VM e LXC — inventario completo](#vm-e-lxc--inventario-completo)
- [Rete](#rete)
- [Dispositivi di rete](#dispositivi-di-rete)
- [Dispositivi ESPHome](#dispositivi-esphome)
- [Storage](#storage)
- [Come trasferire file](#come-trasferire-file)

---

## Cluster Proxmox

| Nodo | Ruolo | Note |
|------|-------|------|
| **GEEKOM** | Primario — tutti i workload produzione | Host principale, Proxmox VE 9.1.1 |
| **BEELINK** | Standby caldo (spento) | Samsung 990 PRO 2 TB — non ancora attivo |

**Target architetturale:** warm standby con LXC replicati inclusi MariaDB e InfluxDB.

### Accesso Proxmox

| Voce | Valore |
|------|--------|
| URL | `https://192.168.1.100:8006` |
| Login | `root@pam` |
| IP host | `192.168.1.100` |

---

## VM e LXC — inventario completo

### Mappa macchine

| ID | Nome | Tipo | IP | Scopo |
|----|------|------|----|-------|
| 100 | `haos` | VM QEMU | 192.168.1.53 | **Home Assistant OS** — sistema principale |
| 101 | `haos-test` | VM QEMU | — | HA di test — NON toccare in produzione |
| 110 | `influxdb` | LXC | 192.168.1.48 | Database time-series InfluxDB |
| 120 | `grafana` | LXC | — | Dashboard Grafana |
| 130 | `meteo-corrector` | LXC | 192.168.1.68 | Previsioni FV pvlib + correttore k |
| 131 | `energy-allocator` | LXC | — | Allocatore energia PDC (AppDaemon-like) |
| 200 | `deb12-backup` | LXC | 192.168.1.165 | MariaDB per Recorder HA + backup |

> **Backup:** tutte e 7 le macchine sono inclusse nei job vzdump (Livello 2). Confermato 2026-05-28.

### Dettaglio VM 100 — HAOS

| Voce | Valore |
|------|--------|
| IP | 192.168.1.53 |
| URL | `http://192.168.1.53:8123` |
| Database | MariaDB (migrato da SQLite) |
| Path config | `/root/config/` (o `/config/` dentro HA) |
| Packages | 40+ file YAML in `/config/packages/` |
| Lovelace | `/config/views/` (file iniziano con `views:`) |

**App daemon path:** `/mnt/data/supervisor/addon_configs/a0d7b954_appdaemon/apps/`

### Dettaglio LXC 130 — meteo-corrector

| Voce | Valore |
|------|--------|
| IP fisso | 192.168.1.68 (assegnato via UniFi) |
| Path base | `/opt/meteo_corrector/` |
| Python venv | `/opt/meteo_corrector/venv/bin/python` |
| Log | `/var/log/pvlib.log`, `/var/log/correttore.log` |
| Cron | ogni ora alle `:00` |
| DNS fix post-restart | `echo 'nameserver 8.8.8.8' > /etc/resolv.conf` |

**Script principali:**

| File | Scopo | Pubblica su HA? |
|------|-------|-----------------|
| `pvlib_ha2.py` | Calcola previsioni FV 4 array, pubblica ~100 sensori via REST API | ✅ Sì |
| `correttore.py` | Calcola fattore k reale/previsto | ✅ Sì |
| `test_domani.py` | Stampa previsione domani a schermo | ❌ No |
| `test_domani2.py` | Stampa oggi e domani a schermo | ❌ No |
| `ha_publisher.py` | Stub vecchio — non usato | ❌ No |

**Parametri chiave pvlib_ha2.py:**

```python
LAT, LON, ALT = 44.88, 7.33, 626    # coordinate impianto FV
HA_URL = "http://192.168.1.53:8123"  # VM 100 haos
INFLUX = "http://192.168.1.48:8086"  # LXC 110
```

**4 Array FV modellati:**

| Nome | Azimuth | Tilt | kWp | Cap inverter |
|------|---------|------|-----|-------------|
| sud | 170° | 58° | 7.200 | 6.00 kW |
| est_a | 85° | 30° | 2.625 | 2.03 kW |
| est_b | 85° | 30° | 2.700 | 2.16 kW |
| ovest | 265° | 30° | 3.750 | 2.90 kW |

**Correzioni ombreggiamento:**
- SUD (staccionata pomeriggio): 14:00→0.90, 15:00→0.75, 16:00→0.64, 17:00→0.33, 18:00→0.20, 19:00→0.05
- OVEST (montagna pomeriggio): 16:00→0.80, 17:00→0.30, 18:00→0.10, 19:00→0.02

**Parametri chiave correttore.py:**
```python
K_MIN = 0.55       # clamp minimo fattore k
K_MAX = 1.10       # clamp massimo fattore k
K_DEFAULT = 0.85   # default se non ci sono 14gg di storia
ORA_GIORNO_CHIUSO = 21  # pubblica k_oggi solo dopo le 21
```

### Dettaglio LXC 131 — energy-allocator

| Voce | Valore |
|------|--------|
| Path base | `/opt/allocator/` |
| Cron | ogni notte alle 23:30 |
| Log | `/var/log/allocator.log` |

| File | Scopo |
|------|-------|
| `allocator.py` | Logica principale allocazione energia |
| `calc_pbase.py` | Calcolo potenza base PDC |
| `inputs.py` | Lettura input da HA |
| `pdc_cal.py` | Calibrazione PDC |
| `thermal_cal.py` | Calibrazione termica |

---

## Rete

- **Router:** Ubiquiti Dream Router 7 (UDR7)
- **Switch:** TP-Link managed
- **IP fissi:** assegnati via UniFi
- **DNS fix LXC 130:** dopo restart, `echo 'nameserver 8.8.8.8' > /etc/resolv.conf`

---

## Dispositivi di rete

| IP | Dispositivo | Porta | Protocollo | Note |
|----|-------------|-------|------------|------|
| 192.168.1.53 | Home Assistant (VM 100) | 8123 | HTTP | HAOS produzione |
| 192.168.1.100 | Proxmox host GEEKOM | 8006 | HTTPS | Login: root@pam |
| 192.168.1.48 | InfluxDB (LXC 110) | 8086 | HTTP | DB: home_assistant |
| 192.168.1.165 | MariaDB (LXC 200) | 3306 | MySQL | DB: homeassistant, utente: ha |
| 192.168.1.5 | Gateway Modbus Hitachi ATW-MBS-02 | 502 | TCP Modbus | Bridge H-LINK↔Modbus |
| 192.168.1.175 | ESP32 "Tesla BLE 770224" | 6053 | ESPHome API | Garage, BLE→Tesla |
| 192.168.1.176 | ESP32 "Tesla BLE Cancello" | 6053 | ESPHome API | Cancello, BLE→Tesla |
| 192.168.1.178 | NAS WD MyCloud EX2 Ultra | — | NFS/CIFS | Storage backup Proxmox |
| 192.168.1.57 | Grafana (LXC 120) | — | HTTP | Dashboard storiche |

---

## Dispositivi ESPHome

3 device, 115 entita' totali:

| Device | IP | Stanza | Entita' | Scopo |
|--------|----|--------|---------|-------|
| Termosifone-Ingresso | — | Cucina | 29 | Controllo termosifone ingresso |
| Termosifone-Televisione | — | Cucina | 29 | Controllo termosifone televisione |
| Tesla BLE 770224 | 192.168.1.175 | Garage | 57 | Bridge BLE Tesla — package 210 |
| Tesla BLE Cancello | 192.168.1.176 | Cancello | — | Bridge BLE Tesla — package 211 |

---

## Storage

| Nome | Tipo | Contenuto | Posizione fisica |
|------|------|-----------|-----------------|
| `local` | Directory | ISO, template | Disco sistema Proxmox |
| `local-lvm` | LVM-Thin | Dischi VM/LXC live | Disco sistema Proxmox |
| `backup_wd_4tb` | Directory | Backup vzdump (`/mnt/pve/backup_wd/dump`) | Disco 4TB locale Proxmox |
| `nas_wd` | NFS/CIFS | Backup vzdump (`/mnt/pve/nas_wd/dump`) | NAS WD MyCloud 192.168.1.178 |

---

## Come trasferire file

### Situazione
- Niente SSH diretto con tastiera italiana (caratteri speciali impossibili)
- Niente Samba
- Niente terminale HA comodo

### Procedura collaudata (2026-05-16)

**Step 1** — Scarica il file dal browser

**Step 2** — PowerShell su Windows:
```powershell
scp $env:USERPROFILE\Downloads\nomefile.py root@192.168.1.100:/tmp/
```
Chiede la password di root Proxmox. Il file arriva in `/tmp/` sull'host.

**Step 3** — Shell Proxmox (browser → 192.168.1.100:8006 → nodo proxmox → Shell):
```bash
# Backup del file originale
pct exec 130 -- cp /opt/meteo_corrector/nomefile.py /opt/meteo_corrector/nomefile.py.bak

# Copia nuovo file nel container
pct push 130 /tmp/nomefile.py /opt/meteo_corrector/nomefile.py
```

**Sostituisci 130 con il numero del container giusto.**

### Per file HA (packages, configuration.yaml)
```powershell
scp $env:USERPROFILE\Downloads\900_hitachi_control.yaml root@192.168.1.100:/tmp/
```
```bash
pct push 100 /tmp/900_hitachi_control.yaml /root/config/packages/900_hitachi_control.yaml
```

### Scaricare file da LXC a Windows
```bash
# Da shell Proxmox
pct exec 130 -- cat /opt/meteo_corrector/pvlib_ha2.py > /tmp/pvlib_ha2.py
```
```powershell
# Da PowerShell Windows
scp root@192.168.1.100:/tmp/pvlib_ha2.py "C:\Users\marco\Downloads\pvlib_ha2.py"
```

---

*Documento generato il 2026-06-23*
