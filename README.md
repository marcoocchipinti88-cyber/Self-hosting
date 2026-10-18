# Self-hosting

Homelab domestico, autonomo dai servizi cloud: mini PC + NAS, Proxmox, Docker, Home Assistant, accesso esterno solo tramite VPN/SSO.

## Documenti
- [Architettura e decisioni](docs/ARCHITECTURE.md)
- [Roadmap operativa con task da spuntare](docs/ROADMAP.md)

## Stato
Fase 0 — decisioni e inventario. Dati raccolti: Vodafone dietro NAT, UDM-Pro, 1 mini PC, UPS, dominio gratuito. Da fare: scegliere il NAS e confermare il CGNAT.

## Regole
- Tutto in git, nessun segreto in chiaro (vedi `.gitignore`).
- Immagini con versione fissa, backup 3-2-1 cifrati, ripristino testato.
