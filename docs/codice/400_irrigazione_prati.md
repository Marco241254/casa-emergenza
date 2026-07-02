# 400_irrigazione_prati.md
# Versione: 1.1.1
# Data: 2026-07-01
# Autore: Marco / TERMO_OPTIMA
# Scope: Documentazione sistema irrigazione FV-Only + bypass rete. Sonoff 4CH PRO R3.
# Sorgente: config/packages/400_irrigazione_prati.yaml
# =============================================================================

# 400_irrigazione_prati.yaml

## 1. Identità

| Campo | Valore |
|-------|--------|
| **File sorgente** | `config/packages/400_irrigazione_prati.yaml` |
| **Documentazione** | `docs/codice/400_irrigazione_prati.md` |
| **Versione** | 1.1.1 |
| **Data** | 2026-07-01 |
| **Autore** | Marco / TERMO_OPTIMA |
| **Scope** | Sistema irrigazione FV-Only con bypass rete. Sonoff 4CH PRO R3. |
| **Area** | Irrigazione / Giardino |
| **Stato** | 🟢 Attivo |

---

## 2. Descrizione Funzionale

Package monolitico per il controllo dell'irrigazione a 3 zone (SOPRA, SOTTO, PICCOLO) tramite Sonoff 4CH PRO R3.

**Logica di business:**
- **Modalità FV-Only (default):** l'irrigazione parte solo se il surplus fotovoltaico supera la soglia configurata. Se durante il ciclo il surplus scende sotto soglia, avvia un timer di grazia di 15 minuti; se non recupera, ferma tutto.
- **Modalità Bypass (v1.1.0):** l'utente può forzare l'avvio anche senza surplus FV, prelevando dalla rete. In bypass le guardie FV sono disattivate.
- **Ciclo singolo:** una zona alla volta, durata configurabile via slider.
- **Rotazione a caldo:** SOPRA → SOTTO → PICCOLO in sequenza, pompa sempre accesa durante il passaggio, durata per zona configurabile.
- **Lockout post-ciclo:** 5 minuti di blocco dopo ogni fine ciclo per evitare avviamenti troppo ravvicinati (protezione elettropompa).
- **Sequenza sicura:** valvola prima, pompa dopo (+2s); stop inverso: pompa prima, valvola dopo (+2s).

---

## 3. Input

