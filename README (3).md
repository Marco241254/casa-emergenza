# Casa GEEKOM — Documentazione Sistema Domotico

**Indice documentazione** | [Infrastruttura](00_infrastruttura.md) | [Home Assistant](01_home_assistant.md) | [PDC Hitachi](02_pdc_hitachi.md) | [Fotovoltaico](03_fotovoltaico.md) | [Tesla/Wallbox](04_tesla_wallbox.md) | [Energia](05_energia.md) | [Clima](06_clima_temperature.md) | [Operazioni](07_operazioni.md) | [Riferimenti](08_riferimenti.md)

---

**Sistema:** TERMO_OPTIMA — gestione energetica integrata
**Ubicazione:** Pomaretto (TO), Piemonte, ~626 m s.l.m.  
**Aggiornato:** 2026-06-23  
**Versione documentazione:** 2.0

---

## Panoramica del sistema

Casa GEEKOM e' un sistema domotico avanzato che coordina in tempo reale produzione fotovoltaica (16.28 kWp), accumulo in batterie (19.7 kWh), riscaldamento a pompa di calore (Hitachi RASM-4VNE), acqua calda sanitaria, ricarica solare Tesla Model Y e irrigazione — tutto gestito da Home Assistant su Proxmox.

### Filosofia di controllo

- **Autoconsumo massimo**: ogni carico flessibile (Tesla, PDC, ACS) viene attivato/modulato sul surplus FV istantaneo
- **Riserva batterie**: 33% (6.5 kWh) sempre disponibile per carichi domestici
- **Resilienza**: ogni sottosistema ha fail-safe hardware per funzionamento manuale indipendente da HA
- **Ottimizzazione COP**: la PDC viene modulata dinamicamente per massimo rendimento

---

## Architettura ad alto livello

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLUSTER PROXMOX (GEEKOM)                      │
│                                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│  │ VM 100   │ │ LXC 110  │ │ LXC 120  │ │ LXC 130  │          │
│  │ HAOS     │ │ InfluxDB │ │ Grafana  │ │meteo-    │          │
│  │192.168.1.│ │192.168.1.│ │          │ │corrector │          │
│  │    53    │ │    48    │ │          │ │          │          │
│  └────┬─────┘ └──────────┘ └──────────┘ └──────────┘          │
│       │                                            ┌──────────┐│
│       │                                            │ LXC 131  ││
│       │                                            │energy-   ││
│       │                                            │allocator ││
│       │                                            └──────────┘│
│       │                                            ┌──────────┐│
│       │                                            │ LXC 200  ││
│       │                                            │ MariaDB  ││
│       │                                            │192.168.1.││
│       │                                            │   165    ││
│       │                                            └──────────┘│
│       ▼                                                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              HOME ASSISTANT OS (VM 100)                   │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐         │   │
│  │  │ 40 YAML │ │  Modbus │ │  Tesla  │ │ AppDae- │         │   │
│  │  │packages │ │ Hitachi │ │   BLE   │ │  mon    │         │   │
│  │  │         │ │ TCP 502 │ │  PID    │ │pdc_opti-│         │   │
│  │  └─────────┘ └────┬────┘ └─────────┘ └─────────┘         │   │
│  │                   │                                       │   │
│  └───────────────────┼───────────────────────────────────────┘   │
│                      │                                           │
└──────────────────────┼───────────────────────────────────────────┘
                       │
    ┌──────────────────┼──────────────────┬──────────────┐
    │                  │                  │              │
    ▼                  ▼                  ▼              ▼
┌─────────┐    ┌─────────────┐    ┌──────────┐    ┌──────────┐
│ATW-MBS-2│    │ESP32 Garage │    │Shelly EM │    │ SolarEdge│
│Gateway  │    │BLE Tesla    │    │  family  │    │ +Enphase │
│Modbus   │    │192.168.1.175│    │sensors   │    │ +Batterie│
│192.168.1│    │             │    │          │    │          │
│    .5   │    │ESP32 Cancel-│    │Shelly EM3│    │CONTECA   │
└────┬────┘    │lo 192.168.1 │    │trifase   │    │termico   │
     │         │    .176      │    │          │    │          │
     │         └─────────────┘    └──────────┘    └──────────┘
     │
     ▼
