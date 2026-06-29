# Riferimenti — Contatti, Modelli e Credenziali

**Indice documentazione** | [README](README.md) | [Infrastruttura](00_infrastruttura.md) | [Home Assistant](01_home_assistant.md) | [PDC Hitachi](02_pdc_hitachi.md) | [Fotovoltaico](03_fotovoltaico.md) | [Tesla/Wallbox](04_tesla_wallbox.md) | [Energia](05_energia.md) | [Clima](06_clima_temperature.md) | [Operazioni](07_operazioni.md)

---

**Aggiornato:** 2026-06-23

---

## Indice

- [Contatti tecnici](#contatti-tecnici)
- [Mappa IP completa](#mappa-ip-completa)
- [Hardware — modelli e seriali](#hardware--modelli-e-seriali)
- [Storage Proxmox](#storage-proxmox)
- [Riferimenti esterni](#riferimenti-esterni)
- [Pannello di emergenza](#pannello-di-emergenza)

---

## Contatti tecnici

| Nome | Telefono | Specializzazione |
|------|----------|-----------------|
| Marco Mourglia | 335-1354309 | Internet, sistema generale, Home Assistant |
| Elvis Martinat (BLUMAR — San Germano) | 347-4958076 | Idraulica, gas, PDC |
| Alessandro Reale (IS ENERGY — Abbadia Alpina) | 370-3068232 | Fotovoltaico, PDC, VMC |
| Claudio Massello | 348-6977962 | Impianto elettrico |

---

## Mappa IP completa

### Infrastruttura

| IP | Dispositivo | Porta | Protocollo |
|----|-------------|-------|------------|
| 192.168.1.53 | Home Assistant (VM 100) | 8123 | HTTP |
| 192.168.1.100 | Proxmox host GEEKOM | 8006 | HTTPS |
| 192.168.1.48 | InfluxDB (LXC 110) | 8086 | HTTP |
| 192.168.1.57 | Grafana (LXC 120) | — | HTTP |
| 192.168.1.68 | meteo-corrector (LXC 130) | — | — |
| 192.168.1.165 | MariaDB (LXC 200) | 3306 | MySQL |

### Dispositivi fisici

| IP | Dispositivo | Porta | Note |
|----|-------------|-------|------|
| 192.168.1.5 | Gateway Modbus Hitachi ATW-MBS-02 | 502 TCP | Bridge H-LINK↔Modbus |
| 192.168.1.175 | ESP32 "Tesla BLE 770224" | 6053 | Garage, BLE→Tesla |
| 192.168.1.176 | ESP32 "Tesla BLE Cancello" | 6053 | Cancello, BLE→Tesla |
| 192.168.1.178 | NAS WD MyCloud EX2 Ultra | — | Storage backup |
| 192.168.1.181 | BEELINK (spento) | — | Warm standby |

### Shelly monitorati

| MAC / Nome | Funzione |
|------------|----------|
| `485519dc8109` | Shelly EM3 — scambio rete ENEL trifase |
| `c8c9a33e5ca7` | Shelly EM3 — a monte wallbox (feedback PID Tesla) |
| `c45bbee1c642` | Shelly EM — PDC (canale 1) |
| `90150687a670` | Shelly Plus 1 — R1 abilita PDC |
| `1c692008dea0` | Shelly Plus 1 — R2 circolatore |

---

## Hardware — modelli e seriali

### Fotovoltaico

| Componente | Modello | Codice/SN | Note |
|------------|---------|-----------|------|
| Inverter stringa Sud | SolarEdge SE6000H | — | 6.00 kW, L3 |
| Inverter stringa Est | Enphase IQ7A / IQ8AC | — | 2.03 + 1.08 + 1.08 kW, L1 |
| Inverter stringa Ovest | Enphase IQ7A | — | 2.90 kW, L2 |
| Pannelli Sud (18x) | SunPower SPR-MAX3-400 | — | 400W, 7.20 kWp |
| Pannelli Est-A (7x) | SunPower P3-AC 375W | — | 375W, 2.63 kWp |
| Pannelli Est-B (3+3x) | SunPower P7-dc 450W | — | 450W, 2.70 kWp |
| Pannelli Ovest (10x) | SunPower P3-AC 375W | — | 375W, 3.75 kWp |

### Batterie

| Componente | Modello | Capacita' |
|------------|---------|-----------|
| Batteria SolarEdge | Li-Ion | 9.70 kWh |
| Batteria Enphase Est | IQ Battery 5P ×1 | 5.00 kWh |
| Batteria Enphase Ovest | IQ Battery 5P ×1 | 5.00 kWh |
| **Totale** | | **19.70 kWh** |

### Pompa di calore

| Componente | Modello | Codice/SN |
|------------|---------|-----------|
| PDC | Hitachi RASM-4VNE | — |
| Controller | PC-ARFH1E | — |
| Gateway Modbus | Hitachi ATW-MBS-02 | S/N 7E549924 |
| Contatore termico | Caleffi CONTECA EASY 1"1/4 RS485 | 7504/7507 |

### Tesla

| Componente | Modello/Codice |
|------------|----------------|
| Auto | Tesla Model Y Long Range |
| VIN | XP7YGCES1RB482601 |
| BLE MAC | 40:79:12:8B:95:53 |
| Wall Connector Gen 3 | TSN: PGT24118019155 |
| Capacita' utile calcolo | 60 kWh (package 210), 55 kWh (allocator) |

### ESP32

| Dispositivo | Board | IP | MAC WiFi | BLE MAC auto |
|-------------|-------|-----|----------|-------------|
| Tesla BLE Garage | esp32dev | 192.168.1.175 | 88:57:21:77:02:24 | 40:79:12:8B:95:53 |
| Tesla BLE Cancello | esp32dev | 192.168.1.176 | 88:57:21:6D:AA:B0 | 40:79:12:8B:95:53 |

---

## Storage Proxmox

| Nome | Tipo | Contenuto | Posizione fisica |
|------|------|-----------|-----------------|
| `local` | Directory | ISO, template | Disco sistema Proxmox |
| `local-lvm` | LVM-Thin | Dischi VM/LXC live | Disco sistema Proxmox |
| `backup_wd_4tb` | Directory | Backup vzdump | Disco 4TB locale (`/mnt/pve/backup_wd/dump`) |
| `nas_wd` | NFS/CIFS | Backup vzdump | NAS WD MyCloud 192.168.1.178 (`/mnt/pve/nas_wd/dump`) |

---

## Riferimenti esterni

| Risorsa | Link |
|---------|------|
| yoziru/esphome-tesla-ble | <https://github.com/yoziru/esphome-tesla-ble> |
| simple-pid (libreria Python) | <https://github.com/m-lundberg/simple-pid> |
| AppDaemon (add-on frenck) | <https://github.com/hassio-addons/addon-appdaemon> |
| pvlib-python | <https://pvlib-python.readthedocs.io/> |
| Open-Meteo API | <https://open-meteo.com/> |
| Home Assistant | <https://www.home-assistant.io/> |
| Proxmox VE | <https://www.proxmox.com/> |

---

## Pannello di emergenza

**URL:** <https://marco241254.github.io/casa-emergenza/indexnew.html>

Sito statico GitHub Pages con 18 moduli HTML per la risoluzione dei problemi domestici, consultabile da chiunque si trovi in casa anche senza conoscere il sistema.

### Moduli principali

| # | Modulo | Responsabile |
|---|--------|-------------|
| 1 | Riscaldamento con PDC | Elvis Martinat |
| 2 | Acqua Calda Sanitaria | Elvis Martinat |
| 3 | Acqua Fredda | Elvis Martinat |
| 4-7 | Impianto elettrico (cucina, piano 1°, mansarda, locale tecnico) | Claudio Massello |
| 8 | Blackout totale ENEL | Claudio Massello |
| 9 | Gas cucina | Elvis Martinat |
| 10 | Fotovoltaico | Alessandro Reale |
| 11 | Internet / WiFi | Marco |
| 12 | Contatti completi | Marco |

---

*Documento generato il 2026-06-23*
