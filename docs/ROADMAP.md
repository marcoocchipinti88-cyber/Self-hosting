# Roadmap operativa

Spunta `[x]` quando un task è completato **e verificato**. Ogni fase ha una *Definition of Done* (DoD): non si passa alla successiva finché non è soddisfatta.
Le fasi 0–5 sono la base obbligatoria; dalla 6 in poi si scala a piacere.

Legenda: 🔒 sicurezza · 💾 backup/ripristino · 📈 scalabilità · 🔔 monitoraggio

---

## Fase 0 — Decisioni e inventario
- [x] 0.0 Dati di partenza raccolti: Vodafone senza IP statico dietro NAT, UDM-Pro, 1 mini PC, UPS presente, dominio gratuito
- [ ] 0.1 Elencare hardware: specifiche del mini PC (CPU/RAM/NVMe), **scegliere il NAS** (modello, dischi, RAID/ZFS), switch
- [ ] 0.2 Confermare il **CGNAT**: confrontare l'IP WAN mostrato dalla UDM-Pro con l'IP visto da Internet (se diversi, o se inizia con 100.64–100.127, c'è CGNAT); chiedere a Vodafone se offre IP pubblico
- [ ] 0.3 Confermare la soluzione di accesso esterno (vedi ARCHITECTURE: consigliata B, VPS con Pangolin)
- [ ] 0.4 Decidere strategia Alexa (vedi ARCHITECTURE, punto 2)
- [ ] 0.5 Scegliere il provider del dominio gratuito (proposta: DuckDNS) e registrare il nome
- [ ] 0.6 Definire piano IP e VLAN (MGMT, SERVER, IoT, OSPITI)
- [ ] 0.7 Elencare i servizi desiderati, in ordine di priorità (da awesome-selfhosted)
- [ ] 0.8 Definire RPO/RTO: quanti dati posso perdere (es. 24 h) e in quanto tempo devo ripartire (es. 2 h)

**DoD:** documento hardware + schema di rete + RPO/RTO scritti in `docs/`.

## Fase 1 — Rete e firewall 🔒
- [ ] 1.1 Configurare la **UDM-Pro**: account locale, aggiornare il firmware, disattivare l'accesso remoto via cloud Ubiquiti, attivare IDS/IPS
- [ ] 1.2 Creare le VLAN e le regole: IoT e Ospiti non raggiungono MGMT/SERVER
- [ ] 1.3 DNS locale con **AdGuard Home** o Unbound (nome host interni, blocco tracker)
- [ ] 1.4 DHCP con prenotazioni IP fisse per server e NAS
- [ ] 1.5 Regola per far funzionare mDNS/discovery tra IoT e Home Assistant (mDNS della UDM-Pro, solo ciò che serve)
- [ ] 1.6 Aggiornamenti firmware automatici/pianificati su router, switch, AP
- [ ] 1.7 Configurazione del firewall esportata e salvata nei backup

**DoD:** da una rete Ospiti non si raggiunge nessun server; il DNS interno risolve i nomi.

## Fase 2 — Hypervisor e NAS 💾📈
- [ ] 2.1 Installare **Proxmox VE** sul mini PC #1 (ZFS sull'NVMe, se la RAM lo permette)
- [ ] 2.2 Hardening Proxmox: utente nominale + 2FA, SSH solo con chiave, firewall attivo, nessun accesso da WAN
- [ ] 2.3 Configurare il NAS: pool/volumi, utenti, 2FA, snapshot ravvicinati, share NFS/SMB per VLAN SERVER
- [ ] 2.4 Collegare lo storage del NAS a Proxmox
- [ ] 2.5 Installare **Proxmox Backup Server** (datastore su NAS o disco dedicato; **cifratura attiva**)
- [ ] 2.6 Job di backup notturni di tutte le VM + verifica periodica (verify job) + prune
- [ ] 2.7 **Test di ripristino reale** di una VM di prova (tempo misurato)
- [ ] 2.8 UPS collegato con spegnimento ordinato automatico (NUT)
- [ ] 2.9 (Quando disponibile) secondo mini PC → cluster con QDevice, replica ZFS/HA

**DoD:** una VM cancellata viene ripristinata da PBS entro l'RTO stabilito.

## Fase 3 — Base delle VM Docker 🔒
- [ ] 3.1 Template VM **Debian stable** minimale (cloud-init) per clonare nuove VM in minuti
- [ ] 3.2 Hardening del template: utente non-root, SSH a chiave, `ufw`/nftables, `unattended-upgrades`, `fail2ban`/CrowdSec
- [ ] 3.3 Installare Docker Engine + Compose dal repository ufficiale; log con rotazione (`json-file` max-size)
- [ ] 3.4 Creare VM `docker-core` (reverse proxy, auth, monitoraggio) e `docker-apps` (applicazioni)
- [ ] 3.5 Struttura repo: `compose/<servizio>/compose.yaml`, `.env.example`, `secrets/` cifrato con **SOPS+age**
- [ ] 3.6 Installare **Dockge** per gestire gli stack
- [ ] 3.7 Installare **docker-socket-proxy** (nessuna app accede direttamente a `/var/run/docker.sock`)
- [ ] 3.8 Regole standard nei compose: versione immagine fissa, `restart: unless-stopped`, `healthcheck`, limiti CPU/RAM, `read_only`, `cap_drop`
- [ ] 3.9 Pipeline di scansione immagini con **Trivy** (plugin Docker Image Scan) prima di ogni nuovo servizio

**DoD:** creo una VM nuova dal template e metto in piedi uno stack di prova in meno di 15 minuti.

## Fase 4 — Accesso esterno sicuro 🔒
- [ ] 4.0 Se la fase 0.2 conferma il CGNAT: prendere un **VPS minimo** con IP pubblico (UE, ~€3–5/mese), con hardening di base (chiavi SSH, firewall, aggiornamenti automatici)
- [ ] 4.1 Installare **Pangolin** (con Traefik e Gerbil) sul VPS
- [ ] 4.2 Puntare il dominio gratuito al VPS; certificati HTTPS validi (DNS challenge, wildcard)
- [ ] 4.3 SSO con **2FA obbligatoria** per tutti gli utenti
- [ ] 4.4 Installare l'agent **Newt** nella VM `docker-core` per collegare i servizi di casa al VPS (tunnel in uscita)
- [ ] 4.5 Nessuna porta aperta in ingresso sulla UDM-Pro; opzionale: UDM-Pro come client WireGuard verso il VPS per raggiungere tutta la LAN
- [ ] 4.6 **CrowdSec** (o equivalente) davanti al proxy, con blocco automatico degli IP ostili
- [ ] 4.7 Configurare client VPN (telefono/PC) e testare l'accesso da rete mobile
- [ ] 4.8 Test dall'esterno: scansione porte, nessun servizio raggiungibile senza login
- [ ] 4.9 Decidere il ruolo di **Coolify** rispetto a Pangolin (VM separata, proxy dietro Pangolin)

**DoD:** da fuori casa raggiungo i servizi solo dopo autenticazione 2FA; una scansione esterna mostra solo le porte previste.

## Fase 5 — Monitoraggio, allarmi e auto-ripristino 🔔💾
- [ ] 5.1 **ntfy** self-hosted per le notifiche push sul telefono
- [ ] 5.2 **Uptime Kuma**: controlli HTTP/TCP/ping su ogni servizio, allarme via ntfy
- [ ] 5.3 **Beszel** (CPU/RAM/disco/Docker) su ogni VM e sul NAS
- [ ] 5.4 **Glance** come dashboard unica (stato servizi, metriche, feed)
- [ ] 5.5 Auto-riavvio dei container "unhealthy" (`autoheal` o equivalente)
- [ ] 5.6 Watchdog di Proxmox: riavvio automatico di una VM bloccata (HA / `qm` watchdog)
- [ ] 5.7 Allarmi su: disco >85%, backup fallito, certificato in scadenza, SMART del disco, batteria UPS
- [ ] 5.8 Aggiornamenti controllati: **Renovate/Diun** propone, io approvo; rollback con un `git revert`
- [ ] 5.9 **Backup applicativi** (database con dump, volumi) con **restic/Kopia** → NAS, cifrati
- [ ] 5.10 **Script di auto-ripristino**: ricrea lo stack da git + ripristina dati dall'ultimo backup (`scripts/restore.sh`)
- [ ] 5.11 **Test di ripristino automatico mensile** in una VM temporanea, con esito notificato
- [ ] 5.12 Copia **off-site** cifrata (disco esterno a rotazione o NAS in altra sede) → regola 3-2-1
- [ ] 5.13 Runbook (`docs/runbooks/`): cosa fare se cade internet, la corrente, un disco, un nodo, il firewall

**DoD:** spengo un servizio di proposito → arriva l'allarme sul telefono in <2 minuti e il servizio riparte da solo; il test di ripristino mensile ha esito positivo.

## Fase 6 — Home Assistant 🏠
- [ ] 6.1 VM **Home Assistant OS** (VLAN IoT con accesso controllato), backup PBS + backup interni di HA su NAS
- [ ] 6.2 Dongle USB Zigbee/Z-Wave in passthrough alla VM (preferire Zigbee2MQTT/ZHA, protocolli locali)
- [ ] 6.3 **Mosquitto** (MQTT) con utenti e TLS
- [ ] 6.4 Integrazioni locali prima di quelle cloud (Matter/Thread, Zigbee, ESPHome)
- [ ] 6.5 Accesso a HA solo tramite Pangolin/VPN con 2FA
- [ ] 6.6 Alexa: implementare la strategia scelta in 0.4
- [ ] 6.7 Assistente vocale locale (Whisper + Piper + openWakeWord) e satellite di prova
- [ ] 6.8 Automazioni critiche funzionanti **anche senza Internet**; test staccando il WAN
- [ ] 6.9 Allarmi HA (batterie sensori, dispositivi offline) verso ntfy

**DoD:** con Internet scollegato le luci, le automazioni e l'antifurto funzionano ancora.

## Fase 7 — Servizi base da awesome-selfhosted 📈
Ordine consigliato (adatta in base alla fase 0.7). Per ogni servizio: compose, versione fissa, healthcheck, SSO, backup, voce in Uptime Kuma e Glance.
- [ ] 7.1 **Vaultwarden** (password manager)
- [ ] 7.2 **Nextcloud** o **Immich** (file / foto)
- [ ] 7.3 **Paperless-ngx** (documenti)
- [ ] 7.4 **Jellyfin** (media) + eventuale gestione download
- [ ] 7.5 **Gitea/Forgejo** (repository locali) e registry Docker locale
- [ ] 7.6 **Syncthing** o simile per sincronizzazione
- [ ] 7.7 **Authentik** (se serve un provider OIDC più ricco del SSO di Pangolin)
- [ ] 7.8 Wiki/note (Outline, BookStack) e altro dal catalogo

## Fase 8 — Scalabilità 📈
- [ ] 8.1 Secondo e terzo nodo Proxmox (cluster, migrazione live, HA)
- [ ] 8.2 Provisioning con **Terraform/OpenTofu + Ansible** (plugin IaC Reviewer per le verifiche)
- [ ] 8.3 GitOps: un push in `main` aggiorna gli stack (Coolify/Komodo o Ansible)
- [ ] 8.4 Valutare Docker Swarm / k3s solo se serve davvero (non prima)
- [ ] 8.5 Rete a 2.5/10 GbE tra nodi e NAS se i trasferimenti diventano il collo di bottiglia

## Fase 9 — Verifica continua 🔒
- [ ] 9.1 Audit di sicurezza trimestrale (plugin OWASP Security Audit, Trivy sulle immagini)
- [ ] 9.2 Revisione account, chiavi e accessi VPN
- [ ] 9.3 Aggiornamento di documentazione e runbook
- [ ] 9.4 Simulazione di disastro annuale: ricostruzione completa da zero con soli backup + repo

---

## Come lavoriamo
- Un task alla volta, in ordine; ogni task chiuso = commit con la spunta aggiornata.
- Prima di ogni modifica rischiosa: snapshot della VM.
- Nessun task è "fatto" senza prova (comando, schermata o test descritto nel commit).
