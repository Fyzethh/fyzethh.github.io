---
title: Virtualización  CentOS 9 stream 
date: 2024-07-24 22:17 +/-0000 
categories: CentOS Linux
tags:
  - CentosOS
  - Linux
  - ssh
author: Fyzethh
toc: true
img_path: /assets/img/avatar.jpg
---

Resumen de configuración: 

Espacio en disco: 50 GB 

Memoria RAM: 1 GB 

Configuración de red: 
    - ip estática: 10.27.27.11


El primer paso es descargar el ISO de la imagen centOS 9 stream desde su repositorio oficial, 
luego de tener la iso procederemos a montarla en nuestro programa de virtualizacion en este caso VMware

![imagen-iso](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2014-47-08.png)

Al tener la imagen ISO procederemos a crear una nueva maquina virtual desde VMWARE y agregamos la imagen descargada de nuestro CENTOS 9 stream

![imagen-iso-vmware](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2014-48-06.png)

configuramos nuestro disco duro con 50 GB, agregaremos 10 gb extra de espacio para el contenido web que aojará este servidor.
![imagen-iso-vmware-disk](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2014-51-29.png)

configuramos la memoria ram en 1GB.

![imagen-iso-vmware-disk](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2014-51-48.png)

al configurar estos aspectos de nuestra maquina virtual, veremos la pantalla de instalación como se muestra en la siguiente imagen:

![install-centos-starts](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2014-54-10.png)

procederemos a configurar nuestro usuario con permisos de administrador

![install-centos-users](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2014-55-07.png)

y nos aseguramos que estamos instalando Centos en su modo "server with GUI"
![install-centos-gui](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2014-56-51.png)

ya tenemos listos los parámetros iniciales para poder ejecutar la instalación
al presionar el botón "Begin installation", podemos ver como a empezado nuestra instalación.
![install-centos-init](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2014-57-20.png)

una vez que ya tenemos la instalación completa nos iniciamos session con nuestro usuario y podremos ver nuestra instalación de centos stream completa.

![desktop-centos9](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2015-28-39.png)


Etapa de configuración:
Los pasos a realizar son los siguientes:
 - Configurar direccionamiento estático
 - actualizar servidor
 - permitir el trafico ssh y web
 - habilitar X11-forwarding 

1.- Configurar direccionamiento estático, para realizar este paso ocuparemos la ip "10.27.27.11" nuestro default gateway sera el "10.27.27.1" que corresponde a nuestro router local y configuraremos nuestra tarjeta de red en modo "bridge", como se muestra a continuación.

![ip-statica-step1](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2015-38-05.png)
![ip-statica-step2](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2015-46-30.png)
comprobamos que podamos acceder a internet desde nuestra nueva puerta de enlace
![ip-statica-step3](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2015-48-27.png)

para actualizar nuestro servidor ocuparemos los siguientes comandos:
```bash
sudo yum -y upgrade --skip-broken
sudo yum -y update
 
```

instalación ssh y habilitar daemon:
```bash
sudo yum install -y openssh-server openssh-client
systemctl enable sshd
systemctl start sshd
```
![sshd](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2016-15-11.png)

Configuración servicio web:


Configuración servicio web:
instalación de httpd y habilitar daemon:
```bash
sudo yum install -y httpd
systemctl enable httpd
```

![httpd](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2016-22-07.png)

Configuration reglas de firewall para poder habilitar trafico hacia nuestros servicios ssh y web recién configurados:

```bash
firewall-cmd --permanent --add-service-http
firewall-cmd --permanent --add-service-https
firewall-cmd --permanent --add-service-ssh
```
![firewall-cmd](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2017-39-23.png)

Acontinuacion habilitamos  X11-forwarding editando el archivo ubicado en 
/etc/ssh/ssh_config, descimentamos la linea de X11-forwarding y al lado en lugar de un "no" lo cambiamos a un "yes", como se muestra a continuación.

![x11](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2016-04-15.png)


ahora realizamos una prueba de nuestro servicio ssh desde nuestra maquina local que es un linux mint "ubuntu" conectándonos hacia nuestro servidor que tiene la ip estática "10.27.27.11"
![ssh-test](/assets/img/captures/centos-stream/Screenshot%20from%202024-07-24%2017-21-02.png)
