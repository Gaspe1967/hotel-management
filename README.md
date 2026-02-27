
# HotelSuite — Guida Installazione LXC Proxmox

## Architettura

```
Proxmox Host
└── Container LXC 100
    ├── Ubuntu 22.04 LTS
    ├── Node.js 20 LTS  (build React)
    ├── nginx            (serve app statica)
    └── HotelSuite       → http://<IP-CT100>
```

---

## 1. Prerequisiti sul HOST Proxmox

### 1a. Crea il container LXC 100 (se non esiste)
Accedi alla shell del host Proxmox ed esegui:

```bash
# Scarica template Ubuntu 22.04
pveam update
pveam download local ubuntu-22.04-standard_22.04-1_amd64.tar.zst

# Crea container 100
pct create 100 local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
  --hostname hotelsuite \
  --memory 1024 \
  --cores 2 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --password TuaPasswordSicura123! \
  --unprivileged 1 \
  --start 1
```

> **Nota:** modifica `local-lvm` con il tuo storage, e imposta la password che preferisci.

---

## 2. Installazione HotelSuite

### 2a. Carica i file sul host Proxmox

Copia la cartella `hotelsuite/` sul host Proxmox (via SCP, USB, o SFTP):

```bash
scp -r ./hotelsuite root@<IP-PROXMOX>:/root/
```

### 2b. Pacchettizza l'app

Dalla cartella `hotelsuite/` sul host Proxmox, crea il tar:

```bash
cd /root/hotelsuite
tar -czf hotelsuite.tar.gz src/ public/ nginx/ systemd/ package.json vite.config.js index.html
```

### 2c. Lancia lo script di installazione

```bash
cd /root/hotelsuite
chmod +x install-proxmox.sh setup-container.sh
bash install-proxmox.sh
```

Lo script in automatico:
- Avvia il container 100
- Copia i file nell'LXC
- Installa Node.js 20 + npm
- Esegue `npm install` e `npm run build`
- Configura nginx come web server
- Avvia il servizio

---

## 3. Accesso all'applicazione

Dopo l'installazione, recupera l'IP del container:

```bash
pct exec 100 -- hostname -I
```

Apri nel browser:
```
http://<IP-CONTAINER-100>
```

### Accesso da rete locale
Per accedere dall'esterno della rete Proxmox, puoi configurare un **port forward** sul tuo router verso `<IP-CONTAINER>:80`, oppure usare il reverse proxy di Proxmox.

---

## 4. IP Statico (consigliato)

Per assegnare un IP fisso al container, modifica la rete da Proxmox:

```bash
pct set 100 --net0 name=eth0,bridge=vmbr0,ip=192.168.1.200/24,gw=192.168.1.1
pct reboot 100
```

Poi l'app sarà sempre su: `http://192.168.1.200`

---

## 5. Comandi utili

```bash
# Stato container
pct status 100

# Console interattiva
pct console 100

# Avvia / Ferma / Riavvia
pct start 100
pct stop 100
pct reboot 100

# Log nginx in tempo reale
pct exec 100 -- journalctl -u nginx -f

# Aggiornare l'app (dopo aver copiato nuovi file)
pct exec 100 -- hotelsuite-update
```

---

## 6. Struttura file nel container

```
/opt/hotelsuite/          ← sorgenti app (React + build tools)
├── src/App.jsx
├── package.json
├── vite.config.js
└── dist/                 ← build produzione

/var/www/hotelsuite/      ← file serviti da nginx (copia di dist/)
/etc/nginx/sites-available/hotelsuite.conf
/usr/local/bin/hotelsuite-update
```

---

## 7. Aggiornamento app

Quando vuoi aggiornare l'app con una nuova versione:

```bash
# 1. Copia i nuovi sorgenti
scp -r ./hotelsuite/src root@<IP-PROXMOX>:/opt/hotelsuite/

# 2. Lancia l'aggiornamento dal container
pct exec 100 -- hotelsuite-update
```

---

## 8. Backup container

```bash
# Backup snapshot
vzdump 100 --storage local --mode snapshot

# Ripristino
pct restore 100 /var/lib/vz/dump/vzdump-lxc-100-*.tar.zst --storage local-lvm
```

---

## Requisiti minimi container

| Risorsa | Minimo | Consigliato |
|---------|--------|-------------|
| RAM     | 512 MB | 1024 MB     |
| CPU     | 1 core | 2 core      |
| Disco   | 4 GB   | 8 GB        |
| OS      | Ubuntu 22.04 | Ubuntu 22.04 |

---

*HotelSuite v1.0 — Hotel Bellavista*
