# Failover fibra TIM su LTE TIM con UDR7 + TP-Link Archer MR600

**Documento tecnico operativo**  
**Sistema:** rete domestica con fibra TIM principale e backup LTE TIM  
**Router principale:** UniFi UDR7  
**Router/modem LTE di backup:** TP-Link Archer MR600  
**Versione documento:** 2026-06-19  
**Stato:** configurazione funzionante e testata con ONT fibra spento

---

## 1. Scopo del sistema

Lo scopo del sistema è evitare che la casa rimanga senza connessione Internet quando cade la fibra TIM.

La rete locale deve continuare a funzionare senza modifiche per:

- Home Assistant;
- sensori Shelly / Zigbee / Wi-Fi;
- Tesla e servizi collegati;
- notifiche e accesso remoto via servizi cloud;
- dispositivi domestici già configurati sulla rete `192.168.1.0/24`.

La logica corretta è:

```text
Fibra disponibile  → usa WAN1 fibra TIM
Fibra assente      → passa automaticamente a WAN2 LTE TIM Mobile
Fibra ripristinata → torna automaticamente a WAN1
```

Il backup LTE non sostituisce la fibra come prestazioni assolute, ma consente continuità operativa reale.

---

## 2. Schema generale del sistema

```text
                    ┌─────────────────────┐
                    │  Fibra TIM / ONT     │
                    └──────────┬──────────┘
                               │
                               │ Ethernet
                               ▼
                    ┌─────────────────────┐
                    │ UDR7 - Porta 4       │
                    │ WAN1 primaria TIM    │
                    └──────────┬──────────┘
                               │
                               │
      ┌────────────────────────┴────────────────────────┐
      │                                                 │
      │                  UDR7 / UniFi                   │
      │          Router principale rete casa            │
      │                                                 │
      │ LAN casa: 192.168.1.0/24                         │
      │ DHCP casa: attivo su UDR7                        │
      │                                                 │
      └────────────────────────┬────────────────────────┘
                               │
                               │
                    ┌──────────▼──────────┐
                    │ Rete domestica       │
                    │ HA, Tesla, Shelly,   │
                    │ PC, AP, sensori      │
                    └─────────────────────┘


Backup LTE:

┌────────────────────────────────────────────────────────┐
│ Antenna esterna LTE MIMO doppia                        │
│ Posizione: balcone mansarda                            │
└───────────────┬───────────────────────┬────────────────┘
                │                       │
                │ cavo coassiale 1       │ cavo coassiale 2
                ▼                       ▼
        ┌──────────────────────────────────────┐
        │ TP-Link Archer MR600                  │
        │ SIM TIM                               │
        │ LAN TP-Link: 192.168.50.1             │
        │ DHCP TP-Link: 192.168.50.100-199      │
        └──────────────────┬───────────────────┘
                           │
                           │ Ethernet da porta LAN TP-Link
                           ▼
        ┌──────────────────────────────────────┐
        │ UDR7 - Porta 3                        │
        │ WAN2 / Internet 2 / TIM Mobile        │
        │ IP ricevuto: 192.168.50.100           │
        └──────────────────────────────────────┘
```

---

## 3. Componenti fisici

### 3.1 Linea principale

| Componente | Funzione |
|---|---|
| Fibra TIM | Connessione principale |
| ONT TIM esterno | Conversione fibra → Ethernet |
| UDR7 porta 4 | Ingresso WAN1 principale |

La porta 4 dell'UDR7 resta dedicata alla fibra TIM.

---

### 3.2 Linea di backup LTE

| Componente | Funzione |
|---|---|
| Antenna LTE esterna MIMO doppia | Ricezione segnale mobile TIM |
| 2 cavi coassiali | Collegamento antenna → TP-Link |
| TP-Link Archer MR600 | Router/modem LTE con SIM TIM |
| Cavo Ethernet | Collegamento TP-Link → UDR7 |
| UDR7 porta 3 | Ingresso WAN2 backup |

Il TP-Link non viene usato come router principale della casa.  
Serve come sorgente Internet LTE per la seconda WAN dell'UDR7.

---

## 4. Antenna esterna LTE

### 4.1 Tipo antenna

Dalla descrizione prodotto fornita:

```text
Set antenna LTE MIMO per router 4G
Potenza dichiarata fino a 20 dBi
Bande dichiarate: 700 / 800 / 900 / 1800 / 2100 / 2600 MHz
Range dichiarato: 698-960 MHz e 1710-2700 MHz
Sistema a doppia antenna / MIMO
Cavi coassiali: 2 x 10 m
Compatibilità dichiarata: TP-Link Archer MR600
```

