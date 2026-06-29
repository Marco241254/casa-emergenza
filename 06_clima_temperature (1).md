# Clima, Temperature e Forecast

**Indice documentazione** | [README](README.md) | [Infrastruttura](00_infrastruttura.md) | [Home Assistant](01_home_assistant.md) | [PDC Hitachi](02_pdc_hitachi.md) | [Fotovoltaico](03_fotovoltaico.md) | [Tesla/Wallbox](04_tesla_wallbox.md) | [Energia](05_energia.md) | [Operazioni](07_operazioni.md) | [Riferimenti](08_riferimenti.md)

---

**Aggiornato:** 2026-06-23

---

## Indice

- [Temperature ambienti](#temperature-ambienti)
- [Temperatura esterna](#temperatura-esterna)
- [Termostato fasce orarie](#termostato-fasce-orarie)
- [Gradi giorno](#gradi-giorno)
- [Forecast 24h](#forecast-24h)
- [COP forecast 24h](#cop-forecast-24h)
- [Duty cycle PDC](#duty-cycle-pdc)
- [Pompa circolazione](#pompa-circolazione)
- [Irrigazione](#irrigazione)

---

## Temperature ambienti

### Package 925 — Temperature Ambienti

**File:** `925_temp_ambienti.yaml`

Normalizza le temperature degli ambienti interni e calcola la **media pesata delle zone attive**.

**Input:** Sensori Netatmo/Tado per ogni zona  
**Output:**

| Sensore | Descrizione |
|---------|-------------|
| `sensor.temp_cucina` | Temperatura cucina |
| `sensor.temp_salone` | Temperatura salone |
| `sensor.temp_studio` | Temperatura studio |
| `sensor.temp_camera` | Temperatura camera letto |
| `sensor.temp_media_zone_attive` | Media pesata zone attive (input per PDC) |

**Dipende da:** Integrazioni Netatmo, Tado  
**Stato:** ✅ Aggiornato

---

## Temperatura esterna

### Package 926 — Temperatura Esterna

**File:** `926_temp_esterna.yaml`

Normalizza e fa la media delle temperature esterne da **tre fonti**:

1. **Netatmo lato PDC** — `sensor.netatmo_*_temperature`
2. **Netatmo Pomaretto ufficiale**
3. **Sensore PDC Modbus** — `sensor.pdc_r_tempext`

**Output:**
- `sensor.temp_media_esterna`
- `sensor.temp_esterna_netatmo_pdc`
- `sensor.temp_esterna_pomaretto`

**Dipende da:** Integrazione Netatmo, 920  
**Stato:** ✅ Nel progetto

---

## Termostato fasce orarie

### Package 930 — Termostato Fasce

**File:** `930_termostato_fasce.yaml`

Regolazione temperatura ambienti per fasce orarie:
- **Cucina** — setpoint giorno/notte/weekend
- **Salone** — setpoint giorno/notte/weekend
- **Studio** — setpoint giorno/notte/weekend
- **Camera letto** — setpoint giorno/notte/weekend

**Input:** `sensor.temp_*` zone, `input_number` setpoint per fascia  
**Output:** `climate.*` o comandi termostato per zona, `input_boolean` modalita'  
**Dipende da:** 925, integrazioni Tado/Netatmo  
**Stato:** ✅ Nel progetto

---

## Gradi giorno

### Package 927 — Gradi Giorno

**File:** `927_gradi_giorno.yaml`

Calcola i **Gradi Giorno (GG)** per il riscaldamento — indice energetico giornaliero.

**Input:**
- `sensor.temp_media_esterna` (da 926)
- `sensor.temp_media_zone_attive` (da 925)

**Output:**
- `sensor.gradi_giorno_oggi`
- `sensor.gradi_giorno_sett`
- `sensor.gradi_giorno_mens`
- Utility meter GG

**Dipende da:** 925, 926  
**Stato:** ✅ Nel progetto

---

## Forecast 24h

### Package 972 — GG Forecast 24h

**File:** `972_gg_forecast_24h.yaml`

Scarica temperature orarie da Open-Meteo (48h), simula curva T interna (campana asimmetrica), calcola GG previsti prossime 24h. Snapshot giornaliero alle 00:05.

**Input:** API Open-Meteo (lat 44.958373, lon 7.182863, Pinerolo)

**Output:**
- `sensor.t972_gg_forecast_24h`
- `sensor.t972_ext_forecast_now`
- `sensor.t972_ext_avg/min/max_24h`
- `sensor.t972_interna_simulata_now`
- `binary_sensor.t972_dati_freschi`
- `input_number.t972_gg_snapshot`

> **⚠️ ATTENZIONE:** Le coordinate qui (44.958373, 7.182863) devono corrispondere a quelle in `configuration.yaml` (44.958349, 7.182826). Se modifichi le coordinate in `configuration.yaml`, aggiorna ANCHE questo file.

**Dipende da:** API Open-Meteo (internet)  
**Stato:** ✅ Nel progetto

---

## COP forecast 24h

### Package 973 — COP Forecast 24h

**File:** `973_cop_forecast_24h.yaml`

Previsione COP sulle prossime 24h: COP medio, minimo, massimo, ora migliore/peggiore. Calcola fabbisogno energetico previsto.

**Input:**
- `sensor.t972_*` (da 972)
- `sensor.conteca_temperatura_mandata`
- `input_number.t972_gg_snapshot`

**Output:**
- `sensor.cop_forecast_medio_24h`
- `sensor.cop_forecast_min/max_24h`
- `sensor.fabbisogno_elettrico_previsto_24h`

**Dipende da:** 972, 907  
**Stato:** ✅ Nel progetto

---

## Duty cycle PDC

### Package 970 — PDC PWM Relay Step1

**File:** `970_pdc_pwm_relay_step1.yaml`

Definisce il duty cycle della PDC in relazione al fabbisogno termico giornaliero.

**Input:** `sensor.gradi_giorno_oggi` (da 927), `input_number` fabbisogno kWh/GG  
**Output:** `sensor.pdc_duty_cycle_target`, `input_number.pdc_toff_minuti`, timer duty cycle  
**Dipende da:** 927  
**Stato:** ✅ Nel progetto

### Package 971 — PDC Duty Measure

**File:** `971_pdc_duty_measure_cmd_vs_power.yaml`

Misura il duty cycle reale con due metodi:
1. **Comando** — timer ON/OFF
2. **Rilievo potenza** — compressore > soglia

**Input:** `sensor.shellyem_c45bbee1c642_channel_1_potenza`, comandi relay PDC  
**Output:** `sensor.pdc_duty_cmd`, `sensor.pdc_duty_power`, `sensor.pdc_duty_delta`  
**Dipende da:** 831, 920  
**Stato:** ✅ Nel progetto

---

## Pompa circolazione

### Package 300 — Riduzione Consumi

**File:** `300_riduz_cons_energ.yaml`

Accende/spegne la pompa di circolazione termosifoni (R2) in AND logico con stato PDC.

**Condizioni ON:**
- `sensor.shellyem_c45bbee1c642_channel_1_potenza` > 200W (PDC accesa)
- `sensor.pdc_r_opstate` == 6 (riscaldamento ON)
- `sensor.pdc_r_acsrun` == 0 (ACS non attiva)

**Output:**
- `binary_sensor.pdc_riscaldamento_attivo`
- `switch.shellyplus1_1c692008dea0` (R2 — circolatore)
- `timer.pompa_circolazione_post_run` (post-run 15-60 min)

**Helper:** `input_boolean.pompa_circolazione_auto`, `input_number.pompa_post_run_minuti`  
**Dipende da:** 900  
**Stato:** ✅ Aggiornato (migrato 2026-05-15)

---

## Irrigazione

### Package 400 — Irrigazione Prati

**File:** `400_irrigazione_prati.yaml`

Centralina Sonoff 4CH in locale tecnico/lavanderia:

| Canale | Funzione |
|--------|----------|
| CH1 | Pompa acqua |
| CH2 | Prato sopra |
| CH3 | Prato sotto |
| CH4 | Prato piccolo |

**Stato:** ✅ Operativo

---

*Documento generato il 2026-06-23*
