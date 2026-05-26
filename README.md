# 🏢 Memoria Técnica: Infraestructura de Redes y Sistemas
**Proyecto Transversal - Grupo 4 (Innovate Tech)**

**Integrantes:** Adam Benmansor, Leonel Coello, Oriol Coll, Victor Barreda.

---

## 📌 1. Introducción
En este documento explicamos de forma detallada cómo hemos diseñado, desplegado y configurado la infraestructura de red corporativa para nuestro Proyecto Transversal (Innovate Tech). 

El objetivo principal de este proyecto no era simplemente encender máquinas virtuales, sino simular un entorno empresarial completo y realista utilizando **Amazon Web Services (AWS)**. Hemos priorizado la aplicación de buenas prácticas del sector, tales como la infraestructura como código (automatizando tareas repetitivas), la centralización del control de usuarios y auditoría, y la fortificación perimetral de los servidores. Para ello, hemos combinado de forma integral herramientas de alto nivel como Ansible, LDAP, DNS y OpenSSL.

---

## ☁️ 2. Despliegue de la Infraestructura en AWS
La base física de nuestro proyecto se asienta sobre la nube de Amazon Web Services (AWS). En lugar de usar redes locales convencionales, aprovechamos el cloud computing para disponer de alta disponibilidad.

* **Claves de Acceso:** Generamos un par de claves criptográficas (`innovate-tech-key`) de tipo RSA/ED25519. Este archivo `.pem` es nuestro pase de seguridad exclusivo; sin él, es imposible acceder a las terminales, mitigando así ataques de fuerza bruta.
* **Security Groups (Firewall):** Siguiendo el principio de "mínimo privilegio", configuramos nuestro firewall de AWS para bloquear todo el tráfico externo por defecto. Solo permitimos el acceso a los puertos estrictamente necesarios (como el 22 para SSH desde nuestras IPs autorizadas, o el 80/443 para tráfico web). A nivel interno, habilitamos que todas las máquinas de nuestra VPC puedan comunicarse libremente entre sí.

<img width="899" height="726" alt="image" src="https://github.com/user-attachments/assets/d234b774-4701-42c6-ae89-3c96f9de14af" />

<img width="1554" height="627" alt="image" src="https://github.com/user-attachments/assets/6c2219c2-e32a-406a-bf74-2c38219cf65f" />

* Levantamos **5 servidores clave** (Instancias EC2) con Ubuntu, cada uno con un rol específico:
    * **Màquina 1: Servei Web - Apache / SFTP (Ansible)**
    * **Màquina 2: LDAP** (y servidor de resolución DNS)
    * **Màquina 3: Centralització de logs (Ansible)** (Nodo de control maestro)
    * **Màquina 4: Servidor de Streaming**
    * **Màquina 5: Base de Dades (Maria DB)**


<img width="1506" height="305" alt="image" src="https://github.com/user-attachments/assets/ae660041-74a3-48ce-b473-daf028ea0822" />

---

## 🤖 3. Automatización de la Infraestructura con Ansible
En una empresa real, ir máquina por máquina instalando servicios a mano es ineficiente y muy propenso a errores humanos. Por ello, hemos implantado **Ansible** para centralizar la configuración mediante código (Infrastructure as Code).

En la **Màquina 1**, me encargué de dar de alta al usuario `admin-itb` con permisos de administrador (sudo) y le configuré nuestras llaves SSH. Después, desde nuestra **Màquina 3** (el cerebro de Ansible), definimos el inventario de hosts e iniciamos la magia. Escribí y lancé nuestro primer Playbook (`mi_config.yml`). Este script se conectó por SSH a la Màquina 1, actualizó los paquetes del sistema e instaló Apache2 de forma totalmente desatendida y automática.

<img width="689" height="662" alt="image" src="https://github.com/user-attachments/assets/a1bc463e-b2d3-45fb-8ffa-2ab478979615" />

<img width="673" height="418" alt="image" src="https://github.com/user-attachments/assets/14d6e738-6402-4571-a267-c6da68590207" />


---

## 🔐 4. Servidor Central de Identidades: LDAP
Para evitar el caos administrativo de tener cuentas de usuario dispersas e independientes en cada servidor, implantamos un directorio activo utilizando el protocolo LDAP (Lightweight Directory Access Protocol) en la **Màquina 2**. 