### 4.2 Posizione installata

```text
Antenna posizionata su balcone mansarda.
```

Questa è una buona posizione perché porta l'antenna fuori dalle schermature interne della casa e permette un puntamento più favorevole verso la cella radio.

### 4.3 Collegamento antenna

L'antenna è composta da due elementi MIMO.  
Devono essere collegati entrambi i cavi al TP-Link MR600.

```text
Antenna 1 → connettore LTE 1 TP-Link
Antenna 2 → connettore LTE 2 TP-Link
```

Non usare un solo cavo: si perderebbe il vantaggio MIMO.

### 4.4 Nota tecnica sul guadagno

Il valore commerciale “20 dBi” va preso come dato dichiarato, non come garanzia reale.

I parametri da guardare davvero sono:

```text
RSRP  = potenza segnale ricevuto
RSRQ  = qualità del segnale
SINR  = rapporto segnale / rumore
```

Il parametro più importante per la velocità è spesso il **SINR**, non la sola potenza.

---

## 5. TP-Link Archer MR600

### 5.1 Ruolo del TP-Link

Il TP-Link Archer MR600 lavora come router LTE separato, collegato alla WAN2 dell'UDR7.

Non gestisce la rete domestica principale.

### 5.2 Configurazione LAN TP-Link

Configurazione impostata:

```text
IP LAN TP-Link:       192.168.50.1
Subnet mask:          255.255.255.0
DHCP server:          attivo
Pool DHCP:            192.168.50.100 - 192.168.50.199
Gateway DHCP:         192.168.50.1
```

Il cambio da `192.168.1.1` a `192.168.50.1` era obbligatorio perché la rete principale UniFi usa già `192.168.1.1`.

### 5.3 Perché il DHCP del TP-Link resta attivo

Il DHCP del TP-Link serve solo a dare un indirizzo IP alla WAN2 dell'UDR7.

Nel caso attuale l'UDR7 riceve:

```text
UDR7 WAN2 / Internet 2: 192.168.50.100
Gateway WAN2:           192.168.50.1
```

Questo DHCP non entra nella rete domestica `192.168.1.0/24`, quindi non crea conflitto.

### 5.4 Rete TP-Link separata dalla rete domestica

```text
Rete TP-Link LTE: 192.168.50.0/24
Rete casa UniFi:  192.168.1.0/24
```

Sono due reti diverse.  
La rete `192.168.50.x` è solo una rete tecnica tra TP-Link e UDR7.

### 5.5 Accesso all'interfaccia TP-Link

Per accedere al TP-Link:

```text
http://192.168.50.1
```

Usare `http`, non necessariamente `https`.

Se il PC è collegato direttamente al TP-Link, deve ricevere un IP tipo:

```text
IP PC:      192.168.50.100
Gateway:   192.168.50.1
```

---

## 6. UDR7 / UniFi

### 6.1 Configurazione WAN

Configurazione finale verificata:

```text
UDR7 porta 4 = WAN1 / Internet 1 / TIM fibra
UDR7 porta 3 = WAN2 / Internet 2 / TIM Mobile
UDR7 porta 5 = non assegnata
Modalità WAN = Solo failover
```

### 6.2 Stato visto da UniFi

Dalla schermata UniFi:

```text
Internet 1
Interfaccia: WAN1
ISP: TIM
Porta: 4
IP pubblico: dinamico TIM
Stato: Online

Internet 2
Interfaccia: WAN2
ISP: TIM Mobile
Porta: 3
IP IPv4: 192.168.50.100
Stato: Online
```

### 6.3 Modalità corretta

La modalità corretta è:

```text
Solo failover
```

Non è stato scelto il bilanciamento del carico.

Motivo: lo scopo non è sommare fibra e LTE, ma avere una linea di emergenza stabile quando la fibra cade.

---

## 7. Rete domestica principale

La rete domestica resta invariata:

```text
Subnet casa:     192.168.1.0/24
Gateway casa:    192.168.1.1
DHCP casa:       UDR7
Dispositivi:     Home Assistant, Shelly, Tesla, PC, AP, sensori
```

Questo è il punto fondamentale: il failover cambia solo il percorso verso Internet, non la rete interna.

I dispositivi domestici non devono cambiare IP, gateway o configurazione.

---

