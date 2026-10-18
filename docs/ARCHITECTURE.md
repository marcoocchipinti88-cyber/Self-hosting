# Architettura e decisioni

Principi: **tutto in casa**, **zero dipendenze cloud**, **sicuro per default**, **tutto ricostruibile da questo repository + i backup**.

## Scelte tecniche

| Livello | Scelta | Perché |
|---|---|---|
| Hypervisor | **Proxmox VE** sui mini PC | Snapshot, backup nativi, cluster (si scala aggiungendo nodi), VM e LXC, ZFS |
| OS delle VM | **Debian stable** (minimale) | Stabilità e sicurezza, aggiornamenti di sicurezza automatici, supporto Docker ufficiale |
| Home Assistant | **HA OS in VM dedicata** | Supervisor, add-on e backup completi (non in un container "core") |
| Container | **Docker Compose** | Semplice, versionabile in git |
| Gestione stack | **Dockge** (e/o Coolify) | Gli stack restano file `compose.yaml` leggibili |
| Accesso esterno | **Pangolin** (WireGuard + SSO) e/o WireGuard su firewall | Nessun accesso senza autenticazione |
| Firewall/rete | **OPNsense** (VM o macchina dedicata) | VLAN, regole, IDS/IPS |
| Backup | **Proxmox Backup Server** + **restic/Kopia** → NAS | Immagini VM e dati applicativi, cifrati |
| Monitoraggio | **Beszel**, **Uptime Kuma**, **Glance** | Rilevare i blocchi subito |
| Notifiche | **ntfy** self-hosted | Allarmi senza servizi cloud |
| Segreti | **SOPS + age** nel repo, **Vaultwarden** | Nessun segreto in chiaro |

## Schema logico

```
Internet ──(solo UDP WireGuard / eventuale 443 verso Pangolin)──┐
                                                               ▼
                                                    [ Firewall OPNsense ]
                         ┌──────────────┬──────────────┬─────────────┐
                    VLAN 10 MGMT   VLAN 20 SERVER   VLAN 30 IoT   VLAN 40 Ospiti
                    Proxmox, NAS   VM Docker,       HA, Alexa,    solo Internet
                    admin          Pangolin,        dispositivi
                                   Coolify, ...     smart

Mini PC #1 (Proxmox)  ──►  VM: docker-core | docker-apps | homeassistant | (OPNsense)
Mini PC #2 (Proxmox)  ──►  replica/failover (cluster con QDevice a 2 nodi)
NAS                    ──►  dati (NFS/SMB), datastore PBS, snapshot, copia off-site cifrata
```

## Punti da decidere (vedi anche ROADMAP, fase 0)

1. **Pangolin** gira di norma su un server con IP pubblico raggiungibile. A casa funziona solo se l'operatore **non usa CGNAT** e puoi aprire le porte. Se c'è CGNAT: serve un IP pubblico, un VPS economico (unica eccezione al "tutto in casa") oppure solo VPN WireGuard verso un router con IP raggiungibile.
2. **Coolify e Pangolin** vogliono entrambi le porte 80/443 (hanno un proxy Traefik proprio). Soluzione: Coolify in una VM separata con il proprio proxy dietro Pangolin, oppure proxy di Coolify disattivato.
3. **Alexa è per natura cloud** (Amazon). Opzioni, dalla più autonoma alla meno:
   - Assistente vocale locale di Home Assistant (Wyoming: Whisper + Piper + openWakeWord) con satelliti ESP32/Raspberry: **nessun cloud**, sostituisce Alexa.
   - Alexa Media Player / emulated Hue: i comandi passano comunque dai server Amazon, ma HA resta in casa.
   - Skill Alexa ufficiale: richiede Nabu Casa o una funzione AWS Lambda tua → è cloud.
   Proposta: partire con Alexa Media Player e in parallelo costruire l'assistente locale.
4. **NAS**: marca/modello e dischi (RAID/ZFS) da confermare. Il RAID **non è un backup**.
5. **Hardware**: quanti mini PC, RAM, NVMe, UPS. Un cluster Proxmox vero richiede 3 nodi (o 2 + QDevice).
6. **Dominio**: un dominio tuo è utile per i certificati HTTPS validi (DNS challenge). Il registrar è un servizio esterno, ma i dati restano in casa.

## Regole di sicurezza (non negoziabili)

- Nessun servizio esposto direttamente: tutto passa da Pangolin/VPN con SSO e 2FA.
- Il pannello di Proxmox, il NAS e il firewall **mai** raggiungibili da Internet.
- Utenti nominali, chiavi SSH al posto delle password, root disabilitato via SSH.
- Immagini Docker con versione fissata (mai `latest`), scansionate con Trivy; utente non-root, `read_only`, `cap_drop: [ALL]` dove possibile.
- Docker socket mai esposto alle app (usare socket-proxy).
- Segreti solo cifrati (SOPS) o in Vaultwarden; `.env` mai in git.
- Backup 3-2-1, cifrati, con **test di ripristino periodico automatico**.
- Ogni modifica passa da git (commit = documentazione + rollback).
