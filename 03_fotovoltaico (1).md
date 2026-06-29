# Sistema Fotovoltaico e Previsioni

**Indice documentazione** | [README](README.md) | [Infrastruttura](00_infrastruttura.md) | [Home Assistant](01_home_assistant.md) | [PDC Hitachi](02_pdc_hitachi.md) | [Tesla/Wallbox](04_tesla_wallbox.md) | [Energia](05_energia.md) | [Clima](06_clima_temperature.md) | [Operazioni](07_operazioni.md) | [Riferimenti](08_riferimenti.md)

---

**Aggiornato:** 2026-06-23

---

## Indice

- [Impianto fotovoltaico](#impianto-fotovoltaico)
- [Sistema di accumulo batterie](#sistema-di-accumulo-batterie)
- [Monitoraggio rete](#monitoraggio-rete)
- [Previsioni FV — pvlib (LXC 130)](#previsioni-fv--pvlib-lxc-130)
- [Setting e blending (package 710)](#setting-e-blending-package-710)
- [Produzione e aggregazioni (packages 820-829)](#produzione-e-aggregazioni-packages-820-829)

---

## Impianto fotovoltaico

### Panoramica — 41 pannelli / 16.28 kWp / 13.09 kW inverter

| # | Sistema | Pannelli | Qtà | kWp | Inverter | Potenza inv. | Esposizione | Inclinazione | Azimut | Fase |
|---|---------|----------|-----|-----|----------|-------------|-------------|--------------|--------|------|
| 1 | SolarEdge SE6000H | SunPower SPR-MAX3-400 (400W) | 18 | 7.20 | SE6000H | 6.00 kW | Sud | 60.5° | 170° | L3 |
| 2 | Enphase IQ7A | SunPower P3-AC 375W | 7 | 2.63 | IQ7A | 2.03 kW | Est | 25° | 74° | L1 |
| 3 | Enphase IQ8AC | SunPower P7-dc 450W | 3 | 1.35 | IQ8AC | 1.08 kW | Est | 25° | 74° | — |
| 4 | Enphase IQ8AC | SunPower P7-dc 450W | 3 | 1.35 | IQ8AC | 1.08 kW | Est | 25° | 74° | — |
| 5 | Enphase IQ7A | SunPower P3-AC 375W | 10 | 3.75 | IQ7A | 2.90 kW | Ovest | 25° | 254° | L2 |
| **TOT** | | | **41** | **16.28** | | **13.09 kW** | | | | **3 fasi** |

### Correzioni ombreggiamento fisico

**SUD (staccionata pomeriggio):**

| Ora | Fattore |
|-----|---------|
| 14:00 | 0.90 |
| 15:00 | 0.75 |
| 16:00 | 0.64 |
| 17:00 | 0.33 |
| 18:00 | 0.20 |
| 19:00 | 0.05 |

**OVEST (montagna pomeriggio):**

| Ora | Fattore |
|-----|---------|
| 16:00 | 0.80 |
| 17:00 | 0.30 |
| 18:00 | 0.10 |
| 19:00 | 0.02 |

**Profilo orizzonte attuale (pvlib_ha2.py):**

| Azimuth | 0° | 45° | 90° | 135° | 180° | 225° | 240° | 260° | 280° | 315° | 360° |
|---------|----|-----|-----|------|------|------|------|------|------|------|------|
| Altitudine | 5° | 4° | 3° | 2° | 2° | 5° | 10° | 12° | 8° | 5° | 5° |

> **⚠️ Problema aperto:** Orizzonte OVEST aggressivo (12° a 260°, 8° a 280°) potrebbe tagliare troppo presto il sole la sera. Da valutare vs dati reali.

---

## Sistema di accumulo batterie

| Sistema | Tecnologia | Capacita' | Fase |
|---------|-----------|----------|------|
| SolarEdge (stringa Sud) | Li-Ion | 9.70 kWh | L3 |
| Enphase IQ Battery 5P ×1 (stringa Est) | Li-Ion | 5.00 kWh | L1 |
| Enphase IQ Battery 5P ×1 (stringa Ovest) | Li-Ion | 5.00 kWh | L2 |
| **Totale** | | **19.70 kWh** | |

### Soglia riserva

**6.5 kWh (33% di 19.7 kWh)** — sotto questa soglia il sistema Tesla Solar Optima sottrae 500 W dal surplus disponibile per la Tesla, preservando energia per i carichi domestici.

### Sensore principale

- `sensor.battery_available_energy_total_kwh` — energia totale disponibile (package 829)

---

## Monitoraggio rete

- **Shelly EM3** (MAC `485519dc8109`): misura scambio rete trifase
- **Sensore scambio totale:** `sensor.potenza_em3_485519dc8109_totale_abc` (negativo = esportazione)
- **Sensore surplus netto:** `sensor.pot_fv_delta_surplus_netto`
- **Formula produzione FV:** `produzione = consumo − prelievo + cessione` (package 825/826)

---

## Previsioni FV — pvlib (LXC 130)

### Infrastruttura

| Voce | Valore |
|------|--------|
| Container | LXC 130 su Proxmox |
| IP fisso | 192.168.1.68 |
| Path script | `/opt/meteo_corrector/pvlib_ha2.py` |
| Venv | `/opt/meteo_corrector/venv/bin/python3` |
| Coordinate | LAT=44.88, LON=7.33, ALT=626 |

### Script principali

| Script | Cron | Funzione |
|--------|------|----------|
| `pvlib_ha2.py` | ogni ora alle :00 | Calcola previsioni orarie per 4 stringhe, pubblica su HA via REST API |
| `correttore.py` | ogni ora alle :00 | Calcola fattore k reale/previsto, aggiorna media 14 giorni |

### pvlib_ha2.py — logica di calcolo

**Input:** Open-Meteo API (GHI, DNI, DHI, cloud_cover orari — 3 giorni)

**Per ogni array chiama** `pvlib.irradiance.get_total_irradiance()` poi applica:
1. **Orizzonte** (`HOR_AZ` / `HOR_ALT`): blocca il sole quando e' sotto la linea dell'orizzonte
2. **Cap potenza**: limita al massimo fisico dell'inverter
3. **Coefficienti ombra manuali** per SUD e OVEST

**Finestra oraria pubblicata:** ore 6–20 incluse

### Sensori pubblicati su HA

Per ogni array e per il totale (oggi e domani):
- `sensor.pvlib_{oggi|domani}_{array}_{HH}h_kw` — potenza oraria [kW]
- `sensor.pvlib_{oggi|domani}_{array}_kwh` — totale giornaliero [kWh]
- `sensor.pvlib_{oggi|domani}_{HH}h_kw` — totale orario tutti gli array [kW]
- `sensor.pvlib_{oggi|domani}_tot_kwh` — totale giornaliero tutti gli array [kWh]
- `sensor.pvlib_{oggi|domani}_est_kwh` — EST-A + EST-B [kWh]
- `sensor.pvlib_{oggi|domani}_enphase_kwh` — EST-A + EST-B + OVEST [kWh]

### correttore.py — logica

Ogni ora legge da HA:
- `sensor.pv_total_power_w_energia_giorn` — energia reale oggi [kWh]
- `sensor.pvlib_oggi_tot_kwh` — energia prevista oggi [kWh]

Calcola **k = reale / previsto**

**Regole:**
- Prima delle 21:00: NON pubblica k_oggi (parziale = falsamente basso)
- Dopo le 21:00: pubblica k_oggi clampato [0.55 – 1.10]
- Sempre: aggiorna k medio 14 giorni da InfluxDB

**Output:**
- `sensor.pvlib_k_oggi` — fattore k giornata corrente
- `sensor.pvlib_k_correzione` — media k ultimi 14 giorni

> **⚠️ Bug noto:** `correttore.py` usa `pubblica()` senza `state_class: measurement` → HA segnala "entita' senza classe di stato". Le statistiche a lungo termine non vengono generate. **Bassa priorita'**.

### Cosa NON toccare

- La struttura dei cron
- La logica di retry Open-Meteo (ben fatta)
- Il meccanismo k/correttore (funziona)
- I coefficienti SUD_OMBRA (sembrano calibrati su dati reali)

---

## Setting e blending (package 710)

**File:** `710_setting_fv.yaml`

Combina:
- **Solcast** per accuratezza previsionale
- **Forecast Solar** per rispettare le proporzioni di orientamento delle stringhe

**Sensori blended:**
- `sensor.casa_ftv_stringa_sud`
- `sensor.casa_ftv_stringhe_est_ovest`

---

## Produzione e aggregazioni (packages 820-829)

| Package | Scopo | Stato |
|---------|-------|-------|
| 820 | Normalizzazione trifase Shelly EM3 | ⚠️ Da verificare |
| 821 | Somma produzione FV Enphase | ⚠️ Da verificare |
| 822 | Calcolo costi ENEL (27 sensori) | ⚠️ Da verificare |
| 823 | Costo reale prelievo ENEL trifase | ⚠️ Da verificare |
| 824 | Produzione FV giornaliera | ⚠️ Da verificare |
| 825 | Produzione FV periodi (settimanale/mensile/bim/ann) | ⚠️ Da verificare |
| 826 | Produzione FV completa | ✅ Nel progetto |
| 827 | Prelievo/cessione costi reali (somma algebrica) | ✅ Nel progetto |
| 828 | Utility meter FV | ✅ Nel progetto |
| 829 | Integrale batteria SolarEdge (carica/scarica) | ✅ Nel progetto |

---

*Documento generato il 2026-06-23*
