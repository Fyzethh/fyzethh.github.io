---
title: Configuración Alta disponibilidad (HA) con firewalls fortigate de fortinet.
date: 2024-10-16
categories: redes 
tags:
  - Redes
  - Fortinet
  - Firewall
  - HA
author: fyzethh
toc: true
img_path: /assets/img/avatar.jpg
---

Este proyecto ha sido desarrollado utilizando el software GNS3, con el objetivo de implementar alta disponibilidad en routers para asegurar la continuidad operativa en una red. 
Se ha diseñado una topología de red que incluye una LAN con máscara 24, una WAN con máscara 24 y una red local para el servicio web. El sistema está compuesto por dos firewalls Fortinet que se conectan a la red WAN para acceder al servicio HTTP hospedado en un servidor Windows. 
Los firewalls recibirán una dirección IP asignada por un router Cisco con un dhcp pool configurado este router simula ser el ISP. 

El objetivo principal es garantizar que, en caso de una falla en el firewall principal, exista un firewall de respaldo capaz de manejar el tráfico, asegurando así la continuidad operativa de la red. 

Componentes Utilizados: 

    2 Firewalls Fortigate – modelo 6.4.15-1 

    1 Windows server 2022  

    1 Router modelo cisco – modelos 7200 153-3.XB12 

    2 switch 1

Topología de red: 

    LAN: 192.168.0.0/24 

    WAN: 200.100.50.0/24 

    Red local de Window Server 2022: 100.100.100.0/24 


![topologia](/assets/img/captures/vrrp-fortigate/1.png)


Los pasos para realizar a grandes rangos son los siguientes: 

    Configuración servicio http web. 

    Configuracion router DHCP WAN. 

    Configuracion firewalls. 

    Configuracion protocolo VRRP. 

    Pruebas finales 

1.- Configuración servicio IIS windows server 

En windows server se configuro el servicio IIS que es el servicio de propietario de microsoft como se puede ver en la imagen al activar el servicio iis nos viene con una pagina por defecto hosteada en el puerto 80 en nuestro servidor 

![capture-iss](/assets/img/captures/vrrp-fortigate/Screenshot%20from%202024-07-15%2020-14-42.png)
 

Página por defecto de windows server 2022 

2.-  Configuración router DHCP WAN. 

Configuramos el pool dhcp y sus respectivas interfaces 

 ![capture-router-cisco](/assets/img/captures/vrrp-fortigate/Screenshot%20from%202024-07-15%2020-16-16.png)


3.- configuración firewalls 

Configuración iniciales interfaz firewall maestro 

```bash
config system interface 
    edit "port1"
        set vdom "root" 
        set mode dhcp 
        set allowaccess https http 
        set type physical 
        set description "only http and https allowed for web services" 
        set alias "External" 
        set snmp-index 1 
    next 
 
   edit "port2" 
        set vdom "root" 
        set ip 192.168.0.1 255.255.255.0 
        set allowaccess https http 
        set type physical 
        set description "only http and https" 
        set alias "Internal" 
        end 
    End
```
Configuracion iniciall firewall backup: 
```bash
config system interface 
    edit "port1" 
        set vdom "root"
        set mode dhcp 
        set allowaccess ping https ssh fgfm 
        set type physical 
        set snmp-index 1 
    next 

    edit "port2" 
        set vdom "root"
        set ip 192.168.0.2 255.255.255.0 
        set allowaccess ping https ssh http telnet 
        set type physical 
        set snmp-index 2 
    next 
```

 
Ahora que tenemos configuradas las interfaces de nuestros firewalls procederemos a la implementación del protocolo vrrp. 

 

4.- Configuracion protocolo VRRP.   

Todos los siguientes pasos se deben realizar en ambos firewalls. 

Según la documentación oficial de Fortinet, los pasos necesarios son los siguientes: 

    1. Add a virtual VRRP router to the internal interface of each of the FortiGates and routers. This adds the FortiGates and routers to the same VRRP domain. 

    2- Set the VRRP IP address of the domain to the internal network default gateway IP address. 

    3. Give one of the VRRP domain members the highest priority so it becomes the primary (or master) router and give the others lower priorities so they become backup routers. 

Agregamos un router virtual en VRRP en ambos firewalls con una ip estatica, esta vez elegiremos la ip 192.168.0.101 de nuestro rango 192.168.0.0/24 

Configuraciones en Firewall maestro: 
```bash
config system interface 
    edit port2 
        set vrrp-virtual-mac enable  
            # config vrrp 
            edit 5 
                set vrgrp 1 
                set vrip 192.168.0.101 
                set priority 101 
                set adv-interval 1 
                set start-time 3   
                set preempt enable  
                set vrdst 200.100.50.31  
                set status enable             
            end 
End 
```



Configuraciones Firewall backup: 
```bash
config system interface 
     edit port2 
        set vrrp-virtual-mac enable 
            # config vrrp 
            edit 1 
                set vrgrp 1            
                set vrip 10.31.101.101 
                set priority 100 
                set adv-interval 1 
                set start-time 3          
                set preempt enable        
                set status enable 
            end 
End 
```

Como se puede apreciar en los comandos anteriores dejamos al router maestro con prioridad 101 y el backup con prioridad 100. 

Si miramos nuestras configuraciones VRRP en nuestros firewalls fortigate podremos ver lo siguiente: 

Maestro: 

 ![capture-router-cisco](/assets/img/captures/vrrp-fortigate/Screenshot%20from%202024-07-15%2020-16-48.png)

 

Backup:

 ![capture-router-cisco](/assets/img/captures/vrrp-fortigate/Screenshot%20from%202024-07-15%2020-17-01.png)

 

En nuestros firewalls, configuraremos una ruta por defecto para que todo paquete cuyo destino no sea conocido por el router sea redirigido a esta ruta.  

Es esencial implementar esta configuración en ambos firewalls.  

En Fortigate, esto se realiza de la siguiente manera: 
```bash
config router static 
    edit 1
        set gateway 200.100.50.1 
        set device "port1" 
    next 
end 
```
Con estos pasos, habremos implementado el protocolo VRRP entre ambos firewalls Fortigate, designando uno como maestro y el otro como respaldo. A continuación, procederemos a realizar las pruebas. Para ello, utilicé un componente de GNS3 llamado Webclient, el cual incluye un navegador para visitar la web alojada en nuestro servidor Windows. 

El paso final es configurar la puerta de enlace predeterminada apuntando hacia nuestro nuevo router virtual en nustro componente *WEBTERM-1*, configurado con VRRP en la dirección “192.168.0.101”. 

 
5.- Pruebas finales 

Para realizar las pruebas, podemos apagar el firewall activo y observar cómo el firewall de respaldo asume la funcionalidad de manera automática. Utilizaremos Wireshark para analizar los paquetes que llegan a nuestro servidor Windows, verificando así la continuidad del servicio y el correcto funcionamiento de la configuración VRRP. 
 

![capture-router-cisco](/assets/img/captures/vrrp-fortigate/Screenshot%20from%202024-07-15%2020-17-21.png)

 
Como se puede apreciar en la captura, la fuente del primer request es 192.168.0.32. Al apagar el router maestro, el segundo router comenzó a recibir el tráfico, como se muestra en el segundo request procesado con la fuente 192.168.0.31. 

Al reactivar el primer router, este vuelve a ser el maestro debido a su mayor prioridad en comparación con el router de respaldo. Esto se observa en un tercer request, donde la fuente vuelve a ser nuestro router primario, “192.168.0.32”. 

![capture-router-cisco](/assets/img/captures/vrrp-fortigate/Screenshot%20from%202024-07-15%2020-17-36.png)
