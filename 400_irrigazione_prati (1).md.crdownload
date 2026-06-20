# Sistema Irrigazione Prati — TERMO_OPTIMA

**Documento:** `irrigazione_prati.md`
**Versione:** 1.0
**Data:** 2026-06-20
**Autore:** Marco / TERMO_OPTIMA
**Package di riferimento:** `/config/packages/400_irrigazione_prati.yaml` v1.0.2
**Dashboard:** `views.yaml` → vista "Irrigazione" (`/30-interuttori-rele/irrigazione`)

---

## 1. Scopo

Sistema di irrigazione dei prati alimentato **esclusivamente da surplus
fotovoltaico**. La pompa è l'ultima priorità di carico: irriga solo quando
c'è produzione FV eccedente, mai prelevando da rete o da batterie.

Controllo manuale, una zona per volta, da dashboard Home Assistant o da
app eWeLink.

---

## 2. Architettura idraulica (catena dell'acqua)

```
 Roggia "Bealera di Rocca Pertusa"
        │  (presa: angolo ovest cancellata, presso confine casa Ribet)
        ▼
 [1] Valvola manuale di intercettazione
        │  (presso scalini Prato Sopra, semi-interrata, da tenere TUTTA APERTA)
        ▼
 [2] Vasca di sedimentazione
        │  (presso la casetta della pompa — decanta sabbia/detriti)
        ▼
 [3] Pompa di rilancio  —  centrifuga monofase 2 CV
        │
        ├──► Elettrovalvola 24V  ZONA SOPRA   (Prato Sopra)
        ├──► Elettrovalvola 24V  ZONA SOTTO   (Prato Sotto)
        └──► Elettrovalvola 24V  ZONA PICCOLO (Prato Piccolo)
```

### Componenti idraulici

| # | Componente | Descrizione | Note operative |
|---|------------|-------------|----------------|
| 1 | Presa roggia | Prelievo da "Bealera di Rocca Pertusa", angolo ovest cancellata, presso confine casa Ribet | — |
| 2 | Valvola manuale intercettazione | Semi-interrata, presso scalini Prato Sopra | **Deve restare tutta aperta** |
| 3 | Vasca di sedimentazione | Presso casetta pompa | Decantazione detriti |
| 4 | Pompa di rilancio | Centrifuga monofase **2 CV** | Carico induttivo |
| 5 | Elettrovalvole | 3 × 24 V, una per zona | SOPRA / SOTTO / PICCOLO |

---

## 3. Architettura elettrica / di comando (catena del segnale)

La logica di comando è disaccoppiata dalla potenza tramite teleruttore:
il modulo Sonoff commuta solo la **bobina** del teleruttore (pochi VA),
mentre il carico induttivo della pompa è gestito dai contatti di potenza
del teleruttore stesso.

```
 Home Assistant  ──┐
 (VM 100, .53)     │
                   ├─► SonoffLAN ─► Sonoff 4CH PRO R3 ─► contatti puliti
 App eWeLink ──────┘   (locale)      "Irrigatori"          │
                                      IP 192.168.1.71       │
                                      MAC C4:DD:57:31:F4:F0 │
                                                            │
   ┌────────────────────────────────────────────────────────┤
   │ CH1 ─► bobina TELERUTTORE ─► contatti potenza ─► POMPA 2CV (230V)
   │ CH2 ─► elettrovalvola 24V ZONA SOPRA
   │ CH3 ─► elettrovalvola 24V ZONA SOTTO
   │ CH4 ─► elettrovalvola 24V ZONA PICCOLO
   └────────────────────────────────────────────────────────
```

### Quadro "Centralina Irrigazione" (locale tecnico / lavanderia)

| Componente | Modello | Funzione |
|------------|---------|----------|
| Interruttore | BTicino FC810NC16 (C16 + diff.) | Protezione linea |
| Trasformatore | Vemer TMD 15/24 (230V → 12/24V) | Alimentazione elettrovalvole 24V *(da confermare come sorgente 24V)* |
| Relè potenza | 4 × BOMGI BCH8-20 (20A) | Etichettati POMPA G. / SOPRA / SOTTO / PICCO |
| Modulo smart | **Sonoff 4CH PRO R3** | Comando WiFi/locale dei 4 canali |

### Mappatura canali Sonoff → entità HA

