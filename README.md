
# Projecte Transversal — InnovateTech
## Sistema Global d'Infraestructura, Serveis de Xarxa i Gestió de Bases de Dades

> **Projecte Transversal ASIXc1 · Curs 2025/2026**
> **Grup 4 (Innovate Tech)**
> **Institut Tecnològic de Barcelona (ITB)**
> **Integrants:** Adam Benmansor, Leonel Coello, Oriol Coll, Victor Barreda.

---

##  Índex General del Projecte

1. [Introducció i Objectiu Global](#1-introducció-i-objectiu-global)
2. [Estructura del Repositori](#2-estructura-del-repositori)
3. [Resum Executiu dels Blocs Tècnics](#3-resum-executiu-dels-blocs-tècnics)
   - [Bloc 1.1: Proposta CPD — Infraestructura Física](#bloc-11-proposta-cpd--infraestructura-física)
   - [Bloc 1.2: Infraestructura de Xarxes i Sistemes (AWS i Ansible)](#bloc-12-infraestructura-de-xarxes-i-sistemes-aws-i-ansible)
   - [Bloc 2: Serveis de Xarxa i Internet (Multimèdia)](#bloc-2-serveis-de-xarxa-i-internet-multimèdia)
   - [Bloc 0377: Administració de Bases de Dades (MariaDB)](#bloc-0377-administració-de-bases-de-dades-mariadb)
4. [Stack Tecnològic Global](#4-stack-tecnològic-global)
5. [Arquitectura i Seguretat Integrada](#5-arquitectura-i-seguretat-integrada)
6. [Conclusions del Projecte](#6-conclusions-del-projecte)

---

## 1. Introducció i Objectiu Global

El projecte de **InnovateTech** consisteix en el disseny, desplegament, automatització i assegurament d'una infraestructura corporativa completa, simulant un entorn empresarial real d'alta disponibilitat i seguretat. 

L'objectiu principal ha estat integrar de manera cohesionada el disseny físic d'un Centre de Processament de Dades (CPD) amb la potència del *Cloud Computing* d'**Amazon Web Services (AWS)**. Mitjançant pràctiques d'**Infraestructura com a Codi (IaC)**, s'ha automatitzat completament la configuració dels servidors, implementant directoris centrals d'identitats (LDAP), polítiques d'auditoria centralitzada, serveis d'emissió multimèdia en temps real i una base de dades corporativa robusta sota el principi de mínim privilegi.

---

## 2. Estructura del Repositori

La documentació tècnica es troba dividida en quatre mòduls o blocs principals, cadascun enfocat en una capa específica de la infraestructura:

| Fitxer / Document | Descripció | Contingut Clau |
| :--- | :--- | :--- |
|  `Proposta CPD  - Infraestructura Física.md` | Disseny físic de la sala de servidors corporativa. | Plànols, climatització, inventari de maquinària, xarxa física, seguretat i estratègia de backup 3-2-1. |
|  `AWS.md` | Implementació del nucli de xarxa i automatització cloud. | Instàncies EC2, playbooks d'Ansible, servidor OpenLDAP central, clients SSSD, DNS intern i securització SSL/TLS. |
|  `Multimèdia.md` | Desplegament de serveis multimèdia i internet. | Streaming d'àudio (Icecast2), streaming de vídeo (NGINX-RTMP), videoconferència (Jitsi Meet) i auditories d'amplada de banda. |
|  `BASE_DE_DADES.md` | Disseny i administració del motor de dades corporatiu. | Model Relacional, servidor MariaDB securitzat, 15 taules relacionals, rols d'accés, triggers d'auditoria i backups automàtics. |

---

## 3. Resum Executiu dels Blocs Tècnics

### Bloc 1.1: Proposta CPD — Infraestructura Física
Aquest bloc defineix els fonaments de la infraestructura física d'InnovateTech, plantejada per a una execució local ideal alineada posteriorment amb els serveis en el núvol.
* **Ubicació i Disseny:** Sala de $6\times4\times3\text{ m}$ situada en un soterrani per millorar l'aïllament climàtic i la seguretat perimetral. Compta amb terra i sostre tècnics, parets ignífugues i porta blindada d'acer.
* **Seguretat Física:** Control d'accés mitjançant doble factor d'autenticació (targeta RFID + empremta dactilar), circuit tancat de televisió (6 càmeres CCTV) i extinció automàtica d'incendis mitjançant gas $\text{CO}_2$.
* **Seguretat Lògica i Còpies:** Disseny de xarxa segmentat en VLANs, emmagatzematge massiu en configuracions RAID i implementació de l'estratègia de **backup 3-2-1** (3 còpies, 2 suports diferents [NAS local + AWS S3] i 1 còpia externa *offsite*).

### Bloc 1.2: Infraestructura de Xarxes i Sistemes (AWS i Ansible)
Es descriu el desplegament logístic de la infraestructura sobre AWS, reduint els temps de configuració manual mitjançant l'automatització.
* **Desplegament en AWS:** Creació de 5 servidors clau (Instàncies EC2 amb Ubuntu) securitzats mitjançant claus criptogràfiques SSH (`innovate-tech-key`) i *Security Groups* que reprimeixen qualsevol trànsit extern no desitjat.
* **Automatització amb Ansible:** Desenvolupament d'un *Playbook Mestre* que configura automàticament els serveis web (Apache), de transferència (SFTP segur), instal·lació de mòduls dinàmics (PHP) i resolució de noms interna.
* **Gestió d'Identitats i Seguretat:** Implementació d'un servidor central **OpenLDAP** integrat amb els clients mitjançant el dimoni **SSSD**. Centralització d'auditories mitjançant **Rsyslog** i fortificació web activant el protocol HTTPS (port 443) a través d'OpenSSL amb claus RSA de 2048 bits (`web-ssl.conf`).

### Bloc 2: Serveis de Xarxa i Internet (Multimèdia)
Enfocat en la implementació de serveis d'alta demanda de transferència sobre el servidor dedicat `innovatetech-media` (srv4, instància `t2.medium` en la regió `eu-west-1`).
* **Streaming d'Àudio (Icecast2):** Configuració de dos canals independents emesos de forma contínua mitjançant instàncies virtuals de **ffmpeg** controlades per serveis de *systemd*: el canal corporatiu (MP3 a 128 kbps) i el canal de formació (OGG Vorbis a 96 kbps).
* **Streaming de Vídeo (NGINX-RTMP):** Compilació i configuració de NGINX amb el mòdul RTMP per processar fluxos de vídeo en directe, distribuint el contingut mitjançant el protocol **HLS** integrat en un reproductor web d'accés corporatiu.
* **Videoconferència (Jitsi Meet):** Desplegament de la plataforma de reunions integrada al domini amb xifrat SSL i configuració NAT específica per resoldre l'encaminament de paquets de veu i vídeo d'AWS (*Jitsi Videobridge*). Inclou un complet test d'amplada de banda amb *iperf3* que valida la viabilitat tècnica en situacions d'alta càrrega.

### Bloc 0377: Administració de Bases de Dades (MariaDB)
Disseny i securització del repositori de dades relacional que sosté tota la lògica operativa de l'organització.
* **Model i Estructura:** Creació de la base de dades `innovatetech` (sota motor MariaDB i codificació `utf8mb4_unicode_ci`) amb un total de **15 taules** estructurades en 5 àrees funcionals: Personal, Comunicació Interna, Contingut Multimèdia, Vendes/Clients i Auditoria.
* **Seguretat i Rols:** Aplicació estricta del principi de mínim privilegi mitjançant la creació de rols departamentals específics i el desenvolupament d'un *script en Bash* per a l'alta automatitzada d'usuaris amb contrasenyes segures.
* **Auditoria Contínua i Resiliència:** Programació de *triggers* de control que intercepten accessos no autoritzats i en guarden un registre automàtic a la taula `avisos` (usuari, taula afectada, operació i timestamp). Addicionalment, s'ha implementat un *Event scheduler* periòdic que automatitza backups diaris garantint un RPO de 24 hores.

---

## 4. Stack Tecnològic Global

Per dur a terme aquest projecte transversal, s'ha seleccionat i combinat un conjunt d'eines líders en el sector d'infraestructures TIC:

```
  [ Cloud Computing ] ──────► Amazon Web Services (AWS EC2, S3, VPC)
  [ Automatització IaC ] ───► Ansible (Playbooks, Roles, Jinja2)
  [ Gestió d'Identitats ] ──► OpenLDAP + SSSD (Autenticació Centralitzada)
  [ Serveis Web i Seg ] ───► Apache2 + OpenSSL (HTTPS / TLS RSA 2048)
  [ Serveis Multimèdia ] ──► Icecast2 + NGINX-RTMP + Ffmpeg + Jitsi Meet
  [ Base de Dades ] ────────► MariaDB (SQL, Triggers, Events, Bash Scripts)
  [ Sistema Operatiu ] ─────► Ubuntu Server 22.04 LTS
```


---

## 5. Arquitectura i Seguretat Integrada

La seguretat s'ha dissenyat com un sistema de **defensa en capes** que interconnecta tots els blocs del projecte:
1. **Capa Física / Perimetral:** Disseny de portes blindades amb doble factor, combinat amb els *Security Groups* d'AWS actuant com a tallafocs de xarxa per tancar tots els ports no essencials.
2. **Capa d'Autenticació:** Eliminació de contrasenyes per defecte de l'administrador i centralització absoluta dels usuaris de l'empresa en el directori LDAP.
3. **Capa d'Aplicació i Transport:** Tot el trànsit crític viatja xifrat (protocol HTTPS per a la web, protocol SFTP per a fitxers i SSL/TLS per a les videoconferències de Jitsi).
4. **Capa de Dades i Auditoria:** Control intern a la base de dades mitjançant triggers de denegació de permisos que alimenten un log central d'incidents (`avisos`), i còpies de seguretat diàries exportades de manera externa seguint el model 3-2-1.

---

## 6. Conclusions del Projecte

El desenvolupament de la infraestructura global d'**InnovateTech** ha permès consolidar de manera pràctica les competències d'administració de sistemes, xarxes i bases de dades en un escenari corporatiu real. 

L'ús d'AWS ens ha aportat l'escalabilitat i alta disponibilitat de recursos, l'adopció d'Ansible ha demostrat com es poden mitigar els errors humans reduint hores de configuració manual a pocs segons d'execució, i les implementacions multimèdia i de bases de dades garanteixen que l'empresa disposi d'eines de col·laboració eficients sota un entorn auditat i completament fortificat contra amenaces externes i internes.