| # | Nome | Sorgente | Tipo | Unità | Range | Freq | Visualizzato da | Consumato da | Note |
|---|------|----------|------|-------|-------|------|-----------------|--------------|------|
| 1 | `sensor.pv_total_power_w` | Inverter FV | sensor | W | 0–15k | 10s | Dashboard, Grafana | `sensor.irrigazione_surplus_fv` | Produzione istantanea impianto FV |
| 2 | `sensor.pot_tot_l1l2l3_abc` | Contatore rete / Gateway | sensor | W | -10k–15k | 5s | Dashboard, Grafana | `sensor.irrigazione_surplus_fv` | Consumo totale casa (L1+L2+L3) |
| 3 | `input_number.irr_durata_minuti` | UI Dashboard | number | min | 1–180 | - | Dashboard | `script.irr_sequenza_avvio` | Durata ciclo singolo |
| 4 | `input_number.irr_durata_rotazione_minuti` | UI Dashboard | number | min | 1–180 | - | Dashboard | `script.irr_rotazione_completa` | Durata per zona in rotazione |
| 5 | `input_number.irr_soglia_avvio_w` | UI Dashboard | number | W | 800–4000 | - | Dashboard | `binary_sensor.irrigazione_fv_sufficiente_avvio` | Surplus minimo per avvio FV |
| 6 | `input_number.irr_soglia_stop_w` | UI Dashboard | number | W | 200–2000 | - | Dashboard | `binary_sensor.irrigazione_fv_sotto_soglia_stop` | Soglia sotto cui scatta grazia |
| 7 | `input_select.irr_zona_scelta` | UI Dashboard | select | - | SOPRA/SOTTO/PICCOLO | - | Dashboard | `script.irr_sequenza_avvio` | Zona per ciclo singolo |
| 8 | `input_boolean.irr_bypass_guardia_fv` | UI Dashboard | bool | - | on/off | - | Dashboard | Script, Automazioni | Forza avvio senza FV |
| 9 | `input_button.irr_avvia_singolo` | UI Dashboard | button | - | press | - | Dashboard | `automation.irr_btn_singolo` | Trigger avvio singolo |
| 10 | `input_button.irr_avvia_rotazione` | UI Dashboard | button | - | press | - | Dashboard | `automation.irr_btn_rotazione` | Trigger avvio rotazione |
| 11 | `binary_sensor.irrigazione_fv_sufficiente_avvio` | **Questo file** (template) | binary | - | on/off | 5s | Dashboard | `script.irr_sequenza_avvio`, `script.irr_rotazione_completa`, `automation.irr_auto_fv_recupero` | Surplus &gt; soglia avvio |
| 12 | `binary_sensor.irrigazione_fv_sotto_soglia_stop` | **Questo file** (template) | binary | - | on/off | 5s | Dashboard | `automation.irr_auto_fv_sotto_soglia` | Surplus &lt; soglia stop |
| 13 | `input_boolean.irr_lockout` | **Questo file** (stato) | bool | - | on/off | - | Dashboard | `script.irr_sequenza_avvio`, `script.irr_rotazione_completa`, `automation.irr_auto_fine_lockout` | Blocco post-ciclo |
| 14 | `input_boolean.irr_ciclo_attivo` | **Questo file** (stato) | bool | - | on/off | - | Dashboard | `automation.irr_auto_fine_durata`, `automation.irr_auto_fv_sotto_soglia`, `automation.irr_auto_fv_grazia_scaduta` | Stato ciclo in corso |
| 15 | `input_boolean.irr_rotazione_attiva` | **Questo file** (stato) | bool | - | on/off | - | Dashboard | `automation.irr_auto_fine_durata`, `automation.irr_auto_fv_grazia_scaduta` | Stato rotazione in corso |
| 16 | `timer.irr_zona` | **Questo file** | timer | - | - | - | Dashboard | `automation.irr_auto_fine_durata` | Scadenza durata singolo |
| 17 | `timer.irr_fv_grazia` | **Questo file** | timer | - | - | - | Dashboard | `automation.irr_auto_fv_grazia_scaduta`, `automation.irr_auto_fv_recupero` | Grazia 15min FV basso |
| 18 | `timer.irr_lockout` | **Questo file** | timer | - | - | - | Dashboard | `automation.irr_auto_fine_lockout` | Lockout 5min post-ciclo |

---

## 4. Output

