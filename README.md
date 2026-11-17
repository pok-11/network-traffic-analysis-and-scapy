# Network Traffic Analysis, Port Scanning & Packet Crafting (Scapy)

Laboratorio pratico di analisi del traffico di rete e sicurezza difensiva/offensiva svolto in ambiente isolato (Kali Linux + target Windows).

## Obiettivi e Strumenti Utilizzati
* **Packet Sniffing & Analisi:** `tcpdump` (filtri BPF) e `Wireshark` (Display Filters, Follow TCP Stream, Protocol Hierarchy)
* **Network Scanning:** `Nmap` (analisi a livello di pacchetto di 9 tecniche di scansione)
* **Packet Crafting:** `Python 3` + libreria `Scapy` per costruire pacchetti raw TCP/IP e ARP
* **Attacchi Layer 2:** Simulazione controllata di `ARP Spoofing` (Man-in-the-Middle) con `arpspoof` ed `Ettercap`

---

## 1. Analisi del Traffico (tcpdump & Wireshark)
Cattura del traffico su interfaccia di rete e filtraggio tramite sintassi BPF (es. `tcpdump -i eth0 host TARGET_IP and port 80 -w capture.pcap`).
* **Ispezione Hex Dump (`-X`):** L'analisi a basso livello evidenzia come i protocolli non cifrati (es. HTTP) espongano payload, intestazioni e credenziali in chiaro, confermando la necessità di cifratura a livello di trasporto/presentazione (TLS/HTTPS).
* **Wireshark:** Ricostruzione delle sessioni tramite *Follow TCP Stream* e verifica della distribuzione dei protocolli tramite *Protocol Hierarchy*.

---

## 2. Confronto Tecniche di Scansione Nmap

| Tecnica | Flag Nmap | Flag TCP Inviati | Risposta Porta Aperta | Risposta Porta Chiusa | Affidabile su Windows? |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **SYN Scan (Half-Open)** | `-sS` | `SYN` | `SYN-ACK` (poi `RST`) | `RST-ACK` | Sì |
| **TCP Connect** | `-sT` | `SYN` | `SYN-ACK` $\rightarrow$ `ACK` | `RST-ACK` | Sì |
| **ACK Scan** | `-sA` | `ACK` | `RST` (unfiltered) | `RST` (unfiltered) | Sì (mappa firewall) |
| **Window Scan** | `-sW` | `ACK` | `RST` (Window > 0) | `RST` (Window = 0) | No |
| **Maimon Scan** | `-sM` | `FIN + ACK` | Nessuna (RFC 793) | `RST` | No |
| **Null Scan** | `-sN` | Nessuno (`0x00`) | Nessuna (RFC 793) | `RST` | No |
| **FIN Scan** | `-sF` | `FIN` | Nessuna (RFC 793) | `RST` | No |
| **Xmas Scan** | `-sX` | `FIN + PSH + URG` | Nessuna (RFC 793) | `RST` | No |
| **UDP Scan** | `-sU` | Pacchetto UDP | Risposta UDP / Silenzio | ICMP Port Unreachable | Sì (lenta) |

### Perché Null, FIN, Xmas e Maimon "falliscono" su Windows?
Secondo la **RFC 793**, una porta chiusa risponde con `RST` a pacchetti anomali (senza flag o con `FIN/PSH/URG`), mentre una porta aperta resta silenziosa. I sistemi Unix/Linux rispettano questo standard, mentre lo stack TCP/IP di **Windows risponde sempre con un `RST`** indipendentemente dallo stato della porta, facendo apparire tutte le porte come `closed`.

### Scelta della tecnica Stealth & Firewall Stateful
* **Massima stealthness:** La **SYN Scan (`-sS`)** resta il miglior compromesso perché non completa il Three-Way Handshake (evitando i log applicativi di livello 7), mentre su target Linux/Unix le scansioni **FIN (`-sF`)** o **Null (`-sN`)** risultano ancora meno rumorose.
* **Dietro un Firewall Stateful:** Le scansioni *Null*, *FIN* e *Xmas* vengono bloccate o droppate istantaneamente perché il firewall non trova una sessione `ESTABLISHED` nella propria **State Table** associata a quei flag anomali. La **SYN Scan (`-sS`)** è l'unica tra le tecniche stealth a passare, poiché simula l'inizio legittimo di una nuova connessione.

---

## 3. Packet Crafting con Scapy (`/scripts`)
Ricostruzione manuale dei pacchetti tramite Python per osservare il comportamento dello stack di rete:
* `syn_probe.py`: Invio di un singolo pacchetto `IP()/TCP(flags="S")` e lettura dei flag di risposta (`0x12` = `SYN-ACK`, `0x14` = `RST-ACK`).
* `mini_syn_scanner.py`: Automazione su range di porte con invio immediato di un pacchetto `RST` per chiudere la mezza connessione.
* `xmas_probe.py`: Verifica pratica della RFC 793 inviando `flags="FPU"` contro target Windows e Linux.
* `arp_sniffer.py`: Sniffer passivo che intercetta pacchetti ARP (`op == 1`, *who-has*) sulla rete locale.

---

## 4. Simulazione ARP Spoofing (Layer 2 MitM)
Test in ambiente controllato per intercettare il traffico tra un client Windows e il Default Gateway:
1. Attivazione dell'**IP Forwarding** su Kali (`sysctl -w net.ipv4.ip_forward=1`) per evitare di causare un Denial of Service e garantire l'inoltro dei pacchetti.
2. Avvelenamento bidirezionale delle ARP Cache tramite `arpspoof` ed `Ettercap` (invio di *Gratuitous ARP Reply* falsificati) e verifica tramite `arp -a` sul target.
3. **Mitigazione:** Implementazione di **Dynamic ARP Inspection (DAI)** e **DHCP Snooping** sugli switch aziendali.

---
*Disclaimer: Attività svolte esclusivamente in laboratorio virtuale isolato a scopo didattico e difensivo.*
