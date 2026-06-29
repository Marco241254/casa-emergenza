# Operazioni — Backup, Recovery e Manutenzione

**Indice documentazione** | [README](README.md) | [Infrastruttura](00_infrastruttura.md) | [Home Assistant](01_home_assistant.md) | [PDC Hitachi](02_pdc_hitachi.md) | [Fotovoltaico](03_fotovoltaico.md) | [Tesla/Wallbox](04_tesla_wallbox.md) | [Energia](05_energia.md) | [Clima](06_clima_temperature.md) | [Riferimenti](08_riferimenti.md)

---

**Aggiornato:** 2026-06-23

---

## Indice

- [Architettura backup (3 livelli)](#architettura-backup-3-livelli)
- [Chiave di crittografia](#chiave-di-crittografia)
- [Inventario backup](#inventario-backup)
- [Scenario A — Recupero singola entita'/automation](#scenario-a--recupero-singola-entitaautomation)
- [Scenario B — Ripristino completo HAOS](#scenario-b--ripristino-completo-haos)
- [Scenario C — Ripristino LXC](#scenario-c--ripristino-lxc)
- [Scenario D — Disaster Recovery da dischi locali/NAS](#scenario-d--disaster-recovery-da-dischi-localinas)
- [Scenario E — Recupero da Google Drive (off-site)](#scenario-e--recupero-da-google-drive-off-site)
- [Backup off-site rclone (Livello 3)](#backup-off-site-rclone-livello-3)
- [Verifiche periodiche](#verifiche-periodiche)
- [Checklist manutenzione mensile](#checklist-manutenzione-mensile)
- [Deploy file su LXC](#deploy-file-su-lxc)
- [Verifiche post-riavvio HA](#verifiche-post-riavvio-ha)

---

## Architettura backup (3 livelli)

Tre livelli indipendenti e complementari. Uno NON sostituisce l'altro.

### Livello 1 — Home Assistant (granulare, veloce)

```
HAOS (VM 100) → Backup nativo HA → /backup locale
                              └──→ Google Drive add-on
                                   (marco.mourglia@gmail.com)

Pianificazione: ogni giorno alle 04:00 — conserva 7
Dimensione tipica: ~46 MB locale, ~270 MB su Drive
Protegge da: errore di configurazione, automation rotta
```

### Livello 2 — Proxmox locale (intera VM/LXC, "bare metal")

```
VM 100 (haos)          ──┐
VM 101 (haos-test)     ──┤
LXC 110 (influxdb)     ──┤
LXC 120 (grafana)      ──┼──→ nas_wd      (job 04:00)
LXC 130 (meteo-corrector)┤    (192.168.1.178, WD MyCloud)
LXC 131 (energy-alloc.)┤
LXC 200 (deb12-backup) ──┴──→ backup_wd_4tb (job 04:30)
                                (disco 4TB locale)

vzdump zstd — retention: keep-daily=7, keep-last=3,
            keep-weekly=4, keep-monthly=3, keep-yearly=2
Protegge da: VM/LXC corrotta, disco di sistema morto
```

### Livello 3 — Off-site su Google Drive (fuori casa)

```
backup_wd_4tb/dump → rclone (script notturno) →
                     Google Drive: proxmox-backups
                     (marco.mourglia@gmail.com)

Tiene gli ULTIMI 2 GIRI completi di tutte le 7 macchine
Pianificazione: ogni notte alle 06:00 (dopo i vzdump)
Dimensione: ~46 GB a giro; primo caricamento ~92 GB
Protegge da: incendio / furto / perdita TOTALE della casa
```

### Quando usare quale livello

| Cosa e' successo | Livello da usare |
|---|---|
| Automation rotta / dashboard / config HA | **L1** — piu' veloce, granulare |
| HAOS non parte / si e' corrotta | **L2** — ripristino VM intera |
| Perso un LXC (influxdb, grafana, ecc.) | **L2** |
| Morto il disco di sistema Proxmox | **L2** da `backup_wd_4tb` o `nas_wd` |
| Morto Proxmox **E** disco 4TB | **L2** dal NAS |
| Morto Proxmox **E** disco 4TB **E** NAS | **L3** — scarica da Google Drive |
| Morto **tutto** in casa (incendio/furto) | **L3** (Proxmox) + **L1** (HA granulare) da Drive |

---

## Chiave di crittografia

I backup HA (Livello 1) sono **crittografati**. Senza la chiave, anche se hai il file `.tar`, **non puoi ripristinare nulla**.

> I backup Proxmox (Livello 2 e 3, file `vzdump-*.zst`) **NON** usano questa chiave: si ripristinano direttamente.

### Dove trovare la chiave

- 📧 **Gmail** → cerca oggetto: `chiave crittografia back-up home assistant HA`
- Mittente: te stesso (`marco.mourglia@gmail.com`)
- Formato: `XXXX-XXXX-XXXX-XXXX-XXXX-XXXX-XXXX` (28 caratteri + trattini)

> **⚠️ NON salvare mai la chiave in chiaro in questo file o in altri file sul sistema.**

### Backup della chiave (oltre a Gmail)
- 📝 Carta nel cassetto / cassaforte (scritta a mano)
- 🔐 Password manager (1Password, Bitwarden, KeePass)
- 💾 Chiavetta USB conservata fuori casa

---

## Inventario backup

### Livello 1: Home Assistant

| Tipo | Quantita' | Dimensione totale | Dove |
|------|-----------|-------------------|------|
| Backup automatici giornalieri | 7 | ~1.8 GB | `/backup` locale + Google Drive |
| Backup aggiornamenti app | 6 | ~200 MB | Locali |
| Backup manuali | 10 | ~2.6 GB | Locali |
| **Google Drive (storico)** | **~30** | **~4.7 GB** | Account `marco.mourglia@gmail.com` |

Add-on responsabile: **"Home Assistant Google Drive Backup"** (sabeechen).

### Livello 2: Proxmox locale

**Macchine sottoposte a backup giornaliero (vzdump zstd) — TUTTE E 7:**

| ID | Tipo | Nome | Dimensione ultimo backup |
|----|------|------|-------------------------|
| 100 | QEMU/KVM | `haos` | ~19.4 GB |
| 101 | QEMU/KVM | `haos-test` | ~6.6 GB |
| 110 | LXC | `influxdb` | ~2.6 GB |
| 120 | LXC | `grafana` | ~0.4 GB |
| 130 | LXC | `meteo-corrector` | ~0.5 GB |
| 131 | LXC | `energy-allocator` | ~0.6 GB |
| 200 | LXC | `deb12-backup` | ~16.1 GB |

**Un giro completo ≈ 46 GB.**

**Due job vzdump (doppia copia locale):**

| Job | Orario | Storage | Disco fisico |
|-----|--------|---------|-------------|
| Job 1 | 04:00 | `nas_wd` | NAS WD MyCloud 192.168.1.178 |
| Job 2 | 04:30 | `backup_wd_4tb` | Disco 4TB locale Proxmox |

**Retention:** `keep-daily=7, keep-last=3, keep-weekly=4, keep-monthly=3, keep-yearly=2` su entrambi.

### Livello 3: Off-site Google Drive

| Voce | Valore |
|------|--------|
| Strumento | `rclone` installato sull'host Proxmox |
| Remote | `gdrive` (scope `drive.file`) |
| Account | `marco.mourglia@gmail.com` |
| Destinazione | `gdrive:proxmox-backups` |
| Sorgente | `/mnt/pve/backup_wd/dump` |
| Quando | Ogni notte alle 06:00 |
| Quanti giri | `KEEP=2` (ultimi 2 giri = ~92 GB) |

---

## Scenario A — Recupero singola entita'/automation

**Quando:** hai modificato un file YAML / un'automation / una dashboard e qualcosa si e' rotto. HA gira ancora.

1. Apri HA: `http://192.168.1.53:8123`
2. Vai su **Impostazioni → Backups**
3. Clicca sul backup piu' recente **antecedente al guasto**
4. Scegli **"Ripristina backup parziale"**
5. Seleziona **solo** cio' che ti serve (es. *Configurazione Home Assistant* senza i container/add-on)
6. Inserisci la **chiave di crittografia**
7. Conferma → HA si riavvia automaticamente

### Recupero manuale di un singolo file YAML

Se ti serve **solo un file specifico** (es. `300_riduz_cons_energ.yaml`):

1. Scarica il `.tar` di backup HA da Google Drive (o lista locale)
2. Decifra ed estrai:
   ```bash
   tar -xf backup_2026_05_22.tar
   tar -xzf homeassistant.tar.gz
   ```
3. Naviga in `data/` e copia il singolo file
4. Caricalo via **File editor** su HA

---

## Scenario B — Ripristino completo HAOS

**Quando:** la VM HAOS non parte, e' corrotta, o ha avuto un aggiornamento andato male.

### Opzione 1 (preferita): ripristino da Proxmox

1. Apri Proxmox: `https://192.168.1.100:8006`
2. Naviga su **Datacenter → proxmox → backup_wd_4tb → Backup**
3. Trova l'ultimo `vzdump-qemu-100-AAAA_MM_GG-04_30_XX.vma.zst` buono
4. Clicca **Ripristino**
5. Configura:
   - **Storage**: `local-lvm`
   - **VM ID**: `100` (sovrascrive)
6. Conferma → attendi 5-10 minuti
7. Avvia la VM: **VM 100 → Start**
8. Apri HA su `http://192.168.1.53:8123` per verifica

> **⚠️ ATTENZIONE:** il ripristino sovrascrive la VM esistente. Se devi confrontare, ripristina prima su un **VM ID diverso** (es. 199).

### Opzione 2: ripristino HA dentro a HAOS gia' funzionante

Se HAOS parte ma la configurazione e' incasinata:

1. Vai su **Impostazioni → Backups → Mostra tutti i backup**
2. Scegli un backup pulito (locale o Google Drive)
3. **"Ripristina backup completo"**
4. Inserisci la **chiave di crittografia**
5. HA si riavvia e ripristina tutto

### Verifica post-ripristino HA

- [ ] Dashboard principale
- [ ] Add-on (ESPHome, File editor, HACS)
- [ ] Integrazioni (Tesla, Hitachi Yutaki, ESPHome devices)
- [ ] Entita' Tesla BLE / FV / PDC tornano online
- [ ] Automation attive
- [ ] InfluxDB connesso

---

## Scenario C — Ripristino LXC

**Quando:** un container LXC si e' corrotto. Procedura uguale per tutti gli LXC (110, 120, 130, 131, 200).

1. Proxmox → **backup_wd_4tb → Backup**
2. Seleziona il file `vzdump-lxc-<ID>-*.tar.zst` piu' recente
3. Clicca **Ripristino**
4. Storage: `local-lvm`, CT ID: lo stesso numero dell'LXC
5. Conferma e avvia il container

### Note per container specifici

- **110 (influxdb)** ⚠️ **CRITICO**: contiene tutta la storia delle misure. Il ripristino sovrascrive i dati: se ti serve la storia recente, **NON ripristinare un backup vecchio**.
- **200 (deb12-backup)**: contiene MariaDB del Recorder HA. Dopo il ripristino verifica che HA riprenda a registrare la storia.
- **120 / 130 / 131**: dopo l'avvio, controlla che i cron e gli script in `/opt/...` ripartano.

---

## Scenario D — Disaster Recovery da dischi locali/NAS

**Quando:** Proxmox non parte piu', disco di sistema morto, ma **almeno un disco di backup e' vivo**.

### Caso 1: disco di sistema morto, NAS e disco 4TB OK

1. **Reinstalla Proxmox VE 9.x** sul nuovo disco
2. Configura rete con IP `192.168.1.100`, hostname `proxmox`
3. Login web: `https://192.168.1.100:8006`
4. **Rimonta il disco 4TB** come storage:
   - `Datacenter → Storage → Add → Directory`
   - ID: `backup_wd_4tb`, Content: `VZDump backup file`
5. **Rimonta il NAS**:
   - `Datacenter → Storage → Add → NFS`
   - Server: `192.168.1.178`, ID: `nas_wd`
6. Ripristina in ordine: VM 100 → LXC 110, 200 → LXC 120, 130, 131 → VM 101
7. Ricrea i **due job di backup** con retention standard
8. Ricrea il **backup off-site rclone** (vedi sotto)

### Caso 2: Proxmox e disco 4TB morti, solo NAS sopravvive

Identico al caso 1 ma usa **solo** `nas_wd`. Una volta ripartito, ricollega un nuovo disco 4TB e ricrea il job duplicato + script off-site.

---

## Scenario E — Recupero da Google Drive (off-site)

**Quando:** hai perso **TUTTE** le copie locali (Proxmox + disco 4TB + NAS). Tipico di incendio/furto.

> **La cosa piu' importante:** i file su Google Drive **si scaricano normalmente dal sito** `drive.google.com`. **NON serve rclone per recuperarli.**

### Metodo SEMPLICE (consigliato)

1. **Procurati l'hardware** e installa Proxmox (IP 192.168.1.100)
2. **Scarica i backup** da `drive.google.com` → cartella `proxmox-backups`
3. **Carica su Proxmox** via `scp`:
   ```powershell
   scp $env:USERPROFILE\Downloads\vzdump-qemu-100-*.vma.zst root@192.168.1.100:/var/lib/vz/dump/
   ```
4. **Ripristina dal pannello Proxmox** — per ogni file: Ripristino → stesso VM/CT ID
5. Avvia prima **VM 100 (haos)**, poi gli altri
6. Verifica HA su `http://192.168.1.53:8123`

### Metodo alternativo (rclone, da shell)

```bash
apt update && apt install -y rclone
# Ricrea connessione gdrive (vedi sezione sotto)
mkdir -p /var/lib/vz/dump
rclone copy gdrive:proxmox-backups /var/lib/vz/dump -P
```

---

## Backup off-site rclone (Livello 3)

### File e impostazioni chiave

| Cosa | Percorso / valore |
|------|-------------------|
| Configurazione rclone (contiene il token!) | `/root/.config/rclone/rclone.conf` |
| Nome remote | `gdrive` |
| Scope | `drive.file` (rclone vede solo i propri file) |
| Script | `/usr/local/bin/backup-to-drive.sh` |
| Cron | `/etc/cron.d/rclone-backup` → ogni notte 06:00 |
| Log | `/var/log/rclone-backup.log` |
| Cartella su Drive | `gdrive:proxmox-backups` |

### Comandi di manutenzione

```bash
# Vedere com'e' andato l'ultimo backup
tail -n 40 /var/log/rclone-backup.log

# Lanciare SUBITO a mano
systemd-run --unit=backup-drive-now --collect /usr/local/bin/backup-to-drive.sh

# Vedere cosa c'e' ORA su Google Drive
rclone lsf gdrive:proxmox-backups
```

### Cambiare frequenza

```bash
# Ogni giorno (attuale)
0 6 * * *
# Solo domenica (settimanale)
0 6 * * 0
```

### Rimettere in piedi rclone dopo disastro

1. Installa: `apt update && apt install -y rclone`
2. Ricrea connessione `gdrive`:
   - **Modo veloce**: se hai salvato `rclone.conf` (~535 byte), rimettilo in `/root/.config/rclone/rclone.conf`
   - **Modo da zero**: `rclone config` → `n` → nome `gdrive` → tipo `drive` → scope `3` (drive.file) → autorizza con `marco.mourglia@gmail.com`
3. Ricrea lo script e il cron

> **💾 Consiglio:** tieni una copia di `rclone.conf` su chiavetta fuori casa. E' minuscolo e ti fa saltare il passaggio 2.

---

## Verifiche periodiche

### Test rapido HA (mensile)

1. Vai su **HA → Impostazioni → Backups**
2. Scarica l'ultimo backup completo
3. Verifica dimensione sensata (>40 MB)
4. Apri — deve contenere `homeassistant.tar.gz`

### Test backup off-site (mensile)

1. Shell Proxmox: `tail -n 40 /var/log/rclone-backup.log` → `===== ... FINE =====` recente
2. `rclone lsf gdrive:proxmox-backups` → ~14 file `vzdump-*.zst` (2 giri × 7 macchine)
3. `drive.google.com` → cartella `proxmox-backups` → file recenti presenti

### Test ripristino HA in VM separata (trimestrale)

1. Crea VM HAOS in Proxmox (VM ID 199)
2. Avvia → "Restore from backup"
3. Carica backup recente + chiave
4. Verifica che parta → **cancella VM di test**

### Test ripristino LXC (trimestrale)

1. Scegli LXC piccolo (es. 130, ~0.5 GB)
2. Ripristina su CT ID 999
3. Avvia, verifica → **cancella container 999**

### Verifica chiave crittografia (semestrale)

1. Apri Gmail, cerca `chiave crittografia back-up home assistant HA`
2. Verifica che la chiave salvata corrisponda a quella in HA
3. Annotala fisicamente in un secondo posto sicuro

---

## Checklist manutenzione mensile

```
[ ] HA: backup automatico ultimo eseguito con successo
[ ] HA: Google Drive add-on online — token non scaduto
[ ] Proxmox: entrambi i job vzdump (04:00 + 04:30) OK ultima settimana
[ ] Proxmox: spazio disponibile su backup_wd_4tb > 20%
[ ] Proxmox: spazio disponibile su nas_wd > 20%
[ ] NAS WD: raggiungibile su 192.168.1.178
[ ] OFF-SITE rclone: log con "FINE" recente
[ ] OFF-SITE rclone: ~14 file vzdump su gdrive:proxmox-backups
[ ] Google Drive: spazio libero sufficiente (~92 GB per 2 giri)
[ ] Chiave crittografia HA: ancora presente in Gmail
[ ] Copia di rclone.conf su chiavetta fuori casa: presente
[ ] Retention: nessun "buco" nella sequenza di backup ultimi 7 giorni
```

---

## Deploy file su LXC

### Su LXC 130 (meteo-corrector)

```powershell
# PowerShell Windows
scp "C:\Users\marco\Downloads\pvlib_ha2.py" root@192.168.1.100:/tmp/
```
```bash
# Shell Proxmox
pct exec 130 -- cp /opt/meteo_corrector/pvlib_ha2.py /opt/meteo_corrector/pvlib_ha2.py.bak
pct push 130 /tmp/pvlib_ha2.py /opt/meteo_corrector/pvlib_ha2.py
pct exec 130 -- /opt/meteo_corrector/venv/bin/python /opt/meteo_corrector/pvlib_ha2.py
```

### Sequenza completa — modifica e deploy LXC 130

```powershell
# 1. Carica su Proxmox
scp "C:\Users\marco\Downloads\pvlib_ha2.py" root@192.168.1.100:/tmp/
```
```bash
# 2. Sposta nel LXC + esegui
pct push 130 /tmp/pvlib_ha2.py /opt/meteo_corrector/pvlib_ha2.py
pct exec 130 -- /opt/meteo_corrector/venv/bin/python /opt/meteo_corrector/pvlib_ha2.py
```

---

## Verifiche post-riavvio HA

Dopo ogni riavvio di Home Assistant:

- [ ] Nessun errore in **Impostazioni → Sistema → Registri**
- [ ] Modbus PDC: `sensor.pdc_r_opstate` ha valore numerico
- [ ] `addr1003` = 3 (Fix)
- [ ] `addr1088` = 2 (CentralMode = Water)
- [ ] `addr1094` = 0 (H-LINK OK)
- [ ] ESP32 Tesla BLE: `binary_sensor.tesla_ble_pronta` = ON (se auto in garage)
- [ ] Previsioni FV: sensori `pvlib_oggi_*` popolati (ore 6-20)
- [ ] InfluxDB connesso
- [ ] Recorder MariaDB: entita' storiche caricano

---

*Documento generato il 2026-06-23 — Integra Recupero_Backup_HA_Proxmox v28/05/2026*
