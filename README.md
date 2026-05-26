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
