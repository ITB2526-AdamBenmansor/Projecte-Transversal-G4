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

### 2.5 Verificació del servei

**Verificació a nivell de sistema:**

```bash
sudo systemctl status icecast2
ss -tlnp | grep 8000
```

La comanda `ss -tlnp` confirma que el port 8000 es troba en
estat `LISTEN` i que el procés responsable és `icecast2`,
consistent amb la configuració definida al fitxer `icecast.xml`.

![Servei Icecast2 actiu](captures/07-icecast2-actiu.png)
![Port 8000 en estat LISTEN](captures/08-port-8000.png)

**Verificació via interfície web:**

Icecast2 inclou una interfície web integrada accessible des
de qualsevol navegador sense necessitat de programari addicional:

- **Pàgina principal:** `http://54.80.18.165:8000/`
- **Panell d'administració:** `http://54.80.18.165:8000/admin/`
- **Llista de muntatges:** `http://54.80.18.165:8000/admin/listmounts.xsl`

El panell d'administració mostra les estadístiques globals
del servidor: nombre de clients connectats, connexions totals,
hostname i versió d'Icecast2 instal·lada.

La secció de muntatges actius confirma que els dos canals
estan operatius amb les seves fonts ffmpeg connectades
i emetent en temps real.

![Panell d'administració Icecast2](captures/10-icecast2-admin.png)
![Muntatges actius al panell](captures/14-icecast2-mountpoints.png)

**Verificació des de clients:**

S'ha verificat l'accés al servei des de dos tipus de clients:

1. **Navegador web:** els dos canals són accessibles
   directament introduint la URL al navegador. El navegador
   reprodueix l'àudio mitjançant HTML5 sense necessitat de
   cap programari addicional.

2. **VLC Media Player:** client d'àudio dedicat compatible
   amb tots els sistemes operatius, que accedeix al canal
   via la funció d'obertura de flux de xarxa.

![Canal corporatiu reproduint al navegador](captures/15-navegador-corporate.png)
![Canal formació reproduint al navegador](captures/16-navegador-formacio.png)
![VLC reproduint el canal corporatiu](captures/17-vlc-reproduint.png)

### 2.6 Incidències i solucions

Durant la implementació del servei d'àudio es van detectar
les incidències següents:

| Incidència | Causa | Solució |
|-----------|-------|---------|
| Pàgina web no carregava des de fora | Port 8000 no obert al Security Group d'AWS | Afegir regla d'entrada al Security Group per al port 8000 TCP |
| Serveis ffmpeg amb error status=8 | Contrasenya incorrecta a la URL d'Icecast | Actualitzar la contrasenya correcta `@ITB2026` als fitxers de servei |
| Canal `/formacio` no carregava al navegador | Format OGG no compatible amb navegadors | Canviar el còdec de sortida de `libvorbis` a `libmp3lame` mantenint el fitxer font OGG |

![Servei icecast-corporate actiu](captures/07-icecast2-actiu.png)
![Servei icecast-formacio actiu](captures/12-icecast-formacio-actiu.png)
![Servei icecast-formacio actualitzat](captures/13-icecast-formacio-service.png)