Configuré nuestro propio árbol de dominio llamado `innovatetech.itb.cat`. Para poblar este directorio, inyecté en el servidor una serie de archivos de estructura `.ldif` para crear las unidades organizativas (OUs) de *grups* y *usuaris*. A modo de validación, creamos nuestro primer usuario corporativo 100% centralizado: `sftpuser`, asignándole un ID de grupo, un directorio *home* y su respectiva contraseña.

<img width="942" height="557" alt="image" src="https://github.com/user-attachments/assets/b893ad65-c8c0-4b81-8be1-bd2dabd1a92c" />

<img width="1384" height="194" alt="image" src="https://github.com/user-attachments/assets/099b0c25-ec58-4d96-8bc0-a46e62457359" />

---

## 📁 5. Integración del Cliente LDAP (SSSD) y SFTP Seguro
El LDAP de la M2 es inútil si el resto de máquinas no saben cómo preguntarle quiénes son los usuarios. El siguiente paso fue vincular nuestro servidor web con esta base de identidades.

Instalé el demonio **SSSD** en la **Màquina 1** y lo configuré para que leyera e interpretara a los usuarios alojados en la Màquina 2 de forma transparente.

<img width="779" height="100" alt="image" src="https://github.com/user-attachments/assets/e049f259-1b20-4395-a002-bf2110a29a1f" />


A continuación, implementamos un servicio de transferencia de archivos seguro (**SFTP**). Para garantizar que un usuario subcontratado o externo no pudiera cotillear los archivos críticos del sistema operativo Ubuntu, apliqué una política estricta de seguridad llamada **Chroot Jail** (`ChrootDirectory` en SSH). Esto "enjaula" al `sftpuser` cuando se conecta, limitando su visión exclusivamente a su carpeta asignada `/home/sftpuser/pujades`. Finalmente, modificamos el *DocumentRoot* de Apache para que cargase el contenido de esa misma carpeta de cara al público.

<img width="853" height="260" alt="image" src="https://github.com/user-attachments/assets/5db3285b-74ea-4b36-b1ed-a28f3ac0e010" />

<img width="530" height="544" alt="image" src="https://github.com/user-attachments/assets/b7b76914-070f-4b1c-ae94-fb4c5c8e1cf2" />


---

## 📡 6. Auditoría y Centralización de Logs
La trazabilidad es clave en la ciberseguridad. Para evitar tener que acceder a 5 servidores distintos en busca de fallos o accesos no autorizados, configuramos la **Màquina 3** como nuestro SIEM (Gestor de eventos e información) básico.

En la Màquina 3, descomenté los módulos `imudp` e `imtcp` en la configuración de `rsyslog` para que abriera el puerto 514 y escuchara el tráfico de red. Posteriormente, con un nuevo Playbook de Ansible (`configurar_logs.yml`), modifiqué de golpe la configuración interna de todos los demás servidores. La regla inyectada obligaba a cada máquina a enviar una copia en tiempo real de cualquier evento del sistema (errores, reinicios, logins) directamente a la Màquina 3. Ahora disponemos de un punto único de monitorización.

<img width="1054" height="657" alt="image" src="https://github.com/user-attachments/assets/f1c1ec05-c572-4332-a3c8-464f2fbede4b" />

<img width="1195" height="655" alt="image" src="https://github.com/user-attachments/assets/b89f859e-b27a-4182-9077-637c4a48124e" />


---

## 🐘 7. Integración Dinámica: Instalación de PHP
Para garantizar que nuestro portal no fuera solo una vitrina de HTML estático y tuviera el potencial de interactuar con aplicaciones modernas, doté a la infraestructura web de capacidades de procesamiento dinámico.

En el servidor web (**Màquina 1**), procedí a la instalación del motor **PHP** junto al paquete de enlace `php-mysql`. Esta librería es el puente vital que permitirá que cualquier aplicación web alojada en la M1 pueda consultar y escribir datos directamente en nuestra **Màquina 5: Base de Dades (Maria DB)**. Para verificar que el módulo se había cargado correctamente en Apache, generé un archivo de diagnóstico `info.php` que nos confirmó que el entorno de desarrollo estaba listo.