┌────────────────────┐
│ Hitachi YUTAKI     │
│ RASM-4VNE          │
│ PC-ARFH1E          │
│ (riscaldamento+ACS)│
└────────────────────┘
```

---

## Sottosistemi — documentazione dedicata

| Documento | Cosa trovi | Stato |
|-----------|-----------|-------|
| [00_infrastruttura.md](00_infrastruttura.md) | Proxmox, VM/LXC, rete, IP, dispositivi fisici, schema completo | Aggiornato 2026-06 |
| [01_home_assistant.md](01_home_assistant.md) | Configurazione HA, 40 packages YAML, mappa dipendenze, entita' principali | Aggiornato 2026-06 |
| [02_pdc_hitachi.md](02_pdc_hitachi.md) | Pompa di calore, gateway Modbus ATW-MBS-02, registri, curva climatica, controllo | Aggiornato 2026-06 |
| [03_fotovoltaico.md](03_fotovoltaico.md) | 4 stringhe FV (16.28 kWp), previsioni pvlib LXC 130, correttore k, forecast | Aggiornato 2026-06 |
| [04_tesla_wallbox.md](04_tesla_wallbox.md) | Tesla Model Y BLE, PID carica solare, ESP32, wallbox, logica surplus | Aggiornato 2026-06 |
| [05_energia.md](05_energia.md) | Monitoraggio consumi per zona, costi ENEL, batterie (19.7 kWh), COP reale | Aggiornato 2026-06 |
| [06_clima_temperature.md](06_clima_temperature.md) | Termostati zone, temperature, gradi giorno, forecast 24h COP | Aggiornato 2026-06 |
| [07_operazioni.md](07_operazioni.md) | Backup 3 livelli, disaster recovery, deploy, checklist, verifiche | Aggiornato 2026-06 |
| [08_riferimenti.md](08_riferimenti.md) | IP, modelli, contatti tecnici, credenziali, materiale esterno | Aggiornato 2026-06 |

---

## Mappa rapida dei 40+ packages YAML

```
Package  Scopo
─────────────────────────────────────────────────────────
2000s    Sistema
  2010   Monitoraggio hardware GEEKOM + VM Proxmox

3000s    Automazioni speciali
   300   Pompa circolazione R2 in AND con stato PDC
   310   Telecomandi Zigbee Ikea Rodret
   400   Irrigazione prati (Sonoff 4ch)

7000s    Fotovoltaico — Previsioni
   710   Setting FV: blending Solcast/Forecast Solar
   720   Previsioni FV da Open-Meteo

8000s    Fotovoltaico — Monitoraggio
   810   Potenze elettriche normalizzate (filtro mediano)
   820   Energie elettriche normalizzate
   821   Somma produzione FV Enphase
   822   Prelievo/cessione ENEL (27 sensori)
   823   Costo reale prelievo ENEL trifase
   824   Produzione FV giornaliera
   825   Produzione FV periodi (settimanale/mensile/bim/ann)
   826   Produzione FV completa
   827   Prelievo/cessione costi reali (somma algebrica)
   828   Utility meter FV
   829   Integrale batteria SolarEdge (carica/scarica)

830s     Energie per zona
   830   Wallbox (2 colonnine trifase)
   831   PDC (Shelly EM c45bbee1c642 ch1)
   832   Cucina
   833   Primo piano
   834   Mansarda
   835   Luci esterne
   836   Lavanderia
   837   Climatizzatori

