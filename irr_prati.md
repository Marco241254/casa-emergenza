# Sistema Irrigazione Prati — TERMO_OPTIMA

**Documento:** `400_irrigazione_prati.md`
**Versione doc:** 1.1
**Data:** 2026-06-20
**Autore:** Marco / TERMO_OPTIMA
**Package di riferimento:** `/config/packages/400_irrigazione_prati.yaml` **v1.0.4**
**Dashboard:** `views.yaml` → vista "Irrigazione" (`/30-interuttori-rele/irrigazione`)

---

## 1. Scopo

Irrigazione dei prati alimentata **esclusivamente da surplus fotovoltaico**.
La pompa è l'ultima priorità di carico: parte solo con produzione FV
eccedente, mai da rete né da batterie. Controllo manuale, una zona per volta,
da dashboard Home Assistant o da app eWeLink.

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

| # | Componente | Descrizione | Note operative |
|---|------------|-------------|----------------|
| 1 | Presa roggia | "Bealera di Rocca Pertusa", angolo ovest cancellata, presso confine casa Ribet | — |
| 2 | Valvola manuale intercettazione | Semi-interrata, presso scalini Prato Sopra | **Deve restare tutta aperta** |
| 3 | Vasca di sedimentazione | Presso casetta pompa | Decantazione detriti |
| 4 | Pompa di rilancio | Centrifuga monofase **2 CV** | Carico induttivo |
| 5 | Elettrovalvole | 3 × 24 V, una per zona | SOPRA / SOTTO / PICCOLO |

---

## 3. Architettura elettrica / di comando (catena del segnale)

Logica di comando disaccoppiata dalla potenza tramite teleruttore: il modulo
Sonoff commuta solo la **bobina** del teleruttore (pochi VA); il carico
induttivo della pompa è gestito dai contatti di potenza del teleruttore.

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
| CH1 | `switch.irrigatori_sonoff_100114bc35_1` | Pompa | Bobina teleruttore → pompa 2CV (~1200 W elettrici) |
| CH2 | `switch.irrigatori_sonoff_100114bc35_2` | Sopra | Elettrovalvola 24V zona SOPRA |
| CH3 | `switch.irrigatori_sonoff_100114bc35_3` | Sotto | Elettrovalvola 24V zona SOTTO |
| CH4 | `switch.irrigatori_sonoff_100114bc35_4` | Piccolo | Elettrovalvola 24V zona PICCOLO |

### Note tecniche elettriche

- Il Sonoff commuta la **bobina** del teleruttore, non la pompa: carico sul
  relè Sonoff = pochi VA, nessun problema di portata.
- **Miglioria consigliata (non urgente):** diodo volano o snubber RC sulla
  bobina del teleruttore, per smorzare il transitorio induttivo all'apertura
  e allungare la vita dei contatti.
- Integrazione HA via **SonoffLAN** (HACS): comunicazione **locale**, nessuna
  dipendenza dal cloud eWeLink. IP fisso 192.168.1.71 (UDR7).

---

## 4. Logica di comando (firmware HA)

Architettura **event-driven** su `timer.finished`: nessun `while`, nessun
polling, nessuna ricorsione. Tre timer fanno da orologi; le automazioni
reagiscono solo quando un timer termina.

### 4.0 Invariante di sicurezza idraulica ⚠️ (vincolo di progetto, v1.0.4)

> **La pompa non deve MAI commutare contro un circuito di mandata chiuso.**

Una pompa centrifuga che parte o si arresta a valvole chiuse lavora a portata
nulla: la prevalenza sale al valore di shut-off e si genera sovrappressione /
colpo d'ariete sul tratto pompa→valvola. Da qui il vincolo sull'ordine delle
commutazioni:

```
 AVVIO :  VALVOLA apre  →  (+2 s)  →  POMPA parte     (mandata già aperta)
 STOP  :  POMPA spegne  →  (+2 s)  →  VALVOLA chiude  (mandata ancora aperta)
```

In entrambi i transitori la sezione di mandata è aperta nell'istante in cui
la pompa cambia stato. La sovrapposizione di 2 s è la guardia temporale che
garantisce l'invariante a fronte della latenza di attuazione di
valvola/teleruttore.

> **Nota storica:** fino alla v1.0.3 la sequenza era invertita (pompa→valvola
> in avvio, con 6 s a vuoto + 30 s di "pressurizzazione" a valvole chiuse).
> Quella logica violava l'invariante e creava la sovrappressione. Corretta in
> v1.0.4 e rimossi i tempi di adescamento.

### 4.1 Sequenza di AVVIO — `script.irr_sequenza_avvio`

```
 t+0   Guardie:  se LOCKOUT attivo        → notifica + stop
                 se surplus FV < soglia   → notifica + stop
 t+0   Tutti i canali OFF (stato pulito)
 t+0   Flag "ciclo attivo" → ON
 t+0   CH_zona ON  → APRE la valvola della zona scelta (mandata aperta)
 t+2   CH1 ON      → AVVIA la pompa su circuito già aperto
 t+2   Avvio timer durata zona (valore dallo slider "Durata minuti")
 t+2   Notifica Telegram di avvio
```

Durante i 2 s di sovrapposizione la valvola è aperta ma la pompa è ferma:
nessun flusso, nessun problema.

### 4.2 Sequenza di STOP — `script.irr_sequenza_stop`

