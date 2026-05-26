# Bloc 2 — Serveis de Xarxa i Internet (0375)

## Índex
- [1. Preparació del servidor](#1-preparació-del-servidor)
- [2. Servei d'àudio — Icecast2](#2-servei-daudio--icecast2)
  - [2.1 Descripció del servei](#21-descripció-del-servei)
  - [2.2 Instal·lació](#22-installació)
  - [2.3 Configuració](#23-configuració)
  - [2.4 Font d'àudio amb ffmpeg](#24-font-dàudio-amb-ffmpeg)
  - [2.5 Verificació del servei](#25-verificació-del-servei)
  - [2.6 Resolució d'incidències](#26-resolució-dincidències)
- [3. Servei de vídeo — NGINX-RTMP](#3-servei-de-vídeo--nginx-rtmp)
  - [3.1 Descripció del servei](#31-descripció-del-servei)
  - [3.2 Instal·lació de dependències](#32-installació-de-dependències)
  - [3.3 Descàrrega i compilació](#33-descàrrega-i-compilació)
  - [3.4 Configuració](#34-configuració)
  - [3.5 Servei systemd](#35-servei-systemd)
  - [3.6 Font de vídeo amb ffmpeg](#36-font-de-vídeo-amb-ffmpeg)
  - [3.7 Reproductor web](#37-reproductor-web)
  - [3.8 Verificació del servei](#38-verificació-del-servei)
  - [3.9 Resolució d'incidències](#39-resolució-dincidències)
- [4. Videoconferència — Jitsi Meet](#4-videoconferència--jitsi-meet)
  - [4.1 Descripció del servei](#41-descripció-del-servei)
  - [4.2 Preparació de l'entorn](#42-preparació-de-lentorn)
  - [4.3 Instal·lació](#43-installació)
  - [4.4 Configuració SSL](#44-configuració-ssl)
  - [4.5 Configuració NAT AWS](#45-configuració-nat-aws)
  - [4.6 Verificació del servei](#46-verificació-del-servei)
  - [4.7 Resolució d'incidències](#47-resolució-dincidències)
- [5. Proves d'amplada de banda](#5-proves-damplada-de-banda)
  - [5.1 Objectiu](#51-objectiu)
  - [5.2 Eines utilitzades](#52-eines-utilitzades)
  - [5.3 Prova 1 — Servidor en repòs](#53-prova-1--servidor-en-repòs)
  - [5.4 Prova 2 — Servidor amb càrrega](#54-prova-2--servidor-amb-càrrega)
  - [5.5 Anàlisi dels resultats](#55-anàlisi-dels-resultats)
  - [5.6 Conclusió tècnica](#56-conclusió-tècnica)

---

## 1. Preparació del servidor

La instància `innovatetech-media` (srv4) és el servidor dedicat
als serveis multimèdia del projecte InnovateTech. Allotja els
serveis d'àudio en streaming (Icecast2), vídeo en streaming
(NGINX-RTMP) i videoconferència (Jitsi Meet).

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

### 1.1 Usuari d'administració

El projecte exigeix no utilitzar l'usuari per defecte (`ubuntu`).
S'ha creat l'usuari específic `innovatech-admin` amb accés
exclusivament per clau pública/privada, sense contrasenya.

**Justificació de seguretat:** L'ús d'un usuari específic i clau
SSH elimina el risc d'atacs de força bruta i garanteix que només
el personal autoritzat pot accedir al servidor.

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
<img width="1002" height="506" alt="Captura de pantalla de 2026-05-26 09-26-58" src="https://github.com/user-attachments/assets/ce85e9da-6ba2-4e70-91fa-2df828d3e761" />

![Connexió SSH amb innovatech-admin] (https://github.com/ITB2526-AdamBenmansor/Projecte-Transversal/blob/main/assets/capturas/Captura%20de%20pantalla%20de%202026-05-26%2009-29-11.png)

### 1.2 Seguretat de xarxa

S'han configurat dos nivells de control d'accés: Security Group
d'AWS (primera capa) i UFW (segona capa a nivell de SO).

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

[↑ Tornar a l'índex](#índex)

---

## 2. Servei d'àudio — Icecast2

### 2.1 Descripció del servei

Icecast2 és un servidor de streaming d'àudio de codi obert que
permet distribuir contingut d'àudio en temps real a múltiples
clients simultàniament. Suporta els formats MP3 i OGG, i ofereix
una interfície web integrada per a la monitorització i
administració del servei.

S'ha triat Icecast2 perquè és l'estàndard del sector per a
streaming d'àudio en entorns empresarials, és lleuger en consum
de recursos i permet configurar múltiples canals independents
amb formats i bitrates diferenciats.

A InnovateTech s'utilitza per cobrir dues necessitats:
- Distribució de contingut corporatiu intern en format MP3.
- Emissions de sessions de formació interna en format OGG.

### 2.2 Instal·lació

```bash
sudo apt install icecast2 -y
```

Durant la instal·lació l'assistent demana les contrasenyes
d'accés per als rols de source, relay i administrador:

| Paràmetre | Valor |
|-----------|-------|
| Hostname | audio.innovatetech.local |
| Source password | @ITB2026 |
| Relay password | @ITB2026 |
| Admin password | @ITB2026 |

![Assistent configuració Icecast2](captures/05-icecast2-assistent.png)
![Icecast2 instal·lat correctament](captures/06-icecast2-instalat.png)

### 2.3 Configuració

El fitxer de configuració principal és `/etc/icecast2/icecast.xml`.
S'han configurat dos canals amb formats diferenciats:

| Canal | Muntatge | Format | Bitrate | Ús |
|-------|----------|--------|---------|-----|
| Corporatiu | `/corporate` | MP3 | 128 kbps | Comunicació interna |
| Formació | `/formacio` | OGG Vorbis | 96 kbps | Sessions de formació |

- **MP3 al canal corporatiu:** format universal compatible amb
  tots els navegadors i clients sense programari addicional.
- **OGG al canal de formació:** format lliure que ofereix millor
  qualitat de veu al mateix bitrate, ideal per a sessions
  formatives.

![Fitxer de configuració icecast.xml](captures/07-icecast2-config.png)

### 2.4 Font d'àudio amb ffmpeg

Com que la instància EC2 no té targeta de so física, s'ha
utilitzat ffmpeg com a font d'àudio virtual. Ffmpeg llegeix
fitxers d'àudio pregravats i els publica en bucle continu
a Icecast2, simulant una emissió en directe.

Fitxers d'àudio preparats al directori
`/home/innovatech-admin/audio/`:

- `corporate.mp3`: canal corporatiu, emès a 128 kbps en MP3.
- `formacio.ogg`: canal de formació, emès a 96 kbps en OGG Vorbis.

![Fitxers d'àudio al servidor](captures/08-audio-fitxers.png)

S'han creat dos serveis systemd per garantir l'inici automàtic
i la recuperació en cas de fallada:

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

Els paràmetres més rellevants dels serveis són:
- `After=icecast2.service`: ffmpeg no publica fins que Icecast2
  estigui completament iniciat.
- `Restart=always`: systemd reinicia el procés automàticament
  en cas de fallada.
- `RestartSec=5`: espera 5 segons abans de reiniciar.
- `User=innovatech-admin`: s'executa amb l'usuari específic
  del projecte, no amb root.
- `-content_type application/ogg`: força el Content-Type correcte
  per al canal OGG (solució a la incidència detectada).

![Servei icecast-corporate actiu](captures/09-icecast-corporate-actiu.png)
![Servei icecast-formacio actiu](captures/10-icecast-formacio-actiu.png)

### 2.5 Verificació del servei

**Verificació a nivell de sistema:**

```bash
sudo systemctl status icecast2
ss -tlnp | grep 8000
```

![Servei Icecast2 active running](captures/11-icecast2-actiu.png)
![Port 8000 en estat LISTEN](captures/12-port-8000.png)

**Verificació via interfície web:**

| URL | Descripció |
|-----|------------|
| `http://54.157.67.55:8000/` | Pàgina principal |
| `http://54.157.67.55:8000/admin/` | Panell d'administració |
| `http://54.157.67.55:8000/admin/listmounts.xsl` | Muntatges actius |

![Panell d'administració Icecast2](captures/13-icecast2-admin.png)
![Muntatges actius amb els dos canals](captures/14-icecast2-mountpoints.png)

**Verificació Content-Type OGG:**

```bash
curl -v http://localhost:8000/formacio 2>&1 | grep "Content-Type"
```

![Content-Type application/ogg verificat](captures/15-content-type-ogg.png)

**Verificació des de clients:**

![Canal corporatiu MP3 reproduint al navegador](captures/16-navegador-corporate.png)
![Canal formació OGG reproduint a Firefox](captures/17-navegador-formacio.png)
![VLC reproduint canal corporatiu](captures/18-vlc-corporate.png)
![VLC reproduint canal formació OGG](captures/19-vlc-formacio.png)

### 2.6 Resolució d'incidències

| # | Incidència | Causa | Solució |
|---|-----------|-------|---------|
| 1 | Pàgina web no carregava des de fora | Port 8000 no obert al Security Group d'AWS | Afegir regla d'entrada al Security Group per al port 8000 TCP |
| 2 | Serveis ffmpeg amb error `status=8` | Contrasenya incorrecta a la URL d'Icecast | Actualitzar la contrasenya `@ITB2026` als fitxers de servei |
| 3 | Canal `/formacio` retornava `Content-Type: audio/mpeg` | ffmpeg envia el Content-Type independentment del format | Afegir `-content_type application/ogg` a la comanda ffmpeg |
| 4 | Canal OGG no carregava al navegador Chrome | Chrome no suporta OGG nativament | Verificar amb Firefox i VLC que sí suporten OGG |

[↑ Tornar a l'índex](#índex)

---

## 3. Servei de vídeo — NGINX-RTMP

### 3.1 Descripció del servei

El servei de vídeo es basa en NGINX compilat manualment amb
el mòdul RTMP, que permet rebre fluxos de vídeo en protocol
RTMP i convertir-los automàticament a format HLS per a la
distribució via HTTP als navegadors.

S'ha triat NGINX+RTMP perquè és la solució estàndard del
sector per a streaming de vídeo empresarial, és compatible
amb tots els navegadors via HLS i és de codi obert.

**Protocols i formats utilitzats:**

| Protocol/Format | Ús | Port |
|----------------|-----|------|
| RTMP | Publicació del flux | 1935 |
| HLS (.m3u8 + .ts) | Distribució als navegadors | 8080 |
| H.264 (libx264) | Còdec de vídeo | — |
| AAC | Còdec d'àudio | — |
| MP4 | Vídeo font i VOD | 8080 |

### 3.2 Instal·lació de dependències

S'ha compilat NGINX manualment perquè la versió dels
repositoris d'Ubuntu no inclou el mòdul RTMP.

```bash
sudo apt install -y build-essential libpcre3 libpcre3-dev \
  libssl-dev zlib1g-dev git
```

- `build-essential`: compilador GCC i eines de compilació.
- `libpcre3/dev`: expressions regulars per al mòdul d'URL.
- `libssl-dev`: biblioteques OpenSSL per a HTTPS.
- `zlib1g-dev`: biblioteca de compressió per a gzip.
- `git`: per descarregar el mòdul RTMP de GitHub.

![Dependències instal·lades](captures/20-nginx-deps.png)

### 3.3 Descàrrega i compilació

```bash
cd /tmp
wget http://nginx.org/download/nginx-1.24.0.tar.gz
tar -xzvf nginx-1.24.0.tar.gz
git clone https://github.com/arut/nginx-rtmp-module.git

cd nginx-1.24.0
./configure \
  --with-http_ssl_module \
  --with-http_v2_module \
  --with-http_mp4_module \
  --add-module=/tmp/nginx-rtmp-module
make
sudo make install
```

Paràmetres de compilació:
- `--with-http_ssl_module`: suport HTTPS.
- `--with-http_v2_module`: suport HTTP/2.
- `--with-http_mp4_module`: suport seek en fitxers MP4.
- `--add-module=/tmp/nginx-rtmp-module`: mòdul RTMP.

![NGINX i mòdul RTMP descarregats](captures/21-nginx-descarregat.png)
![NGINX compilat correctament](captures/22-nginx-compilat.png)

### 3.4 Configuració

Fitxer: `/usr/local/nginx/conf/nginx.conf`

```nginx
worker_processes auto;
events {
    worker_connections 1024;
}

rtmp {
    server {
        listen 1935;
        chunk_size 4096;

        application live {
            live on;
            record off;
            hls on;
            hls_path /tmp/hls;
            hls_fragment 3s;
            hls_playlist_length 60s;
        }

        application vod {
            play /var/videos;
        }
    }
}

http {
    sendfile off;
    tcp_nopush on;
    directio 512;
    default_type application/octet-stream;

    server {
        listen 0.0.0.0:8080;
        server_name video.innovatetech.local;

        location /hls {
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            root /tmp;
            add_header Cache-Control no-cache;
            add_header Access-Control-Allow-Origin *;
        }

        location /vod {
            root /var;
            add_header Cache-Control no-cache;
            add_header Access-Control-Allow-Origin *;
        }

        location /stat {
            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }

        location / {
            root /usr/local/nginx/html;
            index index.html;
        }
    }
}
```

![Fitxer de configuració nginx.conf](captures/23-nginx-config.png)

### 3.5 Servei systemd

```ini
# /etc/systemd/system/nginx-rtmp.service
[Unit]
Description=NGINX RTMP Streaming Server
After=network.target

[Service]
ExecStartPre=/usr/local/nginx/sbin/nginx -t
ExecStart=/usr/local/nginx/sbin/nginx -g 'daemon off;'
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable nginx-rtmp
sudo systemctl start nginx-rtmp
```

![Servei nginx-rtmp actiu](captures/24-nginx-rtmp-actiu.png)

### 3.6 Font de vídeo amb ffmpeg

```bash
sudo mkdir -p /var/videos
sudo wget -O /var/videos/prova.mp4 \
  "https://www.w3schools.com/html/mov_bbb.mp4"
sudo chmod 777 /tmp/hls
```

Servei systemd per publicar el vídeo automàticament:

```ini
# /etc/systemd/system/nginx-stream.service
[Unit]
Description=NGINX RTMP Video Stream
After=network.target nginx-rtmp.service

[Service]
ExecStartPre=/bin/sleep 5
ExecStart=/usr/bin/ffmpeg -re -stream_loop -1 \
  -i /var/videos/prova.mp4 \
  -vcodec libx264 \
  -acodec aac \
  -f flv \
  rtmp://localhost/live/stream1
Restart=always
RestartSec=5
User=innovatech-admin

[Install]
WantedBy=multi-user.target
```

![Vídeo descarregat al servidor](captures/25-video-descarregat.png)
![Segments HLS generats a /tmp/hls](captures/26-hls-segments.png)

### 3.7 Reproductor web

S'ha creat una pàgina HTML amb reproductor integrat
utilitzant la biblioteca HLS.js, que permet als navegadors
reproduir streams `.m3u8` via JavaScript:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>InnovateTech - Streaming Video</title>
  <script src="https://cdn.jsdelivr.net/npm/hls.js@latest">
  </script>
  <style>
    body {
      background-color: #1a1a1a;
      color: white;
      font-family: Arial, sans-serif;
      text-align: center;
      padding: 40px;
    }
    h1 { color: #00aaff; }
    video {
      width: 720px;
      max-width: 100%;
      border: 2px solid #00aaff;
      border-radius: 8px;
    }
  </style>
</head>
<body>
  <h1>InnovateTech - Canal de Vídeo</h1>
  <p>Streaming en directe via HLS</p>
  <video id="video" controls autoplay muted></video>
  <script>
    var video = document.getElementById('video');
    var videoSrc = '/hls/stream1.m3u8';
    if (Hls.isSupported()) {
      var hls = new Hls();
      hls.loadSource(videoSrc);
      hls.attachMedia(video);
    } else if (video.canPlayType(
        'application/vnd.apple.mpegurl')) {
      video.src = videoSrc;
    }
  </script>
</body>
</html>
```

### 3.8 Verificació del servei

```bash
sudo systemctl status nginx-rtmp
sudo systemctl status nginx-stream
ss -tlnp | grep -E "1935|8080"
```

![Serveis nginx-rtmp i nginx-stream actius](captures/27-nginx-serveis-actius.png)
![Ports 1935 i 8080 escoltant](captures/28-nginx-ports.png)
![Reproductor web amb vídeo actiu](captures/29-nginx-web-reproductor.png)
![VLC reproduint stream HLS](captures/30-vlc-video.png)

### 3.9 Resolució d'incidències

| # | Incidència | Causa | Solució |
|---|-----------|-------|---------|
| 1 | Error `unknown directive mp4` | NGINX compilat sense `--with-http_mp4_module` | Recompilar NGINX afegint el mòdul MP4 |
| 2 | Error `location directive not allowed here` | Bloc `location /vod` dins del bloc `rtmp` | Moure el bloc al bloc `http` |
| 3 | Conflicte de ports amb Jitsi | NGINX de Jitsi ocupava el port 80 | Canviar NGINX-RTMP al port 8080 |
| 4 | Port 8080 escoltava a `127.0.0.1` | Interferència del NGINX de Jitsi | Especificar `listen 0.0.0.0:8080` i obrir el port 8080 al Security Group |
| 5 | Directori `/tmp/hls` sense permisos | Creat sense permisos d'escriptura per a NGINX | Executar `sudo chmod 777 /tmp/hls` |
| 6 | Vídeo no carregava al reproductor | ffmpeg no publicava el stream | Crear servei systemd `nginx-stream` |

[↑ Tornar a l'índex](#índex)

---

## 4. Videoconferència — Jitsi Meet

### 4.1 Descripció del servei

Jitsi Meet és una plataforma de videoconferència de codi obert
basada en el protocol WebRTC, que permet realitzar videotrucades
directament des del navegador sense necessitat d'instal·lar cap
aplicació addicional.

**WebRTC** és un estàndard obert que permet la comunicació en
temps real de veu, vídeo i dades directament entre navegadors.
Utilitza els protocols ICE i STUN per a la negociació de la
connexió i SRTP per al xifrat del flux.

La instal·lació de Jitsi Meet desplega tres serveis:

- `jitsi-videobridge2`: gestiona els fluxos de vídeo i àudio
  entre participants via WebRTC.
- `jicofo`: coordina la creació i gestió de les sales.
- `prosody`: servidor XMPP per a la senyalització.

### 4.2 Preparació de l'entorn

Per evitar incompatibilitats amb WebRTC (que requereix HTTPS
sobre un domini real), s'ha utilitzat el servei de DNS dinàmic
invers **sslip.io**. Qualsevol petició cap a
`54.157.67.55.sslip.io` resol automàticament cap a la IP
pública d'AWS, permetent la validació del certificat SSL.

```bash
sudo hostnamectl set-hostname 54.157.67.55.sslip.io
```

Modificació de `/etc/hosts`:
### 4.3 Instal·lació

```bash
# Afegir repositori Jitsi
curl -o /tmp/jitsi-key.gpg.key \
  https://download.jitsi.org/jitsi-key.gpg.key
sudo gpg --dearmor \
  -o /usr/share/keyrings/jitsi-keyring.gpg \
  /tmp/jitsi-key.gpg.key

echo 'deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] \
  https://download.jitsi.org stable/' | \
  sudo tee /etc/apt/sources.list.d/jitsi-stable.list

sudo apt update
sudo apt install -y jitsi-meet
```

Durant la instal·lació:

| Paràmetre | Valor |
|-----------|-------|
| Hostname | 54.157.67.55.sslip.io |
| Certificat SSL | Generate a new self-signed certificate |

### 4.4 Configuració SSL

S'ha obtingut un certificat SSL real de Let's Encrypt mitjançant
l'script oficial de Jitsi:

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo systemctl restart nginx
sudo /usr/share/jitsi-meet/scripts/install-letsencrypt-cert.sh
```

### 4.5 Configuració NAT AWS

Com que les instàncies EC2 utilitzen una IP privada interna
darrere d'un NAT d'AWS, s'ha configurat el Jitsi Videobridge
perquè els clients externs sàpiguen on enviar els fluxos:

```bash
sudo nano /etc/jitsi/videobridge/sip-communicator.properties
```

```properties
org.ice4j.ice.harvest.NAT_HARVESTER_LOCAL_ADDRESS=172.31.34.111
org.ice4j.ice.harvest.NAT_HARVESTER_PUBLIC_ADDRESS=54.157.67.55
```

```bash
# Fitxer /etc/jitsi/videobridge/config
JVB_OPTS="--apis=rest,xmpp"
```

```bash
sudo systemctl restart prosody jicofo jitsi-videobridge2 nginx
```

### 4.6 Verificació del servei

```bash
sudo systemctl status jitsi-videobridge2
sudo systemctl status jicofo
sudo systemctl status prosody
```

![Tres serveis Jitsi actius](captures/31-jitsi-serveis-actius.png)
![Pàgina principal Jitsi Meet](captures/32-jitsi-web.png)
![Videotrucada funcional amb 2 participants](captures/33-jitsi-videotrucada.png)

### 4.7 Resolució d'incidències

| # | Incidència | Causa | Solució |
|---|-----------|-------|---------|
| 1 | Let's Encrypt fallava amb error DNS | Domini `innovatetech.itb.cat` sense registre DNS públic | Utilitzar `sslip.io` com a DNS dinàmic invers |
| 2 | Error 404 a la validació ACME | El virtual host per defecte d'Ubuntu interceptava les peticions | Eliminar `/etc/nginx/sites-enabled/default` i reiniciar NGINX |
| 3 | Videobridge s'aturava constantment | Variable `JVB_OPTS` buida al fitxer de configuració | Afegir `JVB_OPTS="--apis=rest,xmpp"` al fitxer `/etc/jitsi/videobridge/config` |
| 4 | Clients no podien connectar a la sala | NAT d'AWS no configurat al videobridge | Afegir les adreces local i pública al fitxer `sip-communicator.properties` |
| 5 | Port 10000 UDP no accessible | Security Group d'AWS sense la regla UDP 10000 | Afegir regla UDP 10000 al Security Group |

[↑ Tornar a l'índex](#índex)

---

## 5. Proves d'amplada de banda

### 5.1 Objectiu

Verificar que la infraestructura desplegada és capaç de suportar
els serveis d'àudio, vídeo i videoconferència sense degradació,
mesurant velocitat de baixada, pujada i latència en dos escenaris.

### 5.2 Eines utilitzades

```bash
sudo apt install speedtest-cli nethogs -y
```

- **speedtest-cli**: mesura la velocitat de connexió amb
  servidors de referència a internet.
- **nethogs**: monitoritza el consum de bandwidth per procés
  en temps real.

**Requeriments mínims per servei:**

| Servei | Bandwidth mínim per client |
|--------|---------------------------|
| Streaming àudio MP3 128 kbps | 0.128 Mbit/s |
| Streaming àudio OGG 96 kbps | 0.096 Mbit/s |
| Streaming vídeo HLS 720p | 2.5 Mbit/s |
| Videoconferència Jitsi 720p | 3 Mbit/s simètric |

### 5.3 Prova 1 — Servidor en repòs

```bash
speedtest-cli --simple
```

| Mesura | Resultat |
|--------|----------|
| Ping | 5.255 ms |
| Download | 956.00 Mbit/s |
| Upload | 966.14 Mbit/s |

![Speedtest servidor en repòs](captures/34-speedtest-repos.png)

### 5.4 Prova 2 — Servidor amb càrrega

Amb clients connectats als dos canals d'Icecast2 simultàniament:

**Consum per procés (nethogs):**

| Procés | Enviat | Rebut |
|--------|--------|-------|
| icecast2 | 23.479 KB/s | 1.057 KB/s |
| sshd | 0.130 KB/s | 0.052 KB/s |
| **TOTAL** | **23.609 KB/s** | **1.109 KB/s** |

![Nethogs consum per procés](captures/35-nethogs-carrega.png)

**Resultats comparatius:**

| Mesura | Repòs | Amb càrrega | Diferència |
|--------|-------|-------------|------------|
| Ping | 5.255 ms | 4.76 ms | -0.495 ms |
| Download | 956.00 Mbit/s | 954.98 Mbit/s | -1.02 Mbit/s |
| Upload | 966.14 Mbit/s | 968.25 Mbit/s | +2.11 Mbit/s |

![Speedtest servidor amb càrrega](captures/36-speedtest-carrega.png)

### 5.5 Anàlisi dels resultats

La comparació entre les dues proves demostra que els serveis
multimèdia tenen un impacte pràcticament nul sobre el rendiment
de la xarxa. Les diferències entre les mesures en repòs i amb
càrrega són inferiors a l'1%.

| Servei | Consum per client | Capacitat upload | Clients suportats |
|--------|------------------|-----------------|-------------------|
| MP3 128 kbps | 0.128 Mbit/s | 968 Mbit/s | ~7.500 |
| OGG 96 kbps | 0.096 Mbit/s | 968 Mbit/s | ~10.000 |
| Vídeo HLS 720p | 2.5 Mbit/s | 968 Mbit/s | ~387 |
| Jitsi 720p | 3 Mbit/s | 968 Mbit/s | ~322 |

### 5.6 Conclusió tècnica

La infraestructura desplegada a la instància EC2 `t2.medium`
a la regió `eu-west-1` d'AWS disposa d'una amplada de banda
molt superior als requeriments dels serveis multimèdia
d'InnovateTech.

| Criteri | Valor mínim | Valor obtingut | Estat |
|---------|------------|----------------|-------|
| Ping | < 50 ms | 4.76 ms | ✅ |
| Download | > 10 Mbit/s | 954.98 Mbit/s | ✅ |
| Upload | > 5 Mbit/s | 968.25 Mbit/s | ✅ |
| Impacte dels serveis | < 10% | < 1% | ✅ |

La infraestructura es classifica com a **ACCEPTABLE** per als
casos d'ús definits. Si en un futur InnovateTech necessités
escalar el servei a milers de clients simultanis, es recomana
avaluar l'ús d'Amazon CloudFront com a CDN per a la distribució
del contingut.

[↑ Tornar a l'índex](#índex)
