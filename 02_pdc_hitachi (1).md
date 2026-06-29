# Pompa di Calore Hitachi YUTAKI — Controllo e Documentazione

**Indice documentazione** | [README](README.md) | [Infrastruttura](00_infrastruttura.md) | [Home Assistant](01_home_assistant.md) | [Fotovoltaico](03_fotovoltaico.md) | [Tesla/Wallbox](04_tesla_wallbox.md) | [Energia](05_energia.md) | [Clima](06_clima_temperature.md) | [Operazioni](07_operazioni.md) | [Riferimenti](08_riferimenti.md)

---

**Aggiornato:** 2026-06-23

---

## Indice

- [Panoramica](#panoramica)
- [Specifiche PDC](#specifiche-pdc)
- [Architettura controllo](#architettura-controllo)
- [Gateway ATW-MBS-02](#gateway-atw-mbs-02)
- [Configurazione obbligatoria PC-ARFH1E](#configurazione-obbligatoria-pc-arfh1e)
- [Registri Modbus — Tavola completa](#registri-modbus--tavola-completa)
- [Correzione critica addr1003](#correzione-critica-addr1003)
- [Sequenza di avvio operativa](#sequenza-di-avvio-operativa)
- [Verifiche di stato](#verifiche-di-stato)
- [Diagnosi rapida problemi](#diagnosi-rapida-problemi)
- [Quadro elettrico QE1 — Relè](#quadro-elettrico-qe1--relè)
- [Curva climatica inversa (package 902)](#curva-climatica-inversa-package-902)
- [COP e efficienza](#cop-e-efficienza)
- [AI Optimizer (package 950)](#ai-optimizer-package-950)
- [Checklist post-modifica](#checklist-post-modifica)

---

## Panoramica

Il sistema di riscaldamento principale e' una pompa di calore aria-acqua **Hitachi YUTAKI RASM-4VNE**, controllata in tempo reale da Home Assistant via Modbus TCP attraverso il gateway **ATW-MBS-02**. Il sistema gestisce automaticamente:

- Riscaldamento a pavimento (circuito C1)
- Acqua calda sanitaria (ACS)
- Ottimizzazione COP tramite curva climatica inversa
- Integrazione con surplus fotovoltaico

---

## Specifiche PDC

| Parametro | Valore |
|-----------|--------|
| Modello | Hitachi RASM-4VNE |
| Tipo | Pompa di calore aria-acqua, inverter |
| Potenza termica | 3.3 – 11.0 kW (modulazione 30–100%) |
| Refrigerante | R410A |
| Modalita' operative | Riscaldamento, raffrescamento, ACS |
| Controllo | Modbus TCP via ATW-MBS-02 + HA |

### Pannello locale Yutaki

Pannello murale Hitachi con display LCD a colori, navigazione a frecce, tasto OK e pulsante power. Mostra temperature di mandata e ambiente. **Deve essere impostato in modalita' "Local"** per permettere il controllo da HA tramite Modbus; in modalita' Local il pannello fisico mostra i comandi ma li riceve da HA.

---

## Architettura controllo

```
Home Assistant (192.168.1.53:8123)
        │
        │ Modbus TCP (porta 502)
        ▼
ATW-MBS-02 (gateway Modbus — 192.168.1.5:502)
        │
        │ Bus H-LINK interno Hitachi
        ▼
Scheda PCB-MBS della YUTAKI RASM-4VNE
        │
        ├─ Circuito C1 (riscaldamento a pavimento)
        └─ ACS (acqua calda sanitaria)
                    ↕ (sincronizzato)
            PC-ARFH1E (controller locale)
```

**Ruoli:**

| Dispositivo | Funzione |
|---|---|
| **PC-ARFH1E** | Controller utente e installatore. Configurato UNA VOLTA in Central Mode Water. Deve restare acceso con tasto verde premuto. |
| **ATW-MBS-02** | Gateway Modbus. Traduce comandi Modbus in protocollo H-LINK Hitachi. |
| **Home Assistant** | Supervisore di sistema. Legge stati e scrive setpoint via `900_hitachi_control.yaml` (v16). |

---

## Gateway ATW-MBS-02

| Parametro | Valore |
|-----------|--------|
| Modello | ATW-MBS-02 |
| S/N | 7E549924 |
| IP | 192.168.1.5 |
| Porta Modbus TCP | 502 |
| Slave ID | 1 |
| Alimentazione | 230V 50Hz, 4.5W |
| Interfacce | USB (config), Ethernet 10/100, RS485, H-LINK (2 porte) |

### Specifiche tecniche

- **Ethernet:** Modbus TCP, RJ45, full-duplex, max 100m
- **RS485:** Modbus RTU, 9600/19200 baud, max 1200m
- **H-LINK:** Comunicazione con unita' YUTAKI S/S80/S COMBI/M, 9600 baud, max 1000m
- **1 gateway per sistema H-LINK, max 1 YUTAKI**

### DSW (dip switch)

| Switch | Funzione |
|--------|----------|
| SW1-1 | Resistenza terminale Modbus |
| SW1-2 | Non utilizzato — mantenere sempre ON |

---

## Configurazione obbligatoria PC-ARFH1E

Questa configurazione si esegue **una volta sola** sul controller fisico. Non e' modificabile via Modbus.

### Accesso al menu installatore

```
1. Premere CONTEMPORANEAMENTE:  OK + Indietro (Back)
2. Rilasciare
3. Premere:  →  (Freccia destra)
4. Premere:  ↓  (Freccia giu')
5. Premere:  ←  (Freccia sinistra)
6. Confermare:  OK (centrale)
```

### Impostazioni nel menu installatore

| Voce nel menu | Impostazione richiesta | Note |
|---|---|---|
| **Controllo centrale / Central Mode** | **Water (valore 2)** | NON usare Full (3). NON usare Local (0). |
| **Regolazione circuito C1** | **Temperatura fissa / Fix** | Disattivare la curva climatica OTC. |
| **BMS / Modbus / Central Control** | **Abilitato / On** | Necessario per ricevere comandi remoti. |

> **⚠️ IMPORTANTE — Central Mode:**
> - `Local (0)` = controllo remoto disabilitato. Nessun comando Modbus viene accettato.
> - `Air (1)` = controllo split interni. Non applicabile.
> - `Water (2)` = **CORRETTO** per impianto idraulico C1 + ACS.
> - `Full (3)` = controllo totale incluso Air. Non necessario, puo' causare conflitti.

> **⚠️ IMPORTANTE — Curva climatica C1:**
> Se la curva climatica OTC e' attiva sul PC-ARFH1E, il controller locale calcola in autonomia la temperatura di mandata in funzione della temperatura esterna. Questo **entra in conflitto** con il setpoint scritto via Modbus su addr1005. La curva climatica **deve essere disattivata** sul PC-ARFH1E, e il registro addr1003 deve essere = 3 (Fix).

### Verifica con tasto verde

Dopo la configurazione, premere il **tasto verde (Avvio/Arresto)** sul PC-ARFH1E. Sul display devono comparire dati per:
- `C1` con temperatura e setpoint visibili
- `ACS` con temperatura e setpoint visibili

Se C1 o ACS non mostrano dati → verificare che non sia attivo un **timer di pausa**. Se e' attiva una pausa timer, i comandi Modbus non producono effetto immediato.

---

## Registri Modbus — Tavola completa

### Convenzione indirizzi

Il manuale ATW-MBS-02 usa la colonna "Register" (1001, 1002...).  
Home Assistant usa la colonna "Address" = Register − 1 (1000, 1001...).  
In questo documento si usa **sempre la notazione addr per Home Assistant**.

### Registri di controllo — scrittura (addr 1000–1034)

#### Riscaldamento C1 (addr 1000–1013)

| addr HA | Registro | Funzione | Intervallo | Tipo | Valore tipico |
|---:|---:|---|---|---|---:|
| 1000 | 1001 | Controllo unita' Avvio/Arresto | 0=Arresto; 1=Avvio | R/W | 1 |
| 1001 | 1002 | Controllo modalita' unita' | 0=Raffr; 1=Risc; 2=Auto | R/W | 1 (Heat) |
| 1002 | 1003 | Controllo circuito 1 Avvio/Arresto | 0=Arresto; 1=Avvio | R/W | 1 |
| **1003** | **1004** | **Controllo OTC risc. C1** | **0=No; 1=Punti; 2=Grad; 3=Fisso** | **R/W** | **3 (Fix)** |
| 1004 | 1005 | Controllo OTC raffr. C1 | 0=No; 1=Punti; 2=Fisso | R/W | — |
| **1005** | **1006** | **Temperatura fissa risc. acqua C1** | **0–80 °C** | **R/W** | **es. 36** |
| 1006 | 1007 | Temperatura fissa raffr. acqua C1 | 0–80 °C | R/W | — |
| 1007 | 1008 | Modalita' Eco C1 | 0=ECO; 1=Comfort | R/W | 1 |
| 1008 | 1009 | Offset temperatura ECO risc. C1 | 1–10 | R/W | — |
| 1009 | 1010 | Offset temperatura ECO raffr. C1 | 1–10 | R/W | — |
| 1010 | 1011 | Termostato C1 disponibile | 0=no; 1=si | R/W | 0 |
| 1011 | 1012 | Temperatura impostata termostato C1 | 50–350 (5.0–35.0 °C) | R/W | — |
| 1012 | 1013 | Temperatura ambiente termostato C1 | 0–1000 (0.0–100.0 °C) | R/W | — |

#### Circuito C2 (addr 1014–1024)

| addr HA | Registro | Funzione | Intervallo | Tipo |
|---:|---:|---|---|---|
| 1013 | 1014 | Controllo circuito 2 Avvio/Arresto | 0=Arresto; 1=Avvio | R/W |
| 1014 | 1015 | Controllo OTC risc. C2 | 0=No; 1=Punti; 2=Grad; 3=Fisso | R/W |
| 1015 | 1016 | Controllo OTC raffr. C2 | 0=No; 1=Punti; 2=Fisso | R/W |
| 1016 | 1017 | Temperatura fissa risc. acqua C2 | 0–80 °C | R/W |
| 1017 | 1018 | Temperatura fissa raffr. acqua C2 | 0–80 °C | R/W |
| 1018 | 1019 | Modalita' Eco C2 | 0=ECO; 1=Comfort | R/W |
| 1019 | 1020 | Offset ECO risc. C2 | 1–10 | R/W |
| 1020 | 1021 | Offset ECO raffr. C2 | 1–10 | R/W |
| 1021 | 1022 | Termostato C2 disponibile | 0=no; 1=si | R/W |
| 1022 | 1023 | Temperatura impostata termostato C2 | 50–350 | R/W |
| 1023 | 1024 | Temperatura ambiente termostato C2 | 0–1000 | R/W |

#### ACS (addr 1024–1034)

| addr HA | Registro | Funzione | Intervallo | Tipo | Valore tipico |
|---:|---:|---|---|---|---:|
| 1024 | 1025 | Controllo ACS Avvio/Arresto | 0=Arresto; 1=Avvio | R/W | 1 |
| **1025** | **1026** | **Temperatura impostata ACS** | **0–80 °C** | **R/W** | **50** |
| 1026 | 1027 | Controllo boost ACS | 0=nessuna; 1=richiesta | R/W | 0 |
| 1027 | 1028 | Modalita' richiesta ACS | 0=standard; 1=alta | R/W | — |
| 1028 | 1029 | Controllo piscina Avvio/Arresto | 0=Arresto; 1=Avvio | R/W | — |
| 1029 | 1030 | Temperatura impostata piscina | 0–80 °C | R/W | — |
| 1030 | 1031 | Controllo anti-legionella | 0=Arresto; 1=Avvio | R/W | — |
| 1031 | 1032 | Temperatura impostata anti-legionella | 0–80 °C | R/W | — |
| 1032 | 1033 | Controllo blocco menu | 0=No; 1=Blocco | R/W | — |
| 1033 | 1034 | Controllo allarme BMS | 0=No; 1=Allarme | R/W | 0 |

### Registri di stato — lettura (addr 1050–1099)

#### Sistema generale (addr 1050–1099)

| addr HA | Registro | Funzione | Note | Valore atteso |
|---:|---:|---|---|---:|
| 1050 | 1051 | Stato unita' Avvio/Arresto | 0=Arresto; 1=Avvio | 1 |
| 1051 | 1052 | Stato modalita' unita' | 0=Raffr; 1=Risc | 1 (Heat) |
| 1052 | 1053 | Stato circuito 1 Avvio/Arresto | 0=Arresto; 1=Avvio | 1 |
| **1053** | **1054** | **Stato OTC risc. C1** | **0=No; 1=Punti; 2=Grad; 3=Fisso** | **3** |
| 1054 | 1055 | Stato OTC raffr. C1 | 0=No; 1=Punti; 2=Fisso | — |
| **1055** | **1056** | **Temperatura fissa risc. acqua C1** | **conferma setpoint** | **= addr1005** |
| 1056 | 1057 | Temperatura fissa raffr. acqua C1 | — | — |
| 1057 | 1058 | Modalita' Eco C1 | 0=ECO; 1=Comfort | 1 |
| 1058 | 1059 | Offset ECO risc. C1 | 1–10 | — |
| 1059 | 1060 | Offset ECO raffr. C1 | 1–10 | — |
| 1060 | 1061 | Temp. impostata termostato C1 | — | — |
| 1061 | 1062 | Temp. ambiente termostato C1 | — | — |
| 1062 | 1063 | Temp. impostata wireless C1 | — | — |
| 1063 | 1064 | Temp. ambiente wireless C1 | — | — |
| 1076 | 1077 | Stato ACS Avvio/Arresto | 0=Arresto; 1=Avvio | 1 |
| 1077 | 1078 | Temperatura impostata ACS | — | = addr1025 |
| 1078 | 1079 | Stato boost ACS | — | — |
| 1079 | 1080 | Modalita' richiesta ACS | — | — |
| **1081** | **1082** | **Temperatura ACS** | **-80~100 °C** | **reale** |
| 1088 | 1089 | **Modalita' centrale** | **0=Locale; 1=Aria; 2=Acqua; 3=Completa** | **2 (Water)** |
| 1090 | 1091 | Stato funzionamento | 0=OFF...11=allarme | vedi tabella |
| 1092 | 1093 | Temperatura ambiente esterna | -80~100 °C | reale |
| 1093 | 1094 | Temperatura ingresso acqua unita' | -80~100 °C | reale |
| 1094 | 1095 | Temperatura uscita acqua unita' | -80~100 °C | reale |
| 1095 | 1096 | Stato comunicazione H-LINK | 0=OK; 1=errore >180s; 2=init | **0** |

### Stato funzionamento (addr 1090 / R-OpState)

| Valore | Significato |
|---:|---|
| 0 | OFF |
| 1 | Richiesta raffrescamento OFF |
| 2 | Termo raffrescamento OFF |
| 3 | **Termo raffrescamento ON** |
| 4 | Richiesta riscaldamento OFF |
| 5 | Termo riscaldamento OFF |
| 6 | **Termo riscaldamento ON** |
| 7 | ACS OFF |
| 8 | **ACS ON** |
| 9 | Piscina OFF |
| 10 | Piscina ON |
| 11 | Allarme |

### Registri manutenzione (addr 1201–1232)

| addr HA | Registro | Descrizione | Intervallo |
|---:|---:|---|---:|
| 1201 | 1200 | Temperatura uscita acqua PDC | 0–100 °C |
| 1202 | 1201 | Ta2: temperatura media ambiente esterna | -80~100 °C |
| 1203 | 1202 | Ta: seconda temperatura ambiente | -80~100 °C |
| 1204 | 1203 | Ta3: seconda temperatura media ambiente | -80~100 °C |
| 1205 | 1204 | O2: temperatura uscita acqua 2 | -80~100 °C |
| 1206 | 1205 | O3: temperatura uscita acqua 3 | -80~100 °C |
| 1207 | 1206 | Tg: temperatura gas | -80~100 °C |
| 1208 | 1207 | Tl: temperatura liquido | -80~100 °C |
| 1209 | 1208 | Td: temperatura gas scarico | -80~100 °C |
| 1210 | 1209 | Te: temperazione evaporazione | -80~100 °C |
| 1211 | 1210 | EVI: apertura valvola espansione interna | 0–100% |
| 1212 | 1211 | EVO: valvola espansione esterna | 0–100% |
| 1213 | 1212 | H4: frequenza funzionamento inverter | 0–115 Hz |
| 1214 | 1213 | DI: causa arresto | — |
| 1215 | 1214 | P1: corrente funzionamento compressore | 0–30 A |
| 1216 | 1215 | CD: dati capacita' | — |
| 1217 | 1216 | MVP: posizione valvola miscelatrice | Solo C2 |
| 1218 | 1217 | Sbrinamento | — |
| 1219 | 1218 | Modello unita' | 0=YUTAKI S; 1=S COMBI; 2=S80; 3=M |
| 1220 | 1219 | Th: impostazione temperatura acqua | -80~100 °C |
| 1221 | 1220 | Livello flusso acqua | 0–30 (0.0–3.0 m3/h) |
| 1222 | 1221 | Velocita' pompa acqua | 0–100% |
| 1223 | 1222 | Stato sistema 2 | Bit: sbrin/solare/pompa1/pompa2/pompa3/compr/caldaia/riscACS/riscAmb/smart |
| 1224 | 1223 | Numero allarme | 0=nessun allarme |
| 1225 | 1224 | Temperatura scarico R134a | -80~100 °C |
| 1226 | 1225 | Temperatura aspirazione R134a | -80~100 °C |
| 1227 | 1226 | Pressione scarico R134a | 0–510 (0.00–5.10 MPa) |
| 1228 | 1227 | Pressione aspirazione R134a | 0–255 (0.00–2.55 MPa) |
| 1229 | 1228 | Frequenza compressore R134a | 0–115 Hz |
| 1230 | 1229 | Apertura valvola espansione interna 2 | 0–100% |
| 1231 | 1230 | Valore corrente compressore R134a | 0–300 (0.00–30.0 A) |
| 1232 | 1231 | Codice retry R134a | — |

---

## Correzione critica addr1003

### Il problema

Il registro addr1003 (Control Heat OTC Circuit 1) e' stato a lungo etichettato con commento errato `(deve=0)`. Questo e' **sbagliato**.

### Valori possibili addr1003

| Valore | Significato | Usare? |
|---:|---|---|
| 0 | No — OTC non configurato | ❌ NO |
| 1 | Points — curva a punti | ❌ NO |
| 2 | Gradient — curva a gradiente | ❌ NO |
| **3** | **Fix — temperatura fissa** | ✅ **SI'** |

**Valore 0** non significa "temperatura fissa senza curva". Significa che la modalita' OTC non e' definita, e il comportamento dipende dal PC-ARFH1E — tipicamente ritorna a curva climatica interna, **bypassando il comando Modbus su addr1005**.

**Valore 3 (Fix)** attiva esplicitamente la modalita' "usa la temperatura fissa scritta su addr1005". Questa e' l'unica configurazione coerente con il controllo remoto da HA.

### Cosa controllare

```
addr1003 W-OTC (deve=3)    = 3    ✓  Fix confermato
addr1053 R-OTC (0=No 3=Fix) = 3   ✓  Status mirror confermato
addr1055 R-Setpoint acqua °C = 36°C ✓  Setpoint ricevuto correttamente
addr1088 R-CentralMode raw   = 2   ✓  Water confermato
addr1094 R-H-LINK (0=OK)     = 0   ✓  Comunicazione H-LINK OK
```

---

## Sequenza di avvio operativa

### Prima accensione o dopo reset

```
1. Sul PC-ARFH1E: accedere al menu installatore (sequenza tasti sopra)
2. Impostare Central Mode = Water (2)
3. Impostare C1 = Temperatura fissa / Fix
4. Disattivare curva climatica C1 sul PC-ARFH1E
5. Premere il tasto VERDE (Avvio/Arresto)
6. Verificare che sul display compaiano dati su C1 e ACS
7. Verificare da HA/Modbus: addr1088 = 2 (CentralMode = Water)
```

### Avvio normale via Home Assistant

```
Scrivere in sequenza:
  addr1000 = 1    (System Run)
  addr1001 = 1    (Mode = Heat)
  addr1002 = 1    (C1 Run)
  addr1003 = 3    (OTC = Fix)
  addr1005 = XX   (Setpoint acqua C1 desiderato in °C)
  addr1024 = 1    (ACS Run)
  addr1025 = 50   (ACS setpoint)

Verificare dopo 30–60 secondi:
  addr1050 = 1    (System confermato acceso)
  addr1053 = 3    (OTC confermato Fix)
  addr1055 = XX   (setpoint confermato ricevuto)
  addr1088 = 2    (CentralMode = Water)
  addr1094 = 0    (H-LINK OK)
```

### Spegnimento via Modbus

```
  addr1000 = 0    (System Stop)
  addr1002 = 0    (C1 Stop)
  addr1024 = 0    (ACS Stop)
```

---

## Verifiche di stato

### Stato normale di funzionamento (riscaldamento C1 attivo)

| Register HA | Funzione | Valore corretto | Note |
|---:|---|---:|---|
| 1050 | R-System Run | **1** | Sistema acceso |
| 1051 | R-System Mode | **1** | Modalita' riscaldamento |
| 1052 | R-C1 Run | **1** | C1 attivo |
| 1053 | R-OTC C1 | **3** | Fix confermato |
| 1055 | R-Setpoint C1 | **= addr1005** | Conferma ricezione setpoint |
| 1057 | R-ECO | **1** | Comfort attivo |
| 1088 | R-CentralMode | **2** | Water — comunicazione attiva |
| 1094 | R-H-LINK | **0** | H-LINK OK |

---

## Diagnosi rapida problemi

| Sintomo | Causa probabile | Soluzione |
|---|---|---|
| addr1088 ≠ 2 | Central Mode non impostato su Water | Menu installatore PC-ARFH1E |
| addr1053 ≠ 3 | OTC non impostato su Fix | Controllare che addr1003 sia scritto = 3 |
| addr1055 ≠ setpoint scritto | Setpoint non ricevuto | Verificare addr1003 = 3 e che C1 sia in Run |
| addr1094 ≠ 0 | Errore comunicazione H-LINK | Controllare cablaggio ATW-MBS-02 / riavviare gateway |
| C1 non risponde ai comandi | Pausa timer attiva | Verificare sul PC-ARFH1E |
| Macchina accesa ma non scalda | Modalita' o setpoint errati | Verificare addr1001=1 e addr1005 corretto |

---

## Quadro elettrico QE1 — Relè

| Componente | Shelly | Funzione |
|-----------|--------|----------|
| R1 | `shellyplus1_90150687a670` | Abilita PDC (contattore bobina 220V) |
| R2 | `shellyplus1_1c692008dea0` | Circolatore P1 (pompa Danfoss) |
| R6 | — | Smart Grid — blocco PDC (normalmente NON eccitato) |
| R7 | — | Backup caldaia (normalmente SPENTO, PDC prioritaria) |

---

## Curva climatica inversa (package 902)

**File:** `902_pdc_ottimizzazione_v1.yaml`

Alza la temperatura di mandata acqua via Modbus in funzione del surplus FV. Controllore PI con integrale persistente.

**Input:**
- `sensor.pot_fv_totale_l1l2l3` — produzione FV totale
- `sensor.pot_tot_l1l2l3_abc` — consumo casa totale
- `sensor.pot_batterie_totale` — stato batterie
- `sensor.temp_media_zone_attive` — temperatura media zone
- `sensor.pdc_r_tempext` — temperatura esterna PDC

**Output:**
- `sensor.pdc_surplus_fv` — surplus disponibile per PDC
- `sensor.pdc_bilancio_*` — bilancio energetico
- `sensor.pdc_tbase` — temperatura base calcolata
- `sensor.pdc_termine_p` — termine proporzionale
- `sensor.pdc_delta_pi` — correzione PI
- `sensor.pdc_setpoint_target` — setpoint target da scrivere
- `input_number.pdc_k1` — guadagno proporzionale
- `input_number.pdc_integrale_i` — accumulo integrale

**Dipende da:** 810, 920, 925

---

## COP e efficienza

### COP Teorico (package 907)

Calcola il COP teorico basato su dati di targa della Hitachi RASM-4VNE.

**⚠️ ATTENZIONE:** Le formule sono per **R410A**, non per R32. Da rivedere con dati invernali (scadenza: settembre 2026).

**Input:**
- `sensor.temp_media_esterna`
- `sensor.conteca_temperatura_mandata`
- `sensor.conteca_temperatura_ritorno`
- `sensor.conteca_potenza`
- `sensor.shellyem3_485519dc8109_phase_a_potenza`

**Output:**
- `sensor.cop_predetto_pdc_hitachi_teorico`
- `sensor.cop_reale_pdc`
- `sensor.efficienza_pdc_vs_elettrico`
- `sensor.risparmio_energetico_stimato`

### COP Reale (package 935)

Traccia energia termica ed elettrica della PDC separando riscaldamento da ACS.

```
COP_reale = Potenza_termica_CONTECA [kW] / Potenza_elettrica_PDC [kW]
```

**Input:** `sensor.conteca_*`, `sensor.shellyem_c45bbee1c642_channel_1_*`, `binary_sensor.pdc_modalita_riscaldamento`
**Output:** Energia termica/elettrica filtrata, COP globale e filtrato, utility meter multi-periodo

---

## AI Optimizer (package 950)

**File:** `950_pdc_optimizer.yaml`

Helper per il sistema AI di ottimizzazione PDC (AppDaemon `pdc_optimizer.py`).

**Input:** Scritture da AppDaemon su `input_text.pdc_piano_24h_json` e `pdc_report_giornaliero`
**Output:**
- `sensor.pdc_ai_kwh_stimati_oggi`
- `sensor.pdc_ai_risparmio_stimato_*`
- `sensor.pdc_ai_cop_effettivo_previsto`
- `sensor.pdc_risparmio_reale_ieri_kwh`
- `sensor.pdc_accuratezza_previsione`

**Helper:**
- `input_boolean.pdc_ai_mode` — attiva/disattiva AI
- `input_number.pdc_pel_min_kw` — potenza elettrica minima
- `input_number.pdc_partenze_massime` — massimo cicli/giorno
- `input_number.dispersione_termica_coeff`
- `input_number.pdc_t_comfort_minima`
- `input_number.pdc_t_target_sera`

---

## Checklist post-modifica

Dopo ogni modifica a `900_hitachi_control.yaml` o package correlati:

- [ ] Commento addr1003 corretto: `(deve=3 = Fix)`
- [ ] File YAML ricaricato in HA (Developer Tools → YAML → Reload)
- [ ] Nessun errore nel log HA relativo a modbus
- [ ] Card Lovelace aggiornata con etichetta corretta
- [ ] `sensor.pdc_r_opstate` ha valore numerico (non unavailable)
- [ ] addr1088 = 2 (CentralMode = Water)
- [ ] addr1053 = 3 (OTC = Fix)
- [ ] addr1094 = 0 (H-LINK OK)

---

*Documento generato il 2026-06-23 — Integra ARFH1E_ATW-MBS-02_HA v1.0 e manuale ATW-MBS-02 PMML0419 rev.3*