900s     Pompa di Calore Hitachi
   900   Controllo/diagnostica Modbus + definizione hub (v16)
   902   Curva climatica inversa (PI surplus FV -> setpoint)
   905   Correzione portate Hitachi (fattore 1.159 vs CONTECA)
   907   COP teorico (⚠️ formule R410A, non R32)
   910   Sensori contatore termico CONTECA
   920   Sensori Modbus PDC normalizzati (R/W/Status)
   925   Temperature ambienti + media pesata zone attive
   926   Temperatura esterna (media 3 fonti)
   927   Gradi giorno riscaldamento
   928   Lockout AND termostato + allungamento ciclo OFF
   930   Termostato fasce orarie per zona
   935   Energia termica/elettrica PDC + COP reale
   940   Eccedenza FV irrecuperabile + analisi ROI batterie
   950   Helper AI optimizer PDC (AppDaemon)
   960   Costo riscaldamento vs prelievo ENEL
   970   Duty cycle PDC da fabbisogno termico
   971   Misura duty cycle reale (cmd vs potenza Shelly)
   972   GG previsti 24h (Open-Meteo)
   973   COP previsto 24h + fabbisogno elettrico

200s     Tesla
   210   Carica Tesla ottimizzata su surplus FV via BLE ESP32 garage
   212   Pre-controller energia (carica notturna + travaso batterie)
```

---

## Verifiche post-riavvio (checklist rapida)

Dopo ogni riavvio di HA o modifica ai packages, verificare:

- [ ] Nessun errore in **Impostazioni -> Sistema -> Registri**
- [ ] Modbus PDC: `sensor.pdc_r_opstate` ha valore numerico (non `unavailable`)
- [ ] `addr1003` = 3 (Fix) — altrimenti la PDC ignora il setpoint
- [ ] `addr1088` = 2 (CentralMode = Water)
- [ ] `addr1094` = 0 (H-LINK OK)
- [ ] ESP32 Tesla BLE: `binary_sensor.tesla_ble_pronta` = ON se auto in garage
- [ ] Previsioni FV: sensori `pvlib_oggi_*` popolati (ore 6-20)
- [ ] InfluxDB connesso: storico grafici accessibili
- [ ] Recorder MariaDB: entita' storiche caricano

---

## Note di manutenzione aperte

| Problema | Scadenza | Stato |
|----------|----------|-------|
| 907 formule COP per R410A (non R32) | Settembre 2026 | ⚠️ Da rivedere con dati invernali |
| 720_previsioni_FV — verificare friendly_name | Giugno 2026 | ⚠️ Da verificare |
| ACS setpoint reale 30°C (sotto soglia anomalia 38°C) | Da verificare | ⚠️ Intenzionale o residuo? |
| ECO mode = 0 (attivo) — setpoint -2°C al riaccendimento | Settembre 2026 | ⚠️ Da verificare prima inverno |
| `correttore.py` manca `state_class: measurement` | — | ⚠️ Warning HA, nessun dato rotto |
| Orizzonte OVEST aggressivo in `pvlib_ha2.py` | — | ⚠️ Da valutare vs dati reali |
| Backup GEEKOM → BEELINK (warm standby) | Appena possibile | ⏳ In attesa |
| ~~Modbus PDC riabilitare~~ | ~~Maggio~~ | ✅ **Risolto 2026-05-22** |
| ~~addr1003 W-OTC commento errato~~ | ~~Maggio~~ | ✅ **Risolto 2026-05-22** |

---

## Convenzioni del sistema

- **Indirizzi Modbus HA**: usare sempre notazione `addr` = Register − 1 (es. registro 1004 -> addr 1003)
- **File YAML packages**: versione obbligatoria nel commento iniziale, formato `v{N}`
- **Utility meter**: chiave = `unique_id`; nessun campo `name:` per garantire `entity_id` prevedibile
- **Coordinate GPS**: se modificate in `configuration.yaml`, aggiornare ANCHE `packages/972_gg_forecast_24h.yaml`
- **Vista Lovelace**: i file iniziano sempre con `views:`, layout masonry `custom:masonry-layout`
- **Deploy file**: UN file alla volta, con backup prima (`.bak` o backup HA)

---

*Documentazione generata il 2026-06-23 — Versione 2.0*
*Aggiornare a ogni modifica strutturale*