| # | Nome | Destinazione | Tipo | Unità | Range | Usato da | Triggerato da | Note |
|---|------|--------------|------|-------|-------|----------|---------------|------|
| 1 | `switch.irrigatori_sonoff_100114bc35_1` | Sonoff 4CH PRO R3 (CH1) | switch | - | on/off | Pompa elettropompa | `script.irr_sequenza_avvio`, `script.irr_rotazione_completa`, `script.irr_sequenza_stop` | Pompa acqua |
| 2 | `switch.irrigatori_sonoff_100114bc35_2` | Sonoff 4CH PRO R3 (CH2) | switch | - | on/off | Elettrovalvola SOPRA | `script.irr_sequenza_avvio`, `script.irr_rotazione_completa`, `script.irr_sequenza_stop` | Valvola zona SOPRA |
| 3 | `switch.irrigatori_sonoff_100114bc35_3` | Sonoff 4CH PRO R3 (CH3) | switch | - | on/off | Elettrovalvola SOTTO | `script.irr_sequenza_avvio`, `script.irr_rotazione_completa`, `script.irr_sequenza_stop` | Valvola zona SOTTO |
| 4 | `switch.irrigatori_sonoff_100114bc35_4` | Sonoff 4CH PRO R3 (CH4) | switch | - | on/off | Elettrovalvola PICCOLO | `script.irr_sequenza_avvio`, `script.irr_rotazione_completa`, `script.irr_sequenza_stop` | Valvola zona PICCOLO |
| 5 | `input_boolean.irr_ciclo_attivo` | Stato interno HA | bool | - | on/off | Automazioni, Dashboard | `script.irr_sequenza_avvio`, `script.irr_rotazione_completa`, `script.irr_sequenza_stop` | Flag ciclo ON |
| 6 | `input_boolean.irr_rotazione_attiva` | Stato interno HA | bool | - | on/off | Automazioni, Dashboard | `script.irr_rotazione_completa`, `script.irr_sequenza_stop` | Flag rotazione ON |
| 7 | `input_boolean.irr_lockout` | Stato interno HA | bool | - | on/off | Script, Automazioni, Dashboard | `script.irr_sequenza_stop`, `automation.irr_auto_fine_lockout` | Flag blocco post-ciclo |
| 8 | `timer.irr_zona` | Timer interno HA | timer | - | 0–180min | `automation.irr_auto_fine_durata` | `script.irr_sequenza_avvio` | Timer ciclo singolo |
| 9 | `timer.irr_fv_grazia` | Timer interno HA | timer | - | 15min | `automation.irr_auto_fv_grazia_scaduta`, `automation.irr_auto_fv_recupero` | `automation.irr_auto_fv_sotto_soglia` | Timer attesa FV |
| 10 | `timer.irr_lockout` | Timer interno HA | timer | - | 5min | `automation.irr_auto_fine_lockout` | `script.irr_sequenza_stop` | Timer blocco post-ciclo |
| 11 | `sensor.irrigazione_surplus_fv` | Template interno HA | sensor | W | -10k–15k | Dashboard, Grafana, Binary sensor | **Questo file** (template) | Surplus calcolato PV - Casa |
| 12 | `sensor.irrigazione_stato` | Template interno HA | text | - | - | Dashboard | **Questo file** (template) | Stato testo per UI |
| 13 | `telegram_bot.send_message` | Telegram (chat_id: 920319768) | notify | - | - | Utente (smartphone) | Script, Automazioni | Notifiche push |
| 14 | `script.irr_sequenza_stop` | Script interno HA | script | - | - | `script.irr_rotazione_completa`, `automation.irr_auto_fine_durata`, `automation.irr_auto_fv_grazia_scaduta` | `script.irr_rotazione_completa` (fine), Automazioni | Stop sequenziato |

---

## 5. Diagramma di Flusso (Mermaid)

```mermaid
graph LR
    A[Inverter FV] --&gt; B[sensor.pv_total_power_w]
    C[Contatore Rete] --&gt; D[sensor.pot_tot_l1l2l3_abc]
    B --&gt; E[sensor.irrigazione_surplus_fv]
    D --&gt; E
    E --&gt; F[binary_sensor.irrigazione_fv_sufficiente_avvio]
    E --&gt; G[binary_sensor.irrigazione_fv_sotto_soglia_stop]

    H[Dashboard UI] --&gt; I[input_number.irr_soglia_avvio_w]
    H --&gt; J[input_number.irr_soglia_stop_w]
    H --&gt; K[input_select.irr_zona_scelta]
    H --&gt; L[input_boolean.irr_bypass_guardia_fv]
    H --&gt; M[input_button.irr_avvia_singolo]
    H --&gt; N[input_button.irr_avvia_rotazione]

    M --&gt; O[automation.irr_btn_singolo]
    N --&gt; P[automation.irr_btn_rotazione]
    O --&gt; Q[script.irr_sequenza_avvio]
    P --&gt; R[script.irr_rotazione_completa]

    I --&gt; F
    J --&gt; G
    Q --&gt; S{Guardie}
    R --&gt; S
    S --&gt;|OK| T[switch.irrigatori_sonoff_*]
    S --&gt;|Lockout| U[Telegram: negato]
    S --&gt;|FV basso + no bypass| U

    T --&gt; V[timer.irr_zona]
    T --&gt; W[timer.irr_lockout]

    G --&gt; X[automation.irr_auto_fv_sotto_soglia]
    X --&gt; Y[timer.irr_fv_grazia]
    Y --&gt; Z[automation.irr_auto_fv_grazia_scaduta]
    Z --&gt; AA[script.irr_sequenza_stop]
    F --&gt; AB[automation.irr_auto_fv_recupero]
    AB --&gt; AC[timer.cancel irr_fv_grazia]

    V --&gt; AD[automation.irr_auto_fine_durata]
    AD --&gt; AA
    W --&gt; AE[automation.irr_auto_fine_lockout]
    AE --&gt; AF[input_boolean.irr_lockout = OFF]

    AA --&gt; AG[Telegram: completato]
    AA --&gt; AH[input_boolean.irr_lockout = ON]
