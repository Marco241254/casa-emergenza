# 🏠 CHECKLIST RISCALDAMENTO PDC - "CALDO IN CASA"

## 🎯 OBIETTIVO: RISCALDAMENTO FUNZIONANTE
**Questa checklist verifica che TUTTO il sistema lavori correttamente**

---

## 🔥 RISULTATO FINALE (dopo 1 ora da avvio)

- [ ] **1. Termosifoni CALDI** dopo 1h da partenza PDC
- [ ] **2. Valvole termostatiche TADO** regolate correttamente  
- [ ] **3. Batterie TADO** cambiate (non scariche)
- [ ] **4. Pompa circolazione P1** gira regolarmente
- [ ] **5. Pressione circuito** termosifoni: **1,0-1,5 bar**
- [ ] **6. Rumore circolatore** tipo tintinnio (normale)

---

## ⚙️ CONTROLLI ELETTRICI - ARMADIO QE2 (Shelly)

- [ ] **7. LED Shelly R2** (circolatore): **ACCESO** 🔴
- [ ] **8. LED Shelly R1** (PDC): **ACCESO** 🔴  
- [ ] **9. LED Shelly R6** (blocco): **SPENTO** ⚫

---

## 🔴 CONTROLLI VISORI - QUADRO QE1

- [ ] **10. Visore R2** (circolatore): **ROSSO** 🔴
- [ ] **11. Visore R1** (PDC): **ROSSO** 🔴
- [ ] **12. Visore R6** (blocco): **GRIGIO** ⚫

> **❌ Se qualche visore è sbagliato → [PROCEDURA EMERGENZA N°1](#-procedura-emergenza-n1)**

---

## 🎮 CONTROLLI CONTROLLER YOUTAKY

- [ ] **13. Controller YOUTAKY**: ALIMENTATO
- [ ] **14. Set-point acqua PDC**: INFERIORE a temperatura attuale
- [ ] **15. Simbolo richiesta caldo**: PRESENTE su display YOUTAKY

---

## 🌐 SOTTO-CHECKLIST INFRASTRUTTURA

### 📡 CONNETTIVITÀ
- [ ] **A1. Internet**: ATTIVO
- [ ] **A2. Wi-Fi**: CONNESSO  
- [ ] **A3. Router**: ACCESO (Locale Tecnico)

### 🏠 HOME ASSISTANT
- [ ] **B1. Server Home Assistant**: FUNZIONANTE
- [ ] **B2. Interfaccia HA**: ACCESSIBILE
- [ ] **B3. Dispositivi Shelly**: ONLINE in HA
- [ ] **B4. Automazioni**: ATTIVE

### 🔍 COME CONTROLLARE HOME ASSISTANT:

```bash
1. Apri browser → http://homeassistant.local:8123
2. Verifica dashboard principale carica
3. Controlla entità Shelly (devono essere "on/off" non "unavailable")
4. Verifica ultima attività automazioni
```

#### Entità Shelly da verificare:
- **R1 (PDC)**: `shellyplus1_90150687a670_switch_0`
- **R2 (Circolatore)**: `shellyplus1_1c692008dea0_switch_0` 
- **R6 (Blocco)**: `shellyplus1-8813bfcdbda0`
- **R7 (Caldaia)**: `shellyplus1-1c692044c2ac`

---

## ✅ RISULTATO FINALE CHECKLIST

### 🎯 SE TUTTI I CONTROLLI SONO OK:

```
🔥 PDC DEVE PARTIRE IN 60 SECONDI
🏠 RISCALDAMENTO ATTIVO  
🌡️ CASA CALDA IN 1-2 ORE
```

### ❌ SE QUALCOSA NON VA:

#### 🔴 PROBLEMA FISICO (termosifoni, pressione, pompe):
**📞 CHIAMARE ELVIS: [3474958076](tel:3474958076)** (San Germano - Idraulica)

#### ⚡ PROBLEMA ELETTRICO (LED, visori, alimentazioni):
**📞 CHIAMARE PAPÀ: [3351354309](tel:3351354309)**

#### 🌐 PROBLEMA DIGITALE (Internet, Home Assistant, Shelly):
**📞 CHIAMARE PAPA':  [3351354309](tel:3351354309)**

#### 🎮 PROBLEMA YOUTAKY (controller, set-point):
**📞 CHIAMARE ELVIS: [3474958076](tel:3474958076)** (specialista PDC):
**📞 CHIAMARE ALESSANDRO: [370-3068232](tel:370-3068232)** (specialista PDC)
---

## 🚨 PROCEDURA EMERGENZA N°1

**Se visori contattori QE1 sono sbagliati:**

1. **SPEGNERE** interruttore generale QE1 per **30 secondi**
2. **RIACCENDERE** e attendere **2 minuti**  
3. **CONTROLLARE** che tutti i visori tornino alle posizioni corrette:
   - R1: **ROSSO** 🔴
   - R2: **ROSSO** 🔴  
   - R6: **GRIGIO** ⚫
4. **Se ancora sbagliati** → Chiamare **PAPÀ [3351354309](tel:3351354309)**

---

## 📋 SCHEMA FUNZIONAMENTO SISTEMA

### Relè e Contattori (Quadro QE1):

| ID | Tipo | Funzione | Stato Normale | Shelly |
|----|------|----------|---------------|--------|
| **R1** | Contattore 25A | Abilitazione PDC tramite YOUTAKY | ROSSO (ON) | `shellyplus1_90150687a670` |
| **R2** | Contattore 25A | Pompa circolazione P1 termosifoni | ROSSO (ON) | `shellyplus1_1c692008dea0` |
| **R6** | Contattore 25A | Blocco Smart Grid YOUTAKY | GRIGIO (OFF) | `shellyplus1-8813bfcdbda0` |
| **R7** | Contattore 25A | Alimentazione Caldaia Gas (antiblocco) | GRIGIO (OFF) | `shellyplus1-1c692044c2ac` |

### Logica di Funzionamento:
- **R1 ON** → PDC abilitata tramite YOUTAKY
- **R2 ON** → Pompa P1 alimentata per circolazione termosifoni  
- **R6 OFF** → PDC NON bloccata (può funzionare)
- **R6 ON** → PDC BLOCCATA tramite Smart Grid (non può funzionare)

---

## 📱 QR CODE PER ACCESSO RAPIDO

Scansiona per accedere rapidamente a:
- Home Assistant Dashboard
- Questa checklist
- Contatti di emergenza

---

## 📋 NOTE PER L'UTENTE

- **Completare TUTTA la checklist** prima di chiamare assistenza
- **Identificare la sezione** che fallisce per chiamare il tecnico giusto
- **Tenere questa pagina aperta** durante i controlli  
- **Ripetere controlli** dopo ogni intervento tecnico
- **Fotografie** sempre lo stato dei visori QE1 prima di chiamare assistenza

---

*Checklist Master Riscaldamento v1.0*  
*Aggiornata: 29/05/2025*  
*Repository: [casa-emergenze](https://github.com/Marco241254/casa-emergenze)*