| Canale | Entità Home Assistant | Friendly name | Carico comandato |
|--------|----------------------|---------------|------------------|
| CH1 | `switch.irrigatori_sonoff_100114bc35_1` | Pompa | Bobina teleruttore → pompa 2CV (~1200 W) |
| CH2 | `switch.irrigatori_sonoff_100114bc35_2` | Sopra | Elettrovalvola 24V zona SOPRA |
| CH3 | `switch.irrigatori_sonoff_100114bc35_3` | Sotto | Elettrovalvola 24V zona SOTTO |
| CH4 | `switch.irrigatori_sonoff_100114bc35_4` | Piccolo | Elettrovalvola 24V zona PICCOLO |

### Note tecniche elettriche

- Il Sonoff commuta la **bobina** del teleruttore, non la pompa direttamente.
  Carico sul relè Sonoff = pochi VA → nessun problema di portata.
- **Miglioria consigliata (non urgente):** diodo volano (freewheeling) o
  snubber RC sulla bobina del teleruttore, per smorzare il transitorio
  induttivo all'apertura e allungare la vita dei contatti.
- Integrazione HA via **SonoffLAN** (HACS): comunicazione **locale**, nessuna
  dipendenza dal cloud eWeLink. IP fisso 192.168.1.71 assegnato dal UDR7.

---

## 4. Logica di comando (firmware HA)

Tutta la logica è **event-driven** su `timer.finished`: nessun ciclo `while`,
nessun polling, nessuna ricorsione. Tre timer dedicati fanno da orologi; le
automazioni reagiscono solo quando un timer finisce.

### 4.1 Sequenza di AVVIO

Riferimento: `script.irr_sequenza_avvio`

```
 t+0   Guardie:  se LOCKOUT attivo        → stop
                 se surplus FV < soglia   → stop
 t+0   Tutti i canali OFF (stato pulito)
 t+0   Flag "ciclo attivo" → ON
 t+6   CH1 ON   → pompa gira a vuoto (valvole chiuse, acqua ferma)
 t+36  CH_zona ON → si apre la zona scelta → l'acqua fluisce
 t+36  Avvio timer durata zona (valore dallo slug "Durata minuti")
```

Lo sfasamento di 30 s tra pompa e valvola serve a mandare in pressione il
circuito con tutte le valvole chiuse prima di aprire la zona.

### 4.2 Sequenza di STOP

Riferimento: `script.irr_sequenza_stop`

```
 t+0   CH_zona OFF (tutte le valvole di zona chiuse)
 t+2   CH1 OFF (pompa spenta 2 s dopo la chiusura valvola)
 t+2   Ferma timer residui (durata zona, grazia FV)
 t+2   Flag "ciclo attivo" → OFF
 t+2   Flag "lockout" → ON  +  avvio timer lockout 300 s
```

La chiusura della valvola **prima** dello spegnimento pompa evita colpi
d'ariete: la pompa non si ferma mai contro una mandata aperta in pressione.

### 4.3 Guardia fotovoltaica (con isteresi)

```
 surplus = sensor.pv_total_power_w  −  sensor.pot_tot_l1l2l3_abc
```

| Parametro | Default | Slider dashboard |
|-----------|---------|------------------|
| Soglia surplus AVVIO | 1500 W | `input_number.irr_soglia_avvio_w` |
| Soglia surplus STOP | 800 W | `input_number.irr_soglia_stop_w` |
| Finestra di grazia | 5 min | `timer.irr_fv_grazia` |
| Lockout post-ciclo | 300 s | `timer.irr_lockout` |
| Anti-glitch misura | 10 s | (nelle automazioni) |

**Comportamento:**
- Avvio consentito **solo** se surplus > soglia AVVIO (1200 W pompa + margine).
- Durante l'irrigazione, se il surplus scende sotto la soglia STOP in modo
  **continuo per 5 minuti**, parte lo spegnimento automatico.
- Se entro i 5 minuti il surplus risale sopra la soglia AVVIO, il timer di
  grazia si **azzera** e l'irrigazione prosegue.
- L'isteresi a doppia soglia (1500 avvio / 800 stop) evita oscillazioni
  on/off attorno a un valore unico.

---

## 5. Mappa entità (riepilogo)