## 8. Cablaggio fisico definitivo

### 8.1 Fibra

```text
ONT TIM → UDR7 porta 4
```

### 8.2 Backup LTE

```text
Antenna esterna MIMO su balcone mansarda
↓
2 cavi coassiali verso locale tecnico
↓
TP-Link Archer MR600
↓
cavo Ethernet da porta LAN TP-Link
↓
UDR7 porta 3
```

### 8.3 Porte UDR7

| Porta UDR7 | Uso | Stato |
|---|---|---|
| Porta 1 | LAN / PoE | rete interna |
| Porta 2 | LAN | rete interna |
| Porta 3 | WAN2 | TP-Link MR600 / TIM Mobile |
| Porta 4 | WAN1 | fibra TIM / ONT |
| Porta 5 SFP+ | non assegnata | non usata |

---

## 9. Test effettuati

### 9.1 Test con fibra attiva

Con fibra attiva:

```text
WAN1 TIM fibra = Online
WAN2 TIM Mobile = Online ma in backup
Traffico principale = WAN1
```

### 9.2 Test con ONT spento

Test eseguito spegnendo ONT / fibra.

Risultato:

```text
Fibra assente
UDR7 passa a WAN2
Navigazione Internet funzionante tramite TP-Link LTE
```

Questo conferma che il failover funziona realmente.

### 9.3 Speedtest in failover LTE

Risultato rilevato con TP-Link + antenna esterna:

```text
Download: 86,42 Mbps
Upload:   41,33 Mbps
Ping:     43 ms
```

Valori adatti come backup domestico per Home Assistant, Tesla, notifiche e accesso remoto tramite servizi cloud.

---

## 10. Prestazioni osservate

### 10.1 Prima configurazione LTE senza ottimizzazione antenna

Valori indicativi osservati:

```text
Download: circa 25-34 Mbps
Upload:   circa 40-43 Mbps
Ping:     circa 14-34 ms
```

### 10.2 Dopo antenna esterna

Valori osservati:

```text
Download: 86,42 Mbps
Upload:   41,33 Mbps
Ping:     43 ms
```

La velocità in download è migliorata nettamente.

Il ping resta superiore alla fibra: è normale su LTE.

---

## 11. Cosa funziona durante il failover

Durante il failover LTE devono continuare a funzionare:

- navigazione Internet;
- Home Assistant verso Internet;
- notifiche cloud;
- Nabu Casa, se usato;
- app Tesla e servizi Tesla;
- aggiornamenti e chiamate outbound dei dispositivi;
- sensori e automazioni locali della rete interna.

La rete locale non dipende dalla fibra per comunicare internamente.

---

## 12. Limiti tecnici da conoscere

### 12.1 Doppio NAT

In failover la catena è:

```text
Internet mobile TIM
↓
TP-Link MR600
↓
UDR7
↓
rete casa
```

Questo crea una situazione di doppio NAT.

Per navigare e usare servizi cloud non è un problema serio.  
Per ingressi diretti dall'esterno può esserlo.

### 12.2 CGNAT sulla rete mobile

Le SIM dati/mobile spesso sono dietro CGNAT.

Con CGNAT, il port forwarding diretto dall'esterno verso casa può non funzionare.

Quindi durante il failover LTE non bisogna dare per scontato il funzionamento di:

- port forwarding diretto;
- accessi da IP pubblico alla casa;
- servizi esposti direttamente su porta;
- Tesla Proxy esposto solo sulla WAN fibra.

Servizi basati su connessione uscente, invece, hanno molte più probabilità di funzionare:

- Nabu Casa;
- VPN cloud/outbound;
- app e servizi cloud;
- notifiche push.

### 12.3 Port forwarding Tesla Proxy

Nella configurazione UniFi è presente un port forwarding:

```text
Nome: Tesla Proxy
Porta esterna: 443
Destinazione: 192.168.1.53:4430
Interfaccia: Internet 1 / WAN1
```

Questo forwarding è legato alla WAN fibra.

Durante il failover LTE potrebbe non funzionare, soprattutto per CGNAT mobile.  
Non va considerato garantito sul backup LTE.

---

## 13. Procedura di test periodico

Da eseguire ogni tanto per verificare che il backup sia ancora valido.

### 13.1 Test failover

