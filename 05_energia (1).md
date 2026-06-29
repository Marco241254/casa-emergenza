# Monitoraggio Energetico e Costi

**Indice documentazione** | [README](README.md) | [Infrastruttura](00_infrastruttura.md) | [Home Assistant](01_home_assistant.md) | [PDC Hitachi](02_pdc_hitachi.md) | [Fotovoltaico](03_fotovoltaico.md) | [Tesla/Wallbox](04_tesla_wallbox.md) | [Clima](06_clima_temperature.md) | [Operazioni](07_operazioni.md) | [Riferimenti](08_riferimenti.md)

---

**Aggiornato:** 2026-06-23

---

## Indice

- [Monitoraggio potenze elettriche](#monitoraggio-potenze-elettriche)
- [Monitoraggio energie per zona](#monitoraggio-energie-per-zona)
- [Costi ENEL](#costi-enel)
- [Batterie — integrale carica/scarica](#batterie--integrale-caricascarica)
- [COP reale PDC](#cop-reale-pdc)
- [Costo riscaldamento](#costo-riscaldamento)
- [Eccedenza FV e analisi ROI](#eccedenza-fv-e-analisi-roi)
- [Pre-controller notturno Tesla (package 212)](#pre-controller-notturno-tesla-package-212)

---

## Monitoraggio potenze elettriche

### Package 810 — Normalizzazione Potenze

**File:** `810_norm_pot_elettriche.yaml`

Normalizza e aggrega le potenze da tutti gli Shelly EM. Applica **filtro mediano (5 campioni ~2.5 min)** per eliminare picchi.

**Input:**
- Shelly EM3 (contatore generale, MAC 485519dc8109)
- Shelly EM (PDC, wallbox)
- SolarEdge
- Enphase

**Output principali:**
| Sensore | Descrizione |
|---------|-------------|
| `sensor.pot_tot_l1l2l3_abc` | Consumo casa totale trifase |
| `sensor.pot_fv_totale_mediano` | Produzione FV totale filtrata |
| `sensor.pot_fv_indiretta_mediano` | Produzione FV indiretta (da inverter) |
| `sensor.pot_fv_delta_indiretta_diretta` | Delta tra misura indiretta e diretta |
| `sensor.pot_fv_delta_surplus_netto` | Surplus FV netto disponibile |

**Stato:** ✅ Aggiornato (migrato 2026-05-15)

---

## Monitoraggio energie per zona

Ogni zona ha 5 utility meter (giornaliero, settimanale, mensile, bimestrale, annuale):

| Package | Zona | Input | Stato |
|---------|------|-------|-------|
| 830 | Wallbox (2 colonnine trifase) | Integrazioni wallbox | ✅ |
| 831 | PDC | Shelly EM c45bbee1c642 ch1 | ✅ |
| 832 | Cucina | Shelly EM cucina | ✅ |
| 833 | Primo piano | Shelly EM primo piano | ✅ |
| 834 | Mansarda | Shelly EM mansarda | ✅ |
| 835 | Luci esterne | Shelly EM luci esterne | ✅ |
| 836 | Lavanderia | Shelly EM lavanderia | ✅ |
| 837 | Climatizzatori | Shelly EM clima | ✅ |

**Convenzione nomi:** `sensor.energia_{zona}_{periodo}` dove periodo = `giorn/sett/mens/bim/ann`

---

## Costi ENEL

### Package 822 — Prelievo/Cessione

**File:** `822_prel_cess_enel.yaml`

8 contatori nativi HA (daily/weekly/monthly/yearly) per prelievo e cessione ENEL. 3 slider prezzi + memoria bimestrale.

**Input:** `sensor.potenza_em3_*` (Shelly EM3 contatore generale)  
**Output:** 27 sensori energia/costo prelievo e cessione  
**Dipende da:** 810, 823  
**Stato:** ⚠️ Non verificato

### Package 823 — Costo Reale Prelievo

**File:** `823_costo_reale_prel_enel.yaml`

Costi reali sul reale scambio istantaneo trifase. Metodo ENEL bilanciamento tra le fasi, due integrali trapezi.

**Input:** Potenze istantanee trifase Shelly EM3  
**Dipende da:** 810  
**Stato:** ⚠️ Non verificato

### Package 827 — Costi Reali (somma algebrica)

**File:** `827_prel_cess_enel_cost_reali.yaml`

Metodo corretto: somma algebrica trifase.

**Input:** Potenze L1, L2, L3 da Shelly EM3  
**Output:** Sensori prelievo/cessione reale per fascia oraria  
**Stato:** ✅ Nel progetto

---

## Batterie — integrale carica/scarica

### Package 829 — Battery Integer

**File:** `829_battery_integer.yaml`

Integrale di Riemann per la batteria SolarEdge separando potenze positive (carica) e negative (scarica).

**Input:** `sensor.solaredge_battery_power`  
**Output:**
- `sensor.energia_batteria_carica_*`
- `sensor.energia_batteria_scarica_*`

**Stato:** ✅ Nel progetto

---

## COP reale PDC

### Package 935 — Energia e COP

**File:** `935_energia_cop.yaml`

Traccia energia termica ed elettrica della PDC separando riscaldamento da ACS.

**Formula COP:**
```
COP_reale = Potenza_termica_CONTECA [kW] / Potenza_elettrica_PDC [kW]
```

**Input:**
- `sensor.conteca_*` (contatore termico)
- `sensor.shellyem_c45bbee1c642_channel_1_*` (Shelly EM PDC)
- `binary_sensor.pdc_modalita_riscaldamento`

**Output:**
- `sensor.energia_termica_risc_filtrata_n`
- `sensor.energia_elettrica_pdc_risc_filtrata_n`
- `sensor.cop_pdc_*` (COP globale e filtrato)
- Utility meter giornalieri/settimanali/mensili/annuali

**Dipende da:** 910, 920, 831  
**Stato:** ✅ Nel progetto

---

## Costo riscaldamento

### Package 960 — Costo Riscaldamento

**File:** `960_costo_riscaldamento.yaml`

Calcola il costo del riscaldamento PDC rispetto al prelievo da ENEL. Split costi riscaldamento vs altri carichi.

**Input:** `sensor.energia_pdc_*`, `sensor.prel_cess_enel_*`, prezzi energia  
**Output:** Sensori costo riscaldamento per periodo, percentuale sul totale  
**Dipende da:** 831, 827, 822  
**Stato:** ✅ Nel progetto

---

## Eccedenza FV e analisi ROI

### Package 940 — Eccedenza Analyzer

**File:** `940_eccedenza_analyzer.yaml`

Calcola quanta energia FV viene ceduta inutilmente alla rete e quanta potrebbe essere recuperata con batterie aggiuntive. Analisi ROI batterie DIY.

**Input:**
- `sensor.potenza_em3_485519dc8109_totale_abc`
- `sensor.pv_total_power_w`

**Output:**
- `sensor.eccedenza_irrecuperabile_w/kwh`
- `sensor.prelievo_gap_solare_w/kwh`
- `sensor.sistema_saturo_flag`
- Utility meter

**Helper:**
- `input_number.ea_costo_kwh_rete`
- `input_number.ea_costo_kwh_batterie_diy`
- `input_number.ea_capacita_target_kwh`

**Stato:** ✅ Aggiornato

---

## Pre-controller notturno Tesla (package 212)

**File:** `212_tesla_pre_controller_energia.yaml`

Gestisce la carica notturna della Tesla con eventuale travaso di energia dalla batteria di casa alla Tesla.

**Stato:** ✅ Operativo

---

*Documento generato il 2026-06-23*
