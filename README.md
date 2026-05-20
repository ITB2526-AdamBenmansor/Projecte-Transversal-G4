# Bloc 2 — Serveis de xarxa i Internet (0375)

## 1. Preparació del servidor

### 1.1 Descripció de la màquina

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
| IP pública | 54.80.18.165 |
| Regió AWS | eu-west-1 (Irlanda) |

![Instància EC2 en estat running](captures/01-ec2-running.png)

### 1.2 Usuari d'administració

El projecte exigeix no utilitzar l'usuari per defecte del
servidor (`ubuntu`). S'ha creat l'usuari específic
`innovatech-admin` amb accés exclusivament per clau
pública/privada, sense contrasenya.

**Justificació de seguretat:** L'ús d'un usuari específic i
clau SSH elimina el risc d'atacs de força bruta i garanteix
que només el personal autoritzat amb la clau privada pot
accedir al servidor.

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

![Connexió SSH amb innovatech-admin](captures/03-ssh-innovatech-admin.png)

### 1.3 Seguretat de xarxa

S'han configurat dos nivells de control d'accés:
- **Security Group d'AWS:** primera capa, filtra el tràfic
  abans d'arribar al servidor.
- **UFW:** segona capa, filtra a nivell del sistema operatiu.

| Port | Protocol | Servei | Accessible des de |
|------|----------|--------|-------------------|
| 22 | TCP | SSH | IP de l'admin |
| 8000 | TCP | Icecast2 | Internet |
| 80 | TCP | HTTP / HLS | Internet |
| 1935 | TCP | RTMP | Internet |
| 443 | TCP | Jitsi HTTPS | Internet |
| 4443 | TCP | Jitsi Videobridge | Internet |
| 10000 | UDP | Jitsi WebRTC | Internet |

![Security Group AWS](captures/02-security-group.png)
![UFW ports oberts](captures/04-ufw-status.png)
## 2. Servei d'àudio Icecast2

### 2.1 Descripció del servei

Icecast2 és un servidor de streaming d'àudio de codi obert que
permet distribuir contingut d'àudio en temps real a múltiples
clients simultàniament. Suporta els formats MP3 i OGG, i ofereix
una interfície web integrada per a la monitorització i
administració del servei.

S'ha triat Icecast2 perquè és l'estàndard del sector per a
streaming d'àudio en entorns empresarials, és lleuger en consum
de recursos i permet configurar múltiples canals independents
amb bitrates diferenciats segons l'ús.

A InnovateTech s'utilitza per cobrir dues necessitats:
- Distribució de contingut corporatiu intern.
- Emissions de sessions de formació interna.

### 2.2 Instal·lació

```bash
sudo apt install icecast2 -y
```

Durant el procés d'instal·lació, l'assistent de configuració
sol·licita les contrasenyes d'accés per als rols de source,
relay i administrador.

![Icecast2 instal·lat](captures/05-icecast2-instalat.png)

### 2.3 Configuració

El fitxer de configuració principal és `/etc/icecast2/icecast.xml`.

**Canals configurats:**

| Canal | Muntatge | Format | Bitrate | Ús |
|-------|----------|--------|---------|-----|
| Corporatiu | `/corporate` | MP3 | 128 kbps | Comunicació interna |
| Formació | `/formacio` | MP3 | 96 kbps | Sessions de formació |

S'han configurat dos canals amb bitrates diferents:
- **128 kbps** per al canal corporatiu, per a música i
  contingut general.
- **96 kbps** per al canal de formació, suficient per a
  contingut de veu i que redueix el consum d'amplada de banda.

Durant la implementació es va plantejar inicialment utilitzar
OGG Vorbis per al canal de formació. No obstant això, es va
detectar que OGG no és compatible de forma nativa amb tots
els navegadors web actuals. Per garantir l'accessibilitat
universal, es va decidir utilitzar MP3 per als dos canals.

![Fitxer de configuració icecast.xml](captures/06-icecast2-config.png)
### 2.4 Font d'àudio amb ffmpeg

Com que la instància EC2 és un servidor sense targeta de so
física, s'ha utilitzat **ffmpeg** com a font d'àudio virtual.
Ffmpeg llegeix fitxers d'àudio pregravats i els publica en
bucle continu a Icecast2, simulant una emissió en directe.

S'han preparat dos fitxers al directori
`/home/innovatech-admin/audio/`:

- `corporate.mp3`: fitxer MP3 per al canal corporatiu,
  emès a 128 kbps.
- `formacio.ogg`: fitxer OGG utilitzat com a font per al
  canal de formació. Ffmpeg el llegeix i el converteix a
  MP3 en temps real durant l'emissió.

![Fitxers d'àudio al servidor](captures/11-audio-fitxers.png)

S'han creat dos serveis systemd per garantir que els canals
s'iniciïn automàticament amb el servidor i es reiniciïn
automàticament en cas de fallada:

```bash
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

```bash
# /etc/systemd/system/icecast-formacio.service
[Unit]
Description=Icecast Stream Formacio OGG
After=network.target icecast2.service

[Service]
ExecStart=/usr/bin/ffmpeg -re -stream_loop -1 \
  -i /home/innovatech-admin/audio/formacio.ogg \
  -acodec libmp3lame -ab 96k -f mp3 \
  icecast://source:@ITB2026@localhost:8000/formacio
Restart=always
RestartSec=5
User=innovatech-admin

[Install]
WantedBy=multi-user.target
```

Els paràmetres més rellevants de la configuració són:
- `After=icecast2.service`: garanteix que ffmpeg no intenta
  publicar fins que Icecast2 estigui completament iniciat.
- `Restart=always`: si ffmpeg falla, systemd el reinicia
  automàticament.
- `RestartSec=5`: espera 5 segons abans de reiniciar,
  evitant bucles de reinici massa ràpids.
- `User=innovatech-admin`: el procés s'executa amb l'usuari
  específic del projecte, no amb root.

![Servei icecast-corporate actiu](captures/07-icecast2-actiu.png)
![Servei icecast-formacio actiu](captures/12-icecast-formacio-actiu.png)
![Servei icecast-formacio actualitzat](captures/13-icecast-formacio-service.png)
