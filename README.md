# Bloc 2 — Serveis de Xarxa i Internet (0375)

## Índex
1. [Preparació del servidor](#1-preparació-del-servidor)
2. [Servei d'àudio — Icecast2](#2-servei-daudio--icecast2)
3. [Servei de vídeo — NGINX-RTMP](#3-servei-de-vídeo--nginx-rtmp)
4. [Videoconferència — Jitsi Meet](#4-videoconferència--jitsi-meet)
5. [Proves d'amplada de banda](#5-proves-damplada-de-banda)

---

## 1. Preparació del servidor

### 1.1 Descripció de la màquina

La instància `innovatetech-media` (srv4) és el servidor dedicat
als serveis multimèdia del projecte InnovateTech.

| Paràmetre | Valor |
|-----------|-------|
| Nom | innovatetech-media |
| Tipus EC2 | t2.medium |
| RAM | 4 GB |
| Disc | 15 GB gp3 |
| Sistema operatiu | Ubuntu Server 22.04 LTS |
| IP pública | 54.157.67.55 |
| Regió AWS | eu-west-1 (Irlanda) |

![Instància EC2 en estat running](captures/01-ec2-running.png)

### 1.2 Usuari d'administració

El projecte exigeix no utilitzar l'usuari per defecte (`ubuntu`).
S'ha creat l'usuari específic `innovatech-admin` amb accés
exclusivament per clau pública/privada.

```bash
sudo useradd -m -s /bin/bash innovatech-admin
sudo usermod -aG sudo innovatech-admin
sudo mkdir -p /home/innovatech-admin/.ssh
sudo cp /home/ubuntu/.ssh/authorized_keys \
  /home/innovatech-admin/.ssh/
sudo chown -R innovatech-admin:innovatech-admin \
  /home/innovatech-admin/.ssh
sudo chmod 700 /home/innovatech-admin/.ssh
sudo chmod 600 /home/innovatech-admin/.ssh/authorized_keys
sudo passwd -l ubuntu
```

![Connexió SSH amb innovatech-admin](captures/02-ssh-admin.png)

### 1.3 Seguretat de xarxa

| Port | Protocol | Servei | Accessible des de |
|------|----------|--------|-------------------|
| 22 | TCP | SSH | IP de l'admin |
| 8000 | TCP | Icecast2 | Internet |
| 8080 | TCP | NGINX-RTMP HLS | Internet |
| 1935 | TCP | RTMP | Internet |
| 443 | TCP | Jitsi HTTPS | Internet |
| 4443 | TCP | Jitsi Videobridge | Internet |
| 10000 | UDP | Jitsi WebRTC | Internet |

![Security Group AWS](captures/03-security-group.png)
![UFW ports oberts](captures/04-ufw-status.png)

---

## 2. Servei d'àudio — Icecast2

### 2.1 Descripció del servei

Icecast2 és un servidor de streaming d'àudio de codi obert que
permet distribuir contingut d'àudio en temps real a múltiples
clients simultàniament. S'ha triat Icecast2 perquè és l'estàndard
del sector per a streaming d'àudio en entorns empresarials, és
lleuger en consum de recursos i permet configurar múltiples canals
independents amb formats diferenciats.

A InnovateTech s'utilitza per cobrir dues necessitats:
- Distribució de contingut corporatiu intern en format MP3.
- Emissions de sessions de formació interna en format OGG.

### 2.2 Instal·lació

```bash
sudo apt install icecast2 -y
```

Durant la instal·lació l'assistent demana:

| Paràmetre | Valor |
|-----------|-------|
| Hostname | audio.innovatetech.local |
| Source password | @ITB2026 |
| Relay password | @ITB2026 |
| Admin password | @ITB2026 |

![Assistent configuració Icecast2](captures/05-icecast2-assistent.png)
![Icecast2 instal·lat](captures/06-icecast2-instalat.png)

### 2.3 Configuració

El fitxer de configuració principal és `/etc/icecast2/icecast.xml`.

**Canals configurats:**

| Canal | Muntatge | Format | Bitrate | Ús |
|-------|----------|--------|---------|-----|
| Corporatiu | `/corporate` | MP3 | 128 kbps | Comunicació interna |
| Formació | `/formacio` | OGG Vorbis | 96 kbps | Sessions de formació |

El canal `/corporate` utilitza MP3 perquè és el format d'àudio
digital més universal, compatible amb qualsevol navegador i client.

El canal `/formacio` utilitza OGG Vorbis perquè és un format
lliure i obert que ofereix millor qualitat de so que MP3 al mateix
bitrate, especialment en la reproducció de veu humana.

![Fitxer de configuració icecast.xml](captures/07-icecast2-config.png)

### 2.4 Font d'àudio amb ffmpeg

Com que la instància EC2 no té targeta de so física, s'ha
utilitzat ffmpeg com a font d'àudio virtual. Ffmpeg llegeix
fitxers d'àudio pregravats i els publica en bucle continu
a Icecast2, simulant una emissió en directe.

Fitxers d'àudio preparats:
- `corporate.mp3`: canal corporatiu, emès a 128 kbps en MP3.
- `formacio.ogg`: canal de formació, emès a 96 kbps en OGG Vorbis.

![Fitxers d'àudio al servidor](captures/08-audio-fitxers.png)

S'han creat dos serveis systemd per garantir l'inici automàtic:

```ini
# /etc/systemd/system/icecast-corporate.service
[Unit]
Description=Icecast Stream Corporate MP3
After=network.target icecast2.service

[Service]
ExecStart=/usr/bin/ffmpeg -re -stream_loop -1 \
  -i /home/innovatech-admin/audio/corporate.mp3 \
  -acodec libmp3lame -ab 128k -f mp3 \
  icecast://source:@ITB2026@localhost:8000/corporate
Restart=always
RestartSec=5
User=innovatech-admin

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/icecast-formacio.service
[Unit]
Description=Icecast Stream Formacio OGG
After=network.target icecast2.service

[Service]
ExecStart=/usr/bin/ffmpeg -re -stream_loop -1 \
  -i /home/innovatech-admin/audio/formacio.ogg \
  -c:a libvorbis -b:a 96k \
  -content_type application/ogg \
  -f ogg \
  icecast://source:@ITB2026@localhost:8000/formacio
Restart=always
RestartSec=5
User=innovatech-admin

[Install]
WantedBy=multi-user.target
```

![Servei icecast-corporate actiu](captures/09-icecast-corporate-actiu.png)
![Servei icecast-formacio actiu](captures/10-icecast-formacio-actiu.png)

### 2.5 Verificació del servei

```bash
sudo systemctl status icecast2
ss -tlnp | grep 8000
```

![Servei Icecast2 actiu](captures/11-icecast2-actiu.png)
![Port 8000 en estat LISTEN](captures/12-port-8000.png)

**Verificació via interfície web:**

- Pàgina principal: `http://54.157.67.55:8000/`
- Panell d'administració: `http://54.157.67.55:8000/admin/`
- Llista de muntatges: `http://54.157.67.55:8000/admin/listmounts.xsl`

![Panell d'administració Icecast2](captures/13-icecast2-admin.png)
![Muntatges actius](captures/14-icecast2-mountpoints.png)

**Verificació des de clients:**

![Canal corporatiu MP3 al navegador](captures/15-navegador-corporate.png)
![Canal formació OGG al navegador Firefox](captures/16-navegador-formacio.png)
![VLC reproduint canal corporatiu](captures/17-vlc-corporate.png)
![VLC reproduint canal formació OGG](captures/18-vlc-formacio.png)

**Verificació Content-Type OGG:**

```bash
curl -v http://localhost:8000/formacio 2>&1 | grep "Content-Type"
```

![Content-Type application/ogg verificat](captures/19-content-type-ogg.png)

### 2.6 Resolució d'incidències

| Incidència | Causa | Solució |
|-----------|-------|---------|
| Pàgina web no carregava des de fora | Port 8000 no obert al Security Group d'AWS | Afegir regla d'entrada al Security Group per al port 8000 TCP |
| Serveis ffmpeg amb error `status=8` | Contrasenya incorrecta a la URL d'Icecast | Actualitzar la contrasenya `@ITB2026` als fitxers de servei |
| Canal `/formacio` retornava `Content-Type: audio/mpeg` en lloc d'`application/ogg` | ffmpeg envia el Content-Type independentment del format real | Afegir el paràmetre `-content_type application/ogg` a la comanda ffmpeg |
| Canal OGG no carregava al navegador Chrome | Chrome no suporta OGG nativament | Verificar amb Firefox que sí suporta OGG, i confirmar funcionament amb VLC |