> **📸 INSERTA AQUÍ LA CAPTURA 12:** Captura del navegador accediendo a `/info.php` mostrando la tabla azul y lila con la configuración de PHP.

---

## 🌍 8. Resolución de Nombres: Servidor DNS Interno
En una red con múltiples servicios, memorizar e introducir direcciones IP públicas o privadas cada vez que las máquinas necesitan hablar entre ellas es inviable. Para solucionar este problema, montamos un servidor **DNS** autoritativo basado en `bind9` dentro de la **Màquina 2**. 

Definí un archivo de zona directa para el dominio privado `itb.local`. En ese archivo, vinculé nombres lógicos y descriptivos a cada nodo (`web`, `ldap`, `logs`, `stream`, `bd`) para que apuntaran a sus respectivas IPs privadas. Una vez la guía telefónica estuvo lista, programé otro Playbook de Ansible (`configurar_dns.yml`) que modificó el archivo `systemd-resolved` de todas las máquinas en segundos, obligándolas a consultar a la Màquina 2 para cualquier traducción de dominio.

<img width="1022" height="668" alt="image" src="https://github.com/user-attachments/assets/e859b76f-02b3-4409-8749-59de28225b41" />

<img width="1508" height="381" alt="image" src="https://github.com/user-attachments/assets/da0d2dc5-360c-41ad-9eff-805bc67c92cf" />


---

## 🔗 9. Integración Global al Dominio (El Playbook Maestro)
Para consolidar la infraestructura de identidad, faltaba extender la configuración del SSSD a todas las terminales del proyecto. Preparé el Playbook maestro: `unir_dominio.yml`. 

Con la sola pulsación del *Enter*, Ansible viajó simultáneamente a la **Màquina 1**, **Màquina 4** y **Màquina 5**, instaló los paquetes cliente de LDAP, inyectó la ruta de la M2 como autoridad de identificación y, como detalle técnico vital, activó el módulo `mkhomedir` mediante PAM. Esta funcionalidad asegura que cuando cualquier empleado registrado en LDAP inicia sesión por primera vez en un servidor, el sistema operativo le genera al instante su carpeta de trabajo personal.

<img width="1045" height="771" alt="image" src="https://github.com/user-attachments/assets/83462072-2888-405d-9b7e-24336c86401e" />

---

## 🛡️ 10. Fortificación Criptográfica HTTPS (SSL)
El cierre técnico de nuestro despliegue fue la fortificación de la comunicación web. Desplegar servicios corporativos únicamente por HTTP (puerto 80) deja el tráfico vulnerable a escuchas o interceptaciones, por lo que necesitábamos activar el protocolo HTTPS (puerto 443).

Al tratar con un dominio privado (`web.itb.local`) inaccesible desde el internet global, las autoridades certificadoras comerciales no podían validarnos. Como administrador de sistemas, generé mi propio **certificado criptográfico autofirmado** (`web-itb.crt` y `.key`) haciendo uso del paquete criptográfico OpenSSL, empleando el robusto algoritmo RSA de 2048 bits. Finalmente, diseñé y habilité un nuevo archivo de bloque *VirtualHost* en Apache (`web-ssl.conf`) indicándole las rutas de las llaves, consiguiendo así que todas las comunicaciones de nuestro portal web estén cifradas de extremo a extremo.

<img width="1186" height="634" alt="image" src="https://github.com/user-attachments/assets/c66e1f02-cc2e-4d4d-a945-84df377af633" />

<img width="667" height="548" alt="image" src="https://github.com/user-attachments/assets/0d282eda-82d3-400e-90bc-f326b10bbf80" />

<img width="736" height="458" alt="image" src="https://github.com/user-attachments/assets/06380633-8770-499f-aebd-16c2eceaeb22" />

---

**Conclusión**
El desarrollo exhaustivo de este Proyecto Transversal nos ha servido para simular las problemáticas y arquitecturas reales de los entornos Cloud empresariales. La adopción de AWS nos proporcionó el escalado, Ansible redujo las horas de configuración manual a unos pocos segundos de ejecución, LDAP nos brindó el control absoluto sobre los accesos y la centralización del DNS y los logs simplificó enormemente el mantenimiento. Con la encriptación y los enjaulados SFTP añadidos, entregamos una arquitectura funcional, redundante y altamente fortificada.
