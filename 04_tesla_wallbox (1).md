# Tesla Model Y — Ricarica Solare e Wallbox

**Indice documentazione** | [README](README.md) | [Infrastruttura](00_infrastruttura.md) | [Home Assistant](01_home_assistant.md) | [PDC Hitachi](02_pdc_hitachi.md) | [Fotovoltaico](03_fotovoltaico.md) | [Energia](05_energia.md) | [Clima](06_clima_temperature.md) | [Operazioni](07_operazioni.md) | [Riferimenti](08_riferimenti.md)

---

**Aggiornato:** 2026-06-23

---

## Indice

- [Hardware](#hardware)
- [Architettura controllo](#architettura-controllo)
- [Catena di comunicazione](#catena-di-comunicazione)
- [Package 210 — logica di surplus](#package-210--logica-di-surplus)
- [Controllore PID (AppDaemon)](#controllore-pid-appdaemon)
- [Automazioni HA](#automazioni-ha)
- [Mutua esclusione Garage/Cancello](#mutua-esclusione-garagecancello)
- [Tuning PID](#tuning-pid)
- [Dashboard Lovelace](#dashboard-lovelace)
- [Deploy e manutenzione](#deploy-e-manutenzione)
- [Cronologia evoluzione](#cronologia-evoluzione)
- [Problemi aperti](#problemi-aperti)

---

## Hardware

| Componente | Modello/Specifica |
|------------|-----------------|
| **Auto** | Tesla Model Y Long Range |
| **Batteria** | 75 kWh nominali (60 kWh usabili nel calcolo) |
| **VIN** | XP7YGCES1RB482601 |
| **BLE MAC auto** | 40:79:12:8B:95:53 |
| **Wall Connector** | Tesla Wall Connector Gen 3 (TSN: PGT24118019155) |
| **Wallbox x2** | Due colonnine trifase, garage e cancello |

### ESP32 Garage

| Voce | Valore |
|------|--------|
| Board | esp32dev (variant esp32), 4 MB flash |
| IP / MAC WiFi | 192.168.1.175 / 88:57:21:77:02:24 |
| device_name | `tesla-ble` |
| Prefisso entita' | `tesla_ble_770224_*` |
| Firmware | ESPHome + `yoziru/tesla_ble_vehicle` (main) |

### ESP32 Cancello

| Voce | Valore |
|------|--------|
| Board | esp32dev (variant esp32), 4 MB flash |
| IP / MAC WiFi | 192.168.1.176 / 88:57:21:6D:AA:B0 |
| device_name | `tesla-ble-cancello` |
| Prefisso entita' | `tesla_ble_cancello_6daab0_*` |

> **⚠️ Nota:** Il prefisso del cancello include la parola "cancello": `tesla_ble_cancello_6daab0_*`, non `tesla_ble_6daab0_*`.

### Alimentazione e fail-safe

```
Rete 220V → Alimentatore 5V DIN → Interruttore fisico → ESP32
```

L'interruttore fisico e' il fail-safe: **spento → BLE giu' → Tesla manuale**.

---

## Architettura controllo

La corrente di ricarica e' modulata sul surplus fotovoltaico tramite un **controllore PID** (Python, libreria `simple-pid`) che gira in AppDaemon.

```
FV - consumi casa + wallbox
        ↓
disponibilita solare Tesla
        ↓
ampere disponibili = disponibilita / 711
        ↓
AppDaemon: tesla_pid_garage.py (simple-pid)
        ↓
number.tesla_ble_770224_charging_amps
        ↓
ESP32 garage via BLE
        ↓
Tesla
```

**Conversione:** 1 A trifase ≈ 711 W (3 fasi × 237 V)

**Schema PID:**
```
setpoint = sensor.disponibilita_solare_tesla_avg30 / 711   (A)
misura   = sensor.corrente_wallbox_shelly_media            (A)
errore   = setpoint - misura          (lo calcola simple-pid)
uscita   = PID(misura), clamp [2,14]
         -> number.tesla_ble_770224_charging_amps
```

Il termine **integrale** rigetta i disturbi (nuvole, carichi) e converge senza oscillare.

---

## Catena di comunicazione

```
Tesla (BLE) ⇄ ESP32 tesla_ble_770224 → WiFi → HA
                                              ↕
                    /config/packages/210_tesla_solar_optima.yaml
                                              ↕
                    AppDaemon: tesla_pid_garage.py
                                              ↕
                    number.tesla_ble_770224_charging_amps / switch.tesla_ble_770224_charger
```

---

## Package 210 — logica di surplus

**File:** `210_tesla_solar_optima_v5_ble_diretto.yaml` (versione interna v6.0, package 210 v10 + PID)

Il package gestisce tutta la logica decisionale. **Non modula direttamente la wallbox**: calcola il surplus e lo pubblica; AppDaemon legge il sensore e scrive la corrente.

### Sensori e helper prodotti

| Entita' | Tipo | Descrizione |
|---------|------|-------------|
| `sensor.disponibilita_solare_tesla` | Template | Surplus FV per Tesla. Se "carica subito" ON → 8000 W fissi. Altrimenti: `pv − (consumi − wallbox)` |
| `sensor.tesla_surplus_fv_reale` | Template | Surplus FV reale, mai sostituito dal finto. Usato per wake automation |
| `sensor.tesla_energia_mancante_kwh` | Template | `(target% − soc%) × 60 / 100` |
| `sensor.tesla_tempo_carica_ore` | Template | `(kWh_mancanti × 1000) / (3 × 230 × A)`, arrotondato al ceil |
| `input_boolean.tesla_carica_subito` | Helper | ON = forza 8000 W finti → wallbox al massimo |
| `input_boolean.tesla_wake_auto_abilitato` | Helper | ON = wake automatico da surplus FV |
| `input_number.tesla_isteresi_secondi` | Helper | Default 60s — anti-oscillazione |
| `input_datetime.tesla_partenza_domani` | Helper | Ora partenza domani, input per TERMO_ALLOCATOR |

### Sorgenti dati lette

| Sensore | Fonte | Descrizione |
|---------|-------|-------------|
| `sensor.pv_total_power_w` | Package 810 | Produzione FV totale istantanea |
| `sensor.pot_tot_l1l2l3_abc` | Package 810 | Consumo casa totale |
| `sensor.pot_carico_wallbox_totale_abc` | Shelly EM wallbox | Potenza wallbox |
| `sensor.battery_available_energy_total_kwh` | Package 829 | Energia batterie casa |
| `sensor.marcauto_battery_level` | Integrazione Tesla | SOC Tesla (%) |
| `sensor.marcauto_battery_range` | Integrazione Tesla | Autonomia (km) |
| `number.marcauto_charge_limit` | Integrazione Tesla | SOC target |
| `number.marcauto_charge_current` | Integrazione Tesla | Corrente impostata (A) |
| `switch.marcauto_charge` | Integrazione Tesla | ON/OFF carica |
| `binary_sensor.tesla_wall_connector_veicolo_connesso` | Wall Connector | Cavo collegato |

### Due sensori di disponibilita' (v10)

| Sensore | Sorgente | Uso |
|---------|----------|-----|
| `sensor.tesla_ampere_disponibili` | surplus **grezzo** / 711 | soglie on/off (reattivo) |
| `sensor.tesla_ampere_disponibili_avg` | surplus **medio 30s** / 711 | riferimento PID (stabile) |

> Razionale: per decidere on/off serve reattivita' (grezzo). Per modulare serve stabilita' (media 30s).

---

## Controllore PID (AppDaemon)

**File:** `tesla_pid_garage.py`
**Libreria:** `simple-pid` (installata via `python_packages` in configurazione AppDaemon)

### Parametri attuali (v2.2)

| Parametro | Valore | Significato |
|---|---|---|
| KP | 0.35 | Guadagno proporzionale |
| KI | 0.0025 | Guadagno integrale (molto basso) |
| KD | 0.0 | Derivata azzerata |
| LOOP_SEC | 10 | Ciclo PID ogni 10 secondi |
| MIN_WRITE_INTERVAL_SEC | 60 | Max 1 comando/minuto alla Tesla |
| MAX_STEP_PER_WRITE_A | 1 | Max ±1 A per comando |
| ERROR_DEADBAND_A | 0.45 | Sotto questo errore non scrive |
| AMP_MIN / AMP_MAX | 2 / 14 | Limiti uscita + anti-windup |

**Comportamento:** il regolatore calcola ogni 10s, la Tesla riceve al massimo un comando ogni 60s, ogni comando cambia al massimo di ±1 A, se l'errore e' piccolo non scrive.

### Anti-windup

`output_limits = (2, 14)` blocca anche l'accumulo dell'integrale. Risolve il difetto della versione precedente (errore minimo → salto grande).

### Avvio bumpless

All'accensione carica, il PID riparte dalla corrente reale assorbita (`charger_current`), senza scossoni.

### Configurazione AppDaemon (apps.yaml)

```yaml
pdc_optimizer:
  module: pdc_optimizer
  class: PDCOptimizer

tesla_pid_garage:
  module: tesla_pid_garage
  class: TeslaPidGarage
```

**Init command** (copia solo i file attivi):
```bash
mkdir -p /config/apps && cp /homeassistant/apps/apps.yaml /config/apps/apps.yaml && cp /homeassistant/apps/pdc_optimizer.py /config/apps/pdc_optimizer.py && cp /homeassistant/apps/tesla_pid_garage.py /config/apps/tesla_pid_garage.py && rm -f /config/apps/tesla_safety_loop.py
```

> **⚠️ Regola d'oro:** un solo punto deve scrivere su `number.tesla_ble_770224_charging_amps`. Il `tesla_safety_loop.py` e' stato **disattivato** perche' puntava a entita' vecchie e non comandava realmente la Tesla.

---

## Automazioni HA

La modulazione NON e' un'automazione: la fa il PID. In HA restano tre automazioni:

| Automazione | Trigger | Azione |
|---|---|---|
| **Accensione** | `tesla_ampere_disponibili` > 1 A per 60s + master/cavo/BLE OK | wake → delay 5s → switch ON (poi modula il PID) |
| **Spegnimento** | cavo → off (istantaneo) **oppure** `tesla_ampere_disponibili` < 2 A per 60s | switch OFF |
| **Allarme tracking** | `tesla_err_diff_avg5min` > 2 A per 5 min, carica attiva | Notifica Telegram |

**Calcolo errore tracking:** `tesla_err_diff = number.charging_amps − corrente_wallbox_shelly_media`

> **⚠️ Nota:** `sensor.tesla_err_diff` NON deve avere `device_class` ne' `availability` booleano: HA bloccherebbe silenziosamente il template.

### Automation wake da surplus

Risolve il problema del sleep mode: quando la Tesla e' addormentata, AppDaemon non riesce a comunicare via BLE.

**Trigger:** `sensor.tesla_surplus_fv_reale` ≥ 1500 W per ≥ 2 minuti continui

**Condizioni (tutte devono essere vere):**
1. `input_boolean.tesla_wake_auto_abilitato` = ON
2. Orario tra 07:30 e 12:30
3. Cavo collegato (`binary_sensor.tesla_wall_connector_veicolo_connesso` = ON)
4. SOC Tesla < charge_limit (c'e' da caricare)
5. Tesla non gia' in carica
6. Anti-rimbalzo: ultimo trigger > 30 minuti fa

**Azione:** Premi `button.marcauto_wake` + notifica persistente HA con orario e surplus.

---

## Mutua esclusione Garage/Cancello

L'auto e' in una sola posizione alla volta.

| Posizione | Operazione |
|---|---|
| **Garage** | `input_boolean.tesla_solar_optima_abilitato` ON; cancello OFF |
| **Cancello** | garage OFF; `..._cancello_abilitato` ON |

- Entrambi accesi: solo l'ESP vicino all'auto comanda (l'altro ha `tesla_ble_pronta` off)
- Entrambi spenti: Tesla manuale

---

## Tuning PID

1. **Kp, Ki, Kd = 0** → comportamento on/off, osserva overshoot
2. **Alza Kp** finche' oscilla, poi dimezza
3. **Alza Ki** per azzerare l'errore a regime
4. **Kd** solo se serve smorzare. Partenza: Kp=1.0, Ki=0.05, Kd=0

**Frase guida del sistema:**

> *"Il vero malato non e' il PID. E' il riferimento sporco."*
>
> Il PID v2.2 calma l'uscita. La v2.3 deve pulire l'ingresso.

### Prossima versione (v2.3) — filtro ingresso

La v2.3 dovrebbe introdurre:
1. Filtro lento su FV e consumo casa (120–180s)
2. Filtro rapido opzionale per crolli FV reali (20–30s)
3. Comando Tesla a scalini interi, max ±1 A per ciclo
4. Nessun inseguimento dei carichi impulsivi (ferro da stiro, phon, forno)
5. Discesa rapida solo se filtro rapido conferma vero crollo FV
6. Timer 60s ridotto a protezione tecnica BLE (10–15s minimi)

---

## Dashboard Lovelace

View garage (`type: sections`). Entita' chiave:

| Tile | Entita' |
|------|---------|
| Ampere disponibili (grezzo, on/off) | `sensor.tesla_ampere_disponibili` |
| Ampere disponibili (media, setpoint PID) | `sensor.tesla_ampere_disponibili_avg` |
| Ampere comandati (output PID) | `number.tesla_ble_770224_charging_amps` |
| Tuning PID | `input_number.tesla_pid_p / _i / _d` |
| Corrente reale Shelly | `sensor.corrente_wallbox_shelly_media` |
| Corrente reale BLE | `sensor.tesla_ble_770224_charger_current` |
| Toggle "carica subito" | `input_boolean.tesla_carica_subito` |
| Toggle "wake auto" | `input_boolean.tesla_wake_auto_abilitato` |
| kWh mancanti | `sensor.tesla_energia_mancante_kwh` |
| Tempo carica stimato | `sensor.tesla_tempo_carica_ore` |
| Gauge SOC Tesla | `sensor.marcauto_battery_level` |
| Gauge scambio ENEL | `sensor.potenza_em3_485519dc8109_totale_abc` |

> **⚠️** Un file di vista Lovelace inizia sempre con `views:`. Versionare prima di ogni modifica.

---

## Deploy e manutenzione

### Package 210

File Editor → `/config/packages/210_tesla_solar_optima_v5_ble_diretto.yaml` → riscrittura completa → Check Configuration → Reload All YAML.

### PID AppDaemon

1. Add-on AppDaemon → Configuration → `python_packages`: aggiungere `simple-pid`. Salvare, riavviare l'add-on
2. Posizionare `tesla_pid_garage.py` nella stessa cartella di `pdc_optimizer.py`
3. `apps.yaml` come mostrato sopra

> **⚠️ "Module not found" in AppDaemon:** Dall'add-on frenck recente, la directory delle app e' sotto `/addon_configs/a0d7b954_appdaemon/`. Controllare il log AppDaemon per l'`Import path` esatto.

---

## Cronologia evoluzione

| Fase | Approccio | Esito |
|---|---|---|
| v6–v7 | Modulazione su integrale 30s via automazione | Funzionante ma con salti |
| v8 | Template `tesla_charging_amps_final` con rampa ±2A sul feedback | Oscillava |
| v9 | Stessa logica, on/off su media | On/off molle |
| **v10 + PID** | PID vero in AppDaemon (simple-pid), on/off su grezzo | **Architettura attuale** |

### Entita' rimosse (non esistono piu')

`sensor.tesla_charging_amps_final`, `sensor.tesla_raw_amp_target`, `sensor.tesla_ampere_target`, `sensor.tesla_ampere_target_integrato` — eliminare eventuali entita' orfane dal registro HA.

---

## Problemi aperti

| Problema | Priorita' | Note |
|---|---|---|
| Allineare package 211 (cancello) all'architettura PID | Media | Serve `tesla_pid_cancello.py` gemello |
| Uniformare capacita' Tesla: 55 vs 60 kWh | Bassa | Package vs allocator |
| `tesla_safety_loop.py` usa nomi entita' vecchi | Bassa | Aggiornare a `tesla_ble_770224_*` |
| Valutare termine **feed-forward** | Bassa | Disponibilita' grezza per migliorare reattivita' PID |

---

*Documento generato il 2026-06-23 — Integra TESLA_SOLAR_OPTIMA v4.0 e promemoria PID v2.3*