```text
1. Verificare che WAN1 e WAN2 siano entrambe Online su UniFi.
2. Spegnere temporaneamente ONT fibra TIM.
3. Aspettare 60-120 secondi.
4. Verificare che Internet continui a funzionare.
5. Controllare UniFi → Internet:
   WAN1 non disponibile
   WAN2 Online / Backup attivo
6. Fare uno speedtest.
7. Accendere di nuovo ONT.
8. Aspettare 1-2 minuti.
9. Verificare che il traffico torni su WAN1.
```

### 13.2 Test servizi critici

Durante il test con fibra spenta verificare:

```text
Home Assistant da rete esterna
Nabu Casa / accesso remoto
App Tesla
Notifiche Home Assistant
Navigazione da PC o telefono
```

---

## 14. Procedura in caso di guasto fibra reale

Se la fibra cade:

```text
1. Non toccare la rete interna.
2. Controllare UniFi → Internet.
3. Verificare che WAN2 / TIM Mobile sia Online.
4. Verificare che il TP-Link sia acceso.
5. Verificare che i LED LTE del TP-Link siano presenti.
6. Se Internet non va, controllare il cavo Ethernet TP-Link → UDR7 porta 3.
7. Se necessario, riavviare solo TP-Link MR600.
8. Riavviare UDR7 solo come ultima opzione.
```

---

## 15. Cosa non toccare

Non cambiare questi elementi se il sistema funziona:

```text
UDR7 porta 4 = WAN1 fibra TIM
UDR7 porta 3 = WAN2 TIM Mobile
Modalità WAN = Solo failover
TP-Link LAN IP = 192.168.50.1
TP-Link DHCP = ON
UDR7 LAN casa = 192.168.1.1
DHCP casa = UDR7
```

Non collegare il TP-Link a una porta LAN normale della rete casa con configurazione sbagliata.  
Deve restare collegato alla porta 3 configurata come WAN2.

---

## 16. Diagnostica rapida

### 16.1 WAN2 offline su UniFi

Controllare:

```text
TP-Link acceso
SIM TIM attiva
Cavo Ethernet inserito in porta LAN del TP-Link
Cavo Ethernet inserito in porta 3 UDR7
UDR7 porta 3 configurata come WAN2
TP-Link IP LAN = 192.168.50.1
DHCP TP-Link attivo
```

### 16.2 Non si entra più nel TP-Link

Usare:

```text
http://192.168.50.1
```

Se non risponde, collegare temporaneamente un PC direttamente al TP-Link e verificare con:

```text
ipconfig
```

Il PC deve avere:

```text
IPv4:    192.168.50.xxx
Gateway: 192.168.50.1
```

### 16.3 Internet lento su LTE

Controllare:

```text
RSRP
RSRQ
SINR
Banda LTE agganciata
orientamento antenna
stato cella TIM
congestione oraria
```

Non giudicare solo dalle tacche del segnale.

### 16.4 La fibra torna ma resta su LTE

Controllare:

```text
ONT acceso
porta 4 UDR7 attiva
WAN1 Online
Modalità WAN = Solo failover
SLA WAN su Auto
```

Aspettare 1-2 minuti prima di intervenire.

---

## 17. Considerazioni sull'alimentazione elettrica

Il failover Internet funziona solo se sono alimentati:

```text
UDR7
TP-Link MR600
switch/AP necessari
eventuale alimentatore antenna attiva, se presente
Home Assistant server
```

Se l'obiettivo è continuità reale anche con micro-interruzioni elettriche, questi dispositivi devono stare sotto UPS.

---

## 18. Configurazione finale consolidata

```text
Fibra TIM / ONT
→ UDR7 porta 4
→ WAN1 primaria

Antenna LTE MIMO esterna su balcone mansarda
→ 2 cavi coassiali verso locale tecnico
→ TP-Link Archer MR600 con SIM TIM
→ TP-Link LAN 192.168.50.1
→ cavo Ethernet da LAN TP-Link
→ UDR7 porta 3
→ WAN2 backup TIM Mobile

UDR7
→ rete casa 192.168.1.0/24
→ DHCP principale UDR7
→ Home Assistant / Tesla / Shelly / sensori invariati
```

---

## 19. Esito finale

La configurazione è funzionante.

Prova reale eseguita:

```text
ONT spento
fibra assente
UDR7 ha usato WAN2 LTE
Internet funzionante
Speedtest LTE con antenna esterna: 86,42 Mbps down / 41,33 Mbps up / 43 ms ping
```

Il sistema è quindi idoneo come backup Internet domestico per evitare il blocco operativo di Home Assistant e dei servizi collegati in caso di caduta fibra.