```
 t+0   CH1 OFF      → SPEGNE la pompa (valvola ancora aperta)
 t+2   CH_zona OFF  → CHIUDE tutte le valvole di zona
 t+2   Ferma timer residui (durata zona, grazia FV)
 t+2   Flag "ciclo attivo" → OFF
 t+2   Flag "lockout" → ON  +  avvio timer lockout 300 s
```

I 2 s con pompa ferma e valvola ancora aperta scaricano l'eventuale
sovrapressione residua prima della chiusura.

### 4.3 Guardia fotovoltaica (con isteresi)

```
 surplus = sensor.pv_total_power_w  −  sensor.pot_tot_l1l2l3_abc
```

| Parametro | Default | Entità |
|-----------|---------|--------|
| Soglia surplus AVVIO | 1500 W | `input_number.irr_soglia_avvio_w` |
| Soglia surplus STOP | 800 W | `input_number.irr_soglia_stop_w` |
| Finestra di grazia | 5 min | `timer.irr_fv_grazia` |
| Lockout post-ciclo | 300 s | `timer.irr_lockout` |
| Anti-glitch misura | 10 s | (condizione `for:` nelle automazioni) |

**Comportamento:**
- Avvio consentito **solo** se surplus > soglia AVVIO (≈1200 W pompa + margine).
- Se durante l'irrigazione il surplus resta sotto la soglia STOP in modo
  **continuo per 5 min**, parte lo spegnimento automatico.
- Se entro i 5 min il surplus risale sopra la soglia AVVIO, il timer di grazia
  si **azzera** e l'irrigazione prosegue.
- Doppia soglia (1500 / 800 W) = isteresi che evita pendolamento on/off
  attorno a un valore unico. Architettura a banda morta tipica di un
  controllo bang-bang con isteresi.

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

| ID | Trigger | Azione |
|----|---------|--------|
| `irr_auto_fine_durata` | `timer.irr_zona` finito | Sequenza STOP + notifica |
| `irr_auto_fv_sotto_soglia` | surplus < stop per 10 s | Avvia timer grazia + notifica |
| `irr_auto_fv_recupero` | surplus > avvio per 10 s | Annulla timer grazia + notifica |
| `irr_auto_fv_grazia_scaduta` | `timer.irr_fv_grazia` finito | Sequenza STOP + notifica |
| `irr_auto_fine_lockout` | `timer.irr_lockout` finito | Sblocca (lockout → OFF) |

### Macchina a stati (sintesi)

```
   ┌─────────┐  AVVIA (guardie OK)   ┌────────────────┐
   │ PRONTO  │──────────────────────►│ IN IRRIGAZIONE │
   └─────────┘                       └────────────────┘
        ▲                               │   │   │
        │ fine lockout (300s)           │   │   │ durata raggiunta
        │                               │   │   │ / grazia FV scaduta
        │                               │   │   │ / STOP manuale
   ┌─────────┐◄──────────────────────────┘   │   │
   │ LOCKOUT │                                ▼   ▼
   └─────────┘                          (sequenza STOP → LOCKOUT)
```

Guardia FV interna a IN IRRIGAZIONE: sotto-soglia per 5 min continui →
transizione forzata a STOP; recupero entro i 5 min → permanenza.

---

## 7. File di riferimento

| File | Percorso | Contenuto |
|------|----------|-----------|
| Package logico | `/config/packages/400_irrigazione_prati.yaml` | Helper, timer, template, script, automation |
| Dashboard | `views.yaml` (raw config Lovelace) | Vista "Irrigazione" |
| Inclusione | `configuration.yaml` | `irrigazione_prati: !include packages/400_irrigazione_prati.yaml` |
| Documentazione | `400_irrigazione_prati.md` | Questo documento |

---

## 8. Procedure operative

### Avvio manuale normale
1. Dashboard → vista **Irrigazione**
2. Seleziona **Zona da irrigare**
3. Regola **Durata (minuti)** con lo slider
4. Verifica **Surplus FV** sopra la soglia di avvio (gauge in verde)
5. Premi **AVVIA**

### Stop manuale
Premi **STOP** in qualsiasi momento → pompa OFF, valvola chiusa dopo 2 s,
lockout 300 s.

### Reset stato "appeso"
Se "Ciclo attivo" = ON ma canali tutti OFF e timer inattivi (disallineamento
dopo reload o interruzione anomala): premi **STOP** una volta per forzare lo
stato pulito, attendi il lockout, poi riavvia.

---

## 9. Punti aperti / TODO

- [ ] **Confermare** che il 24V delle elettrovalvole provenga dal
      trasformatore TMD 15/24 del quadro.
- [x] ~~Notifiche Telegram~~ → reintegrate in v1.0.3, mantenute in v1.0.4
      (`telegram_bot.send_message`, chat_id `920319768`).
- [ ] Diodo volano / snubber RC sulla bobina del teleruttore.
- [ ] Scheduler orario (al momento solo avvio manuale).
- [ ] **Verificare in campo** la guardia temporale di 2 s sull'invariante
      idraulico: confrontarla con il tempo di attuazione reale
      dell'elettrovalvola (apertura) e del teleruttore (chiusura contatti).
      Se la valvola impiega >2 s ad aprire completamente, aumentare il delay
      di avvio per non far partire la pompa contro una sezione ancora parziale.

---

*Fine documento — 400_irrigazione_prati.md v1.1 (allineato a package v1.0.4)*



















