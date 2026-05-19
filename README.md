# 00 — Preparació del servidor srv4

## 1. Descripció de la màquina

La instància `innovatetech-media` (srv4) és el servidor dedicat als
serveis multimèdia del projecte InnovateTech. Allotja els serveis
d'àudio en streaming (Icecast2), vídeo en streaming (NGINX-RTMP)
i videoconferència (Jitsi Meet).

| Paràmetre | Valor |
|-----------|-------|
| Nom | innovatetech-media |
| Tipus EC2 | t2.medium |
| RAM | 4 GB |
| Disc | 15 GB gp3 |
| Sistema operatiu | Ubuntu Server 22.04 LTS |
| IP pública | X.X.X.X |
| Regió AWS | eu-west-1 (Irlanda) |

![Instància EC2 en estat running](captures/01-ec2-running.png)

## 2. Usuari d'administració

El projecte exigeix no utilitzar l'usuari per defecte del servidor
(`ubuntu`). S'ha creat l'usuari específic `innovatech-admin` amb
accés exclusivament per clau pública/privada, sense contrasenya.

**Justificació de seguretat:** L'ús d'un usuari específic i clau
SSH en lloc de contrasenya elimina el risc d'atacs de força bruta
i garanteix que només el personal autoritzat amb la clau privada
pot accedir al servidor.

```bash
# Creació de l'usuari i còpia de la clau SSH
sudo useradd -m -s /bin/bash innovatech-admin
sudo usermod -aG sudo innovatech-admin
sudo mkdir -p /home/innovatech-admin/.ssh
sudo cp /home/ubuntu/.ssh/authorized_keys \
  /home/innovatech-admin/.ssh/
sudo chown -R innovatech-admin:innovatech-admin \
  /home/innovatech-admin/.ssh
sudo chmod 700 /home/innovatech-admin/.ssh
sudo chmod 600 /home/innovatech-admin/.ssh/authorized_keys

# Desactivació de l'usuari per defecte
sudo passwd -l ubuntu
```

La connexió es realitza amb:

```bash
ssh -i innovatetech-key.pem innovatech-admin@X.X.X.X
```

![Connexió SSH amb l'usuari innovatech-admin](captures/03-ssh-innovatech-admin.png)

## 3. Seguretat de xarxa

S'han configurat dos nivells de control d'accés:

- **Security Group d'AWS:** primera capa, filtra el tràfic abans
  d'arribar al servidor.
- **UFW (Uncomplicated Firewall):** segona capa, filtra a nivell
  del sistema operatiu.

Ports habilitats:

| Port | Protocol | Servei | Accessible des de |
|------|----------|--------|-------------------|
| 22 | TCP | SSH administració | IP de l'admin |
| 8000 | TCP | Icecast2 àudio | Internet |
| 80 | TCP | HTTP / HLS vídeo | Internet |
| 1935 | TCP | RTMP vídeo | Internet |
| 443 | TCP | Jitsi Meet HTTPS | Internet |
| 4443 | TCP | Jitsi Videobridge | Internet |
| 10000 | UDP | Jitsi WebRTC | Internet |

![Regles del Security Group AWS](captures/02-security-group.png)
![UFW actiu amb ports oberts](captures/04-ufw-status.png)