### Switch fisici (SonoffLAN)
- `switch.irrigatori_sonoff_100114bc35_1` … `_4` → CH1–CH4
- `device_tracker.esp_31f4f0` → connettività modulo (IP 192.168.1.71)

### Helper di controllo
| Entità | Tipo | Ruolo |
|--------|------|-------|
| `input_select.irr_zona_scelta` | select | Zona del ciclo (SOPRA/SOTTO/PICCOLO) |
| `input_number.irr_durata_minuti` | slider | Durata irrigazione |
| `input_number.irr_soglia_avvio_w` | box | Soglia surplus avvio |
| `input_number.irr_soglia_stop_w` | box | Soglia surplus stop |
| `input_boolean.irr_ciclo_attivo` | bool | Stato logico ciclo in corso |
| `input_boolean.irr_lockout` | bool | Stato logico lockout |

### Timer
| Entità | Durata | Ruolo |
|--------|--------|-------|
| `timer.irr_zona` | variabile | Durata irrigazione zona |
| `timer.irr_fv_grazia` | 5 min | Tolleranza surplus basso |
| `timer.irr_lockout` | 5 min | Blocco riavvio post-ciclo |

### Sensori derivati (template)
| Entità | Ruolo |
|--------|-------|
| `sensor.irrigazione_surplus_fv` | Surplus = FV − consumo casa |
| `sensor.irrigazione_stato` | PRONTO / IN IRRIGAZIONE / LOCKOUT |
| `binary_sensor.irrigazione_fv_sufficiente_avvio` | surplus > soglia avvio |
| `binary_sensor.irrigazione_fv_sotto_soglia_stop` | surplus < soglia stop |

### Sensori sorgente (esterni al package)
| Entità | Origine |
|--------|---------|
| `sensor.pv_total_power_w` | Produzione FV totale (package 826) |
| `sensor.pot_tot_l1l2l3_abc` | Consumo casa trifase (package 810) |

---

## 6. Automazioni (orchestrazione)

Riferimento: sezione `automation:` del package.

| ID | Trigger | Azione |
|----|---------|--------|
| `irr_auto_fine_durata` | `timer.irr_zona` finito | Esegue sequenza STOP |
| `irr_auto_fv_sotto_soglia` | surplus < stop per 10 s | Avvia timer grazia 5 min |
| `irr_auto_fv_recupero` | surplus > avvio per 10 s | Annulla timer grazia |
| `irr_auto_fv_grazia_scaduta` | `timer.irr_fv_grazia` finito | Esegue sequenza STOP |
| `irr_auto_fine_lockout` | `timer.irr_lockout` finito | Sblocca (lockout → OFF) |

---

## 7. File di riferimento

| File | Percorso | Contenuto |
|------|----------|-----------|
| Package logico | `/config/packages/400_irrigazione_prati.yaml` | Helper, timer, template, script, automation |
| Dashboard | `views.yaml` (raw config Lovelace) | Vista "Irrigazione" |
| Inclusione | `configuration.yaml` | `irrigazione_prati: !include packages/400_irrigazione_prati.yaml` |

---

## 8. Procedure operative

### Avvio manuale normale
1. Dashboard → vista **Irrigazione**
2. Seleziona **Zona da irrigare**
3. Regola **Durata (minuti)** con lo slider
4. Verifica che **Surplus FV** sia sopra la soglia di avvio (gauge verde)
5. Premi **AVVIA**

### Stop manuale
- Premi **STOP** in qualsiasi momento → chiusura valvola, pompa OFF dopo 2 s,
  lockout 300 s.

### Reset stato "appeso"
Se "Ciclo attivo" risulta ON ma i canali sono tutti OFF e i timer inattivi
(stato disallineato dopo un reload o interruzione anomala):
- premi **STOP** una volta per forzare lo stato pulito, attendi il lockout,
  poi riavvia.

---

## 9. Punti aperti / TODO

- [ ] **Confermare** che il 24V delle elettrovalvole provenga dal
      trasformatore TMD 15/24 del quadro.
- [ ] Notifiche Telegram (rimosse nella v1.0.2): reintegrare con
      `telegram_bot.send_message`, chat_id `920319768`.
- [ ] Valutare diodo volano / snubber RC sulla bobina del teleruttore.
- [ ] Eventuale scheduler orario (al momento solo avvio manuale).

---

*Fine documento — irrigazione_prati.md v1.0*
