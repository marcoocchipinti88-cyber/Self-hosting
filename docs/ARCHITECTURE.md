# Architettura e decisioni

Principi: **tutto in casa**, **zero dipendenze cloud**, **sicuro per default**, **tutto ricostruibile da questo repository + i backup**.

## Dati di partenza (confermati)
- Provider: **Vodafone, senza IP statico, dietro NAT** (probabile CGNAT: da confermare, vedi sotto)
- Firewall/router a monte: **UniFi UDM-Pro**
- Server: **1 mini PC nuovo** (si parte con un nodo, poi si scala)
- NAS: da scegliere tra quelli disponibili
- **UPS** presente
- Dominio: **gratuito**

## Scelte tecniche

| Livello | Scelta | Perché |
|---|---|---|
| Firewall/rete | **UDM-Pro** (VLAN, firewall, IDS/IPS, WireGuard client/server) | Già previsto; sostituisce OPNsense, un componente in meno da gestire |
| Hypervisor | **Proxmox VE** sul mini PC | Snapshot, backup nativi, cluster (si scala aggiungendo nodi), VM e LXC, ZFS |
| OS delle VM | **Debian stable** (minimale) | Stabilità e sicurezza, aggiornamenti di sicurezza automatici, supporto Docker ufficiale |
| Home Assistant | **HA OS in VM dedicata** | Supervisor, add-on e backup completi (non in un container "core") |
| Container | **Docker Compose** | Semplice, versionabile in git |
| Gestione stack | **Dockge** (e/o Coolify) | Gli stack restano file `compose.yaml` leggibili |
| Accesso esterno | **Pangolin** su un VPS "porta d'ingresso" + tunnel WireGuard in uscita da casa | Funziona dietro CGNAT, nessuna porta aperta in casa |
| DNS interno | **AdGuard Home** | Nomi host locali e blocco tracker |
| Backup | **Proxmox Backup Server** + **restic/Kopia** → NAS | Immagini VM e dati applicativi, cifrati |
| Monitoraggio | **Beszel**, **Uptime Kuma**, **Glance** | Rilevare i blocchi subito |
| Notifiche | **ntfy** self-hosted | Allarmi senza servizi cloud |
| Segreti | **SOPS + age** nel repo, **Vaultwarden** | Nessun segreto in chiaro |

## Schema logico

```
Telefono/PC fuori casa
        │  (WireGuard / HTTPS con SSO + 2FA)
        ▼
[ VPS minimo con IP pubblico: Pangolin + Traefik ]   ← unico pezzo fuori casa, non contiene dati
        ▲
        │  tunnel WireGuard aperto DA CASA verso il VPS (connessione in uscita, funziona con CGNAT)
        │
Internet ── Vodafone (NAT) ── [ UDM-Pro ] ── nessuna porta aperta in ingresso
                     ┌──────────────┬──────────────┬─────────────┐
                VLAN 10 MGMT   VLAN 20 SERVER   VLAN 30 IoT   VLAN 40 Ospiti
                Proxmox, NAS   VM Docker,       HA, Alexa,    solo Internet
                admin          Newt (agent),    dispositivi
                               Coolify, ...     smart

Mini PC #1 (Proxmox)  ──►  VM: docker-core | docker-apps | homeassistant
Mini PC #2/#3 (futuri) ──►  cluster / failover
NAS                    ──►  dati (NFS/SMB), datastore PBS, snapshot, copia off-site cifrata
```

## Accesso esterno dietro NAT: decisione

Con Vodafone senza IP statico e dietro NAT **non si può aprire una porta sul router**. Le strade possibili:

| # | Soluzione | Autonomia | Note |
|---|---|---|---|
| A | **Verificare se l'IP è davvero in CGNAT** e, se non lo è, usare DDNS + WireGuard sulla UDM-Pro | Massima | Dipende dalla linea; Vodafone a volte offre IP pubblico dedicato su richiesta (da verificare, non garantito) |
| B | **VPS minimo (pochi €/mese) con Pangolin**, tunnel in uscita da casa | Alta: il VPS inoltra e basta, i dati restano in casa | **Consigliata**: è proprio il caso d'uso di Pangolin, già scelto |
| C | **Tailscale** | Media: il coordinamento passa dal servizio di Tailscale | Molto semplice, ma è un servizio cloud |
| D | **Headscale** (Tailscale self-hosted) | Alta | Anche questo richiede un VPS con IP pubblico |
| E | IPv6 nativo | Variabile | Funziona solo se linea e rete mobile hanno entrambe IPv6: poco affidabile |

**Piano:** prima il test A (5 minuti). Se c'è CGNAT → soluzione B. Il VPS è l'**unica eccezione** al "tutto in casa": non ospita dati, solo il punto d'ingresso cifrato, ed è sostituibile in pochi minuti.
Per l'accesso a tutta la LAN (non solo ai siti web) si può in aggiunta far collegare la UDM-Pro al VPS come client WireGuard (funzione WireGuard client nelle versioni recenti di UniFi Network: da verificare sulla versione installata).

## Altri punti da decidere

1. **Coolify e Pangolin** vogliono entrambi le porte 80/443 (hanno un proxy Traefik proprio). Soluzione: Coolify in una VM separata con il proprio proxy dietro Pangolin, oppure proxy di Coolify disattivato.
2. **Alexa è per natura cloud** (Amazon). Opzioni, dalla più autonoma alla meno:
   - Assistente vocale locale di Home Assistant (Whisper + Piper + openWakeWord) con satelliti ESP32/Raspberry: **nessun cloud**, sostituisce Alexa.
   - Alexa Media Player / emulated Hue: i comandi passano comunque dai server Amazon, ma HA resta in casa.
   - Skill Alexa ufficiale: richiede Nabu Casa o una funzione AWS Lambda tua → è cloud.
   Proposta: partire con Alexa Media Player e in parallelo costruire l'assistente locale.
3. **NAS**: serve scegliere quale usare (capacità, dischi, RAID/ZFS, supporto NFS/iSCSI, snapshot). Il RAID **non è un backup**.
4. **Dominio gratuito**: DuckDNS (sottodomini wildcard, certificati HTTPS con DNS challenge) è la scelta più pratica. Limiti: nome del tipo `tuonome.duckdns.org`, pochi tipi di record, dipendi dal provider. In futuro si può passare a un dominio a pagamento cambiando solo la configurazione.
5. **Mini PC**: CPU/RAM/NVMe da definire. Indicazione: almeno 32 GB di RAM e 2 NVMe (o NVMe + SSD) per ZFS e crescita.
6. **UDM-Pro e cloud Ubiquiti**: usarla con account **locale**, senza accesso remoto tramite il cloud Ubiquiti (Teleport/remote access), per restare autonomi.

## Regole di sicurezza (non negoziabili)

- Nessuna porta aperta in ingresso in casa: l'accesso esterno passa solo dal tunnel in uscita, con SSO e 2FA.
- Il pannello di Proxmox, il NAS e la UDM-Pro **mai** raggiungibili da Internet.
- Utenti nominali, chiavi SSH al posto delle password, root disabilitato via SSH.
- Immagini Docker con versione fissata (mai `latest`), scansionate con Trivy; utente non-root, `read_only`, `cap_drop: [ALL]` dove possibile.
- Docker socket mai esposto alle app (usare socket-proxy).
- Segreti solo cifrati (SOPS) o in Vaultwarden; `.env` mai in git.
- Backup 3-2-1, cifrati, con **test di ripristino periodico automatico**.
- Ogni modifica passa da git (commit = documentazione + rollback).
