# Home Assistant — Configurazione e Packages

**Indice documentazione** | [README](README.md) | [Infrastruttura](00_infrastruttura.md) | [PDC Hitachi](02_pdc_hitachi.md) | [Fotovoltaico](03_fotovoltaico.md) | [Tesla/Wallbox](04_tesla_wallbox.md) | [Energia](05_energia.md) | [Clima](06_clima_temperature.md) | [Operazioni](07_operazioni.md) | [Riferimenti](08_riferimenti.md)

---

**Aggiornato:** 2026-06-23

---

## Indice

- [Configurazione principale](#configurazione-principale)
- [Struttura file](#struttura-file)
- [Mappa packages (indice funzionale)](#mappa-packages-indice-funzionale)
- [Dettaglio packages](#dettaglio-packages)
- [Dipendenze critiche](#dipendenze-critiche)
- [Configurazione database](#configurazione-database)
- [Note tecniche](#note-tecniche)

---

## Configurazione principale

**File:** `configuration.yaml` (v1900)

```yaml
# Parametri base
homeassistant:
  name: Casa-GEEKOM
  latitude: 44.958349     # Pomaretto casa
  longitude: 7.182826     # Pomaretto casa
  elevation: 620          # metri s.l.m.
  unit_system: metric
  time_zone: Europe/Rome
  country: IT
  currency: EUR
```

> **⚠️ ATTENZIONE:** Se modifichi lat/lon, aggiorna ANCHE `packages/972_gg_forecast_24h.yaml` (Open-Meteo usa le stesse coordinate nel parametro resource).

### Frontend e inclusioni

```yaml
frontend:
  themes: !include_dir_merge_named themes

automation: !include automations.yaml
script:     !include scripts.yaml
scene:      !include scenes.yaml
```

### Packages
Tutti i package sono inclusi sotto la chiave `packages:` in `homeassistant:` (struttura package moderna).

---

## Struttura file

```
/config/
├── configuration.yaml          # Configurazione principale v1900
├── secrets.yaml                # Password, token, URL (NON condividere)
├── automations.yaml            # Automazioni HA
├── scripts.yaml                # Script HA
├── scenes.yaml                 # Scene HA
├── docs/                       # Questa documentazione
│   ├── README.md
│   ├── 00_infrastruttura.md
│   ├── 01_home_assistant.md    # Questo file
│   ├── 02_pdc_hitachi.md
│   ├── 03_fotovoltaico.md
│   ├── 04_tesla_wallbox.md
│   ├── 05_energia.md
│   ├── 06_clima_temperature.md
│   ├── 07_operazioni.md
│   └── 08_riferimenti.md
├── packages/                   # 40+ file YAML organizzati
│   ├── 2010_system_monitoring.yaml
│   ├── 210_tesla_solar_optima_v5_ble_diretto.yaml
│   ├── 212_tesla_pre_controller_energia.yaml
│   ├── 300_riduz_cons_energ.yaml
│   ├── 310_telecomandi_rodret.yaml
│   ├── 400_irrigazione_prati.yaml
│   ├── 710_setting_fv.yaml
│   ├── 720_prev_FV_new.yaml
│   ├── 810_norm_pot_elettriche.yaml
│   ├── ... (completo sotto)
├── themes/                     # Tema personalizzato
└── views/                      # File Lovelace (iniziano con `views:`)
```

---

## Mappa packages (indice funzionale)

| Area | Package | Documento dedicato |
|------|---------|-------------------|
| 🏠 Sistema & Monitoring | 2010 | [Infrastruttura](00_infrastruttura.md) |
| ⚡ Fotovoltaico — Previsioni | 710, 720 | [Fotovoltaico](03_fotovoltaico.md) |
| ⚡ Fotovoltaico — Monitoraggio | 810, 820, 821, 822, 823, 824, 825, 826, 827, 828, 829 | [Fotovoltaico](03_fotovoltaico.md) + [Energia](05_energia.md) |
| 🔋 Batterie | 829 | [Energia](05_energia.md) |
| 🔌 Carichi elettrici | 830, 831, 832, 833, 834, 835, 836, 837 | [Energia](05_energia.md) |
| 🌡️ Temperature & Clima | 925, 926, 927, 930 | [Clima](06_clima_temperature.md) |
| 🔥 PDC Hitachi Yutaki | 900, 902, 905, 907, 910, 920, 928, 935, 950, 960, 970, 971, 972, 973 | [PDC Hitachi](02_pdc_hitachi.md) |
| 🤖 Automazioni speciali | 210, 212, 300, 310, 400, 940 | [Tesla/Wallbox](04_tesla_wallbox.md) + questo file |

---

## Dettaglio packages

### 2010 — System Monitoring
**File:** `2010_system_monitoring.yaml`
**Scopo:** Monitora le risorse hardware del GEEKOM e delle VM Proxmox
**Input:** API Proxmox (host 192.168.1.100), sensori CPU/RAM/disco
**Output:** Sensori utilizzo CPU, memoria, disco per ogni VM/LXC
**Stato:** ✅ Aggiornato

---

### 210 — Tesla Solar Optima v5 (BLE diretto via ESP32)
**File:** `210_tesla_solar_optima_v5_ble_diretto.yaml` (versione interna v6.0)
**Scopo:** Controlla la carica Tesla per assorbimento ottimale dal surplus FV. Calcola ogni 30s gli ampere giusti, applica isteresi 60s per accensione, spegne immediato se surplus scende o cavo via. Tutto via BLE diretto attraverso ESP32 in garage (no Tesla API cloud).

**Input:**
- `sensor.pv_total_power_w`
- `sensor.pot_tot_l1l2l3_abc`
- `sensor.pot_carico_wallbox_totale_abc`
- `sensor.battery_available_energy_total_kwh`
- Entita' `tesla_ble_770224_*` (ESPHome)

**Output:**
- `sensor.disponibilita_solare_tesla`
- `sensor.tesla_ampere_target`
- `sensor.tesla_energia_mancante_kwh`
- `sensor.tesla_tempo_carica_ore`
- `binary_sensor.tesla_ble_pronta`

**Helper:** `input_boolean.tesla_solar_optima_abilitato`, `input_boolean.tesla_carica_subito`, `input_number.tesla_isteresi_secondi`, `input_datetime.tesla_partenza_domani`

**Hardware:** ESP32 esp32dev in garage, BLE verso Tesla (VIN XP7YGCES1RB482601)
**Sostituisce:** integrazione Tesla API (disabilitata per problema chiave developer)
**Stato:** ✅ Operativo  
**Documentazione completa:** [04_tesla_wallbox.md](04_tesla_wallbox.md)

---

### 212 — Tesla Pre-Controller Energia
**File:** `212_tesla_pre_controller_energia.yaml`
**Scopo:** Carica notturna Tesla con eventuale travaso di energia dalla batteria di casa alla Tesla
**Stato:** ✅ Operativo

---

### 300 — Riduzione Consumi Energetici
**File:** `300_riduz_cons_energ.yaml`
**Scopo:** Accende/spegne la pompa di circolazione termosifoni (R2) in AND logico con lo stato PDC. Post-run configurabile 15-60 min

**Input:** `sensor.shellyem_c45bbee1c642_channel_1_potenza` (PDC >200W), `sensor.pdc_r_opstate` (==6), `sensor.pdc_r_acsrun` (==0)
**Output:** `binary_sensor.pdc_riscaldamento_attivo`, `switch.shellyplus1_1c692008dea0` (R2), `timer.pompa_circolazione_post_run`
**Dipende da:** 900 (sensori Modbus PDC)
**Helper:** `input_boolean.pompa_circolazione_auto`, `input_number.pompa_post_run_minuti`
**Stato:** ✅ Aggiornato (migrato da platform:template 2026-05-15)

---

### 310 — Telecomandi Rodret
**File:** `310_telecomandi_rodret.yaml`
**Scopo:** Gestione telecomandi Zigbee Ikea Rodret
**Stato:** ⚠️ Non verificato

---

### 400 — Irrigazione Prati
**File:** `400_irrigazione_prati.yaml`
**Scopo:** Irrigazione prati — centralina Sonoff in locale tecnico/lavanderia
- Canal 1 = pompa
- Ch2 = prato sopra
- Ch3 = prato sotto
- Ch4 = prato piccolo
**Stato:** ✅ Operativo

---

### 710 — Setting FV
**File:** `710_setting_fv.yaml`
**Scopo:** Settaggio e taratura tra previsioni produzione FV e dati reali. Contiene input_number per calibrazione e sensori mediati (filtro mediano)
**Input:** Sensori FV SolarEdge + Enphase, Open-Meteo
**Output:** Parametri calibrazione, sensori normalizzati `pot_fv_*`
**Dipende da:** 820
**Stato:** ✅ Aggiornato

---

### 720 — Previsioni FV New
**File:** `720_prev_FV_new.yaml`
**Scopo:** Previsioni produzione fotovoltaica da Open-Meteo, visualizzazione su Lovelace
**Input:** API Open-Meteo, parametri impianto FV
**Output:** Sensori previsione oraria/giornaliera produzione FV
**Dipende da:** 710
**Stato:** ⚠️ Da verificare (errore `friendly_name` segnalato)

---

### 810 — Normalizzazione Potenze Elettriche
**File:** `810_norm_pot_elettriche.yaml`
**Scopo:** Normalizza e aggrega le potenze elettriche da tutti gli Shelly EM. Applica filtro mediano (5 campioni ~2.5 min) per eliminare picchi. Calcola diagnostica FV

**Input:** Shelly EM3 (contatore generale), Shelly EM (PDC, wallbox), SolarEdge, Enphase
**Output:**
- `sensor.pot_tot_l1l2l3_abc`
- `sensor.pot_fv_totale_mediano`
- `sensor.pot_fv_indiretta_mediano`
- `sensor.pot_fv_delta_indiretta_diretta`
- `sensor.pot_fv_delta_surplus_netto`
**Stato:** ✅ Aggiornato (migrato 2026-05-15)

---

### 820 — Normalizzazione Energie Elettriche
**File:** `820_norm_energie_elettriche.yaml`
**Scopo:** Aggrega le energie elettriche da tutti gli Shelly EM in sensori normalizzati
**Input:** Sensori energia Shelly EM/EM3
**Stato:** ⚠️ Non verificato

---

### 821 — Somma Produzione FV Enphase
**File:** `821_somma_prod_fv_enph.yaml`
**Scopo:** Somma produzione FV microinverter Enphase
**Input:** Sensori singoli microinverter Enphase
**Output:** `sensor.pot_fv_enphase_totale`
**Stato:** ⚠️ Non verificato

---

### 822 — Prelievo/Cessione ENEL
**File:** `822_prel_cess_enel.yaml`
**Scopo:** 8 contatori nativi HA (daily/weekly/monthly/yearly) per prelievo e cessione ENEL. 3 slider prezzi + memoria bimestrale
**Input:** `sensor.potenza_em3_*` (Shelly EM3 contatore generale)
**Output:** 27 sensori energia/costo prelievo e cessione
**Dipende da:** 810, 823
**Stato:** ⚠️ Non verificato

---

### 823 — Costo Reale Prelievo ENEL
**File:** `823_costo_reale_prel_enel.yaml`
**Scopo:** Costi reali sul reale scambio istantaneo trifase. Metodo ENEL bilanciamento tra le fasi, due integrali trapezi
**Input:** Potenze istantanee trifase Shelly EM3
**Output:** Sensori costo orario/giornaliero prelievo reale
**Dipende da:** 810
**Stato:** ⚠️ Non verificato

---

### 824 — Produzione FV Giornaliera
**File:** `824_produzione_fv_gg.yaml`
**Scopo:** Produzione FV giornaliera come deduzione: `autoconsumo + cessione ENEL`
**Input:** `sensor.energia_prodotta_fv`, sensori utility meter cessione/autoconsumo
**Output:** Sensori produzione FV giornaliera per SolarEdge e Enphase
**Dipende da:** 828, 822
**Stato:** ⚠️ Non verificato

---

### 825 — Produzione FV Periodi
**File:** `825_produzione_fv_periodi.yaml`
**Scopo:** Aggregazioni settimanale, mensile, bimestrale, annuale di produzione FV, autoconsumo, prelievo, cessione
**Input:** Sensori giornalieri di 824
**Output:** Sensori aggregati multi-periodo
**Dipende da:** 824
**Stato:** ⚠️ Non verificato

---

### 826 — Produzione FV Completo
**File:** `826_produzione_fv_completo.yaml`
**Scopo:** Vista completa produzione FV: giornaliero + tutti i periodi in un unico package
**Input:** Sensori 824, 825
**Output:** Dashboard-ready sensors produzione FV completa
**Dipende da:** 824, 825
**Stato:** ✅ Nel progetto

---

### 827 — Prelievo/Cessione Costi Reali
**File:** `827_prel_cess_enel_cost_reali.yaml`
**Scopo:** Prelievo/Cessione ENEL con metodo corretto (somma algebrica trifase)
**Input:** Potenze L1, L2, L3 da Shelly EM3
**Output:** Sensori prelievo/cessione reale per fascia oraria
**Dipende da:** 810, 823
**Stato:** ✅ Nel progetto

---

### 828 — Utility Meters FV
**File:** `828_utility_meters_fv.yaml`
**Scopo:** Utility meter per la produzione fotovoltaica totale (daily/weekly/monthly/yearly)
**Input:** `sensor.pv_total_power_w`
**Output:** `sensor.pv_total_power_w_energia_giorn/sett/mens/ann`
**Dipende da:** 810
**Stato:** ✅ Nel progetto

---

### 829 — Battery Integer SolarEdge
**File:** `829_battery_integer.yaml`
**Scopo:** Integrale di Riemann per la batteria SolarEdge separando potenze positive (carica) e negative (scarica) per generare energie
**Input:** `sensor.solaredge_battery_power`
**Output:** `sensor.energia_batteria_carica_*`, `sensor.energia_batteria_scarica_*`
**Stato:** ✅ Nel progetto

---

### 830–837 — Energie per Zona

| Package | Zona | Input | Stato |
|---------|------|-------|-------|
| 830 | Wallbox (2 colonnine trifase) | Integrazioni wallbox | ✅ |
| 831 | PDC | Shelly EM c45bbee1c642 ch1 | ✅ Aggiornato |
| 832 | Cucina | Shelly EM cucina | ✅ |
| 833 | Primo piano | Shelly EM primo piano | ✅ |
| 834 | Mansarda | Shelly EM mansarda | ✅ |
| 835 | Luci esterne | Shelly EM luci esterne | ✅ |
| 836 | Lavanderia | Shelly EM lavanderia | ✅ |
| 837 | Climatizzatori | Shelly EM clima | ✅ |

---

### 900 — Hitachi Control
**File:** `900_hitachi_control.yaml` (versione interna v16, aggiornata 2026-05-22)
**Scopo:** Controllo e diagnostica dati Modbus PDC Hitachi. Verifica allineamento setpoint, stato operativo, ACS, Water Mode, OTC, ECO. Contiene anche la definizione del blocco `modbus:` (hub `hitachi`, TCP 192.168.1.5:502)

**Input:** Registri Modbus Hitachi, `sensor.pdc_r_*`, `sensor.pdc_w_*`, `sensor.pdc_s_*`
**Output:** `sensor.pdc_sync`, `sensor.pdc_opstate`, `sensor.pdc_centralmode`, `sensor.pdc_otc`, `sensor.pdc_eco`, `sensor.pdc_compressore`, `sensor.pdc_pompa`, `sensor.pdc_defrost`, `sensor.pdc_allarme`, `sensor.pdc_acs_run`, `sensor.pdc_acs_setpoint`, `sensor.pdc_acs_demand`, `sensor.pdc_acs_integrita`, `sensor.pdc_watermode_ok`. Automation `pdc_invia_setpoint` su slider `input_number.pdc_setpoint_mandata`

**🔧 Fix 2026-05-22:** `addr1003 W-OTC` commento corretto da `(deve=0)` a `(deve=3 = Fix)`. Valore 3 attiva esplicitamente la temperatura fissa scritta su `addr1005`; valore 0 lasciava la PDC libera di tornare alla logica interna del PC-ARFH1E.
**Stato:** ✅ Aggiornato e operativo  
**Documentazione completa:** [02_pdc_hitachi.md](02_pdc_hitachi.md)

---

### 902 — Ottimizzazione PDC (Curva Climatica Inversa)
**File:** `902_pdc_ottimizzazione_v1.yaml`
**Scopo:** Curva climatica inversa — alza la temperatura di mandata acqua via Modbus in funzione del surplus FV. Controllore PI con integrale persistente
**Input:** `sensor.pot_fv_totale_l1l2l3`, `sensor.pot_tot_l1l2l3_abc`, `sensor.pot_batterie_totale`, `sensor.temp_media_zone_attive`, `sensor.pdc_r_tempext`
**Output:** `sensor.pdc_surplus_fv`, `sensor.pdc_bilancio_*`, `sensor.pdc_tbase`, `sensor.pdc_termine_p`, `sensor.pdc_delta_pi`, `sensor.pdc_setpoint_target`, `input_number.pdc_k1`, `input_number.pdc_integrale_i`
**Dipende da:** 810, 920, 925
**Stato:** ✅ Aggiornato (migrato 2026-05-15)

---

### 905 — Hitachi Yutaki (Correzione Portate)
**File:** `905_hitachi_yutaki.yaml`
**Scopo:** Corregge l'errore di portata dell'integrazione Aleppe per Hitachi Yutaki. Applica fattore di correzione 1.159 vs CONTECA
**Input:** Sensori portata dall'integrazione Aleppe
**Stato:** ✅ Nel progetto

---

### 907 — COP PDC Teorico
**File:** `907_cop_pdc_theo.yaml`
**Scopo:** Calcola il COP teorico della PDC Hitachi RASM-4VNE basato su dati di targa
**Input:** `sensor.temp_media_esterna`, `sensor.conteca_temperatura_mandata`, `sensor.conteca_temperatura_ritorno`, `sensor.conteca_potenza`, `sensor.shellyem3_485519dc8109_phase_a_potenza`
**Output:** `sensor.cop_predetto_pdc_hitachi_teorico`, `sensor.cop_reale_pdc`, `sensor.efficienza_pdc_vs_elettrico`, `sensor.risparmio_energetico_stimato`
**⚠️ Nota:** Formule per R410A, non R32 — revisione raccomandata con dati invernali
**Stato:** ✅ Nel progetto

---

### 910 — CONTECA Sensori
**File:** `910_conteca_sensori.yaml`
**Scopo:** Normalizza i sensori del contatore termico CONTECA (contatore di calore). Corregge scale, unita' e nomi
**Output:** `sensor.conteca_temperatura_mandata`, `sensor.conteca_temperatura_ritorno`, `sensor.conteca_portata`, `sensor.conteca_potenza`, `sensor.conteca_energia_*`
**Stato:** ✅ Aggiornato

---

### 920 — PDC Core
**File:** `920_pdc_core.yaml`
**Scopo:** Normalizza tutti i sensori raw Modbus della PDC Hitachi. Punto di accesso unico a tutti i dati PDC
**Input:** Registri Modbus Hitachi (TCP 192.168.1.5:502)
**Output:** `sensor.pdc_r_*` (read), `sensor.pdc_w_*` (write), `sensor.pdc_s_*` (status), `binary_sensor.pdc_modalita_riscaldamento`, `binary_sensor.pdc_acs_attiva`
**Nota:** Definizione del blocco `modbus:` e' in `900_hitachi_control.yaml` (non in configuration.yaml)
**Stato:** ✅ Aggiornato

---

### 925–973 — Clima e Ottimizzazione PDC

| Package | Scopo | Documento |
|---------|-------|-----------|
| 925 | Temperature ambienti + media pesata zone attive | [Clima](06_clima_temperature.md) |
| 926 | Temperatura esterna (media 3 fonti) | [Clima](06_clima_temperature.md) |
| 927 | Gradi giorno riscaldamento | [Clima](06_clima_temperature.md) |
| 928 | Lockout AND termostato + allungamento ciclo OFF | [PDC Hitachi](02_pdc_hitachi.md) |
| 930 | Termostato fasce orarie per zona | [Clima](06_clima_temperature.md) |
| 935 | Energia termica/elettrica PDC + COP reale | [PDC Hitachi](02_pdc_hitachi.md) |
| 940 | Eccedenza FV irrecuperabile + analisi ROI batterie | [Energia](05_energia.md) |
| 950 | Helper AI optimizer PDC (AppDaemon) | [PDC Hitachi](02_pdc_hitachi.md) |
| 960 | Costo riscaldamento vs prelievo ENEL | [Energia](05_energia.md) |
| 970 | Duty cycle PDC da fabbisogno termico | [PDC Hitachi](02_pdc_hitachi.md) |
| 971 | Misura duty cycle reale (cmd vs potenza Shelly) | [PDC Hitachi](02_pdc_hitachi.md) |
| 972 | GG previsti 24h (Open-Meteo) | [Clima](06_clima_temperature.md) |
| 973 | COP previsto 24h + fabbisogno elettrico | [Clima](06_clima_temperature.md) |

---

## Dipendenze critiche

```
Modbus (192.168.1.5) ──► 920_pdc_core ──► 900_hitachi_control
                                      ──► 902_pdc_ottimizzazione
                                      ──► 928_pdc_lockout_and
                                      ──► 935_energia_cop
                                      ──► 960_costo_riscaldamento

CONTECA ──────────────► 910_conteca_sensori ──► 935_energia_cop
                                           ──► 973_cop_forecast_24h

Shelly EM (PDC) ───────► 810_norm_pot_elettriche ──► 902_pdc_ottimizzazione
                       ► 831_energie_pdc          ──► 935_energia_cop
                       ► 300_riduz_cons_energ      ──► 928_pdc_lockout_and

Netatmo/Tado ──────────► 925_temp_ambienti ──► 930_termostato_fasce
                       ► 926_temp_esterna  ──► 927_gradi_giorno ──► 970/971/972

SolarEdge + Enphase ───► 820/821 ──► 810 ──► 822/823/824/825/826/827/828
                       ► 829_battery_integer

Open-Meteo ────────────► 720_prev_FV_new
                       ► 972_gg_forecast_24h ──► 973_cop_forecast_24h
```

---

## Configurazione database

### Recorder — MariaDB (LXC 200)

```yaml
recorder:
  db_url: !secret recorder_db_url
  commit_interval: 30
  auto_purge: true
  purge_keep_days: 10
```

- **Host:** 192.168.1.165
- **DB:** homeassistant
- **Utente:** ha
- **Stringa connessione:** in `secrets.yaml` → `recorder_db_url`

### InfluxDB (LXC 110)

```yaml
influxdb:
  include:
    domains:
      - sensor
      - binary_sensor
      - switch
      - climate
```

- **Host:** 192.168.1.48
- **Porta:** 8086
- **DB:** home_assistant
- **Autenticazione:** via UI HA (importata automaticamente)
- **Password:** in `secrets.yaml` → `influxdb_password`

---

## Note tecniche

### Regola di deploy

| Tipo file | Metodo |
|-----------|--------|
| YAML packages | File Editor HA |
| Script AppDaemon (.py) | Proxmox console → copy diretto |
| Script LXC 130/131 (.py) | `pct exec` con heredoc base64 |

### Convenzioni utility meter
- Chiave = `unique_id`
- **Nessun campo `name:`** — garantisce `entity_id` prevedibile

### File aggiornati il 2026-05-15 (migrazione platform:template)
- `900_hitachi_control.yaml`
- `902_pdc_ottimizzazione_v1.yaml`
- `810_norm_pot_elettriche.yaml`
- `300_riduz_cons_energ.yaml`
- `827_prel_cess_enel_cost_reali.yaml`
- `973_cop_forecast_24h.yaml`
- `907_cop_pdc_theo.yaml`

### File aggiornati il 2026-05-22
- `900_hitachi_control.yaml` → v16, fix `addr1003 W-OTC`, Modbus PDC riattivato

### File con problemi aperti
- `720_prev_FV_new.yaml` — `friendly_name` invalido (riga 16) + duplicato `pv_total_power_w`

---

*Documento generato il 2026-06-23*
