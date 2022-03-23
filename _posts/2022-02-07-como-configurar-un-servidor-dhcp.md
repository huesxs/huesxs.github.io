---
title: Cómo configurar un servidor DHCP
date: 2022-02-07 17:27:30 +/-TTTT
categories: [Sysadmin, Protocolos, Teoria]
tags: [protocolos, sysadmin, networking, teoria, knowledge]     # TAG names should always be lowercase
---


**Índice**<br/>
- [¿Qué es el protocolo DHCP?](#qué-es-el-protocolo-dhcp)
    - [El funcionamiento del protocolo DHCP](#el-funcionamiento-del-protocolo-dhcp)
    - [Métodos de asignación bajo el protocolo DHCP](#m%C3%A9todos-de-asignaci%C3%B3n-bajo-el-protocolo-dhcp)
- [Nuestro servidor DHCP en Ubuntu](#nuestro-servidor-dhcp-en-ubuntu)
    - [Introducción](#introducci%C3%B3n)
    - [Instalación del servidor DHCP](#instalaci%C3%B3n-del-servidor-dhcp)
    - [Configurar nuestro servidor DHCP](#configurar-nuestro-servidor-dhcp)
    - [Comprobación de la configuración y funcionamiento](#comprobaci%C3%B3n-de-la-configuraci%C3%B3n-y-funcionamiento)
- [Conclusión](#conclusi%C3%B3n)

# ¿Qué es el protocolo DHCP?

El DHCP (Dynamic Host Configuration Protocol) es un protocolo que admite que un activo conectado a una red se le pueda asignar su propia configuración alojada en un servidor, de ahí, la distinción entre cliente y servidor en cuanto a DHCP se refiere. Todo esto se realiza de forma dinámica, es decir, no tendremos que ajustar ningún tipo de configuración especial de manera manual en los equipos clientes que se incluyan en los ajustes del servidor. Básicamente con fines de simplificación a la hora de administrar la red en cuestión.

![img-description](/images/dhcp-kekw.jpeg)
_El protocolo DHCP be like... pero bueno, ya tendremos tiempo en indagar DORA_

## El funcionamiento del protocolo DHCP

Perfecto, aunque ¿cómo lo podríamos definir de forma amena? El DHCP se basa en cuestión de peticiones de cliente a servidor y viceversa, me explico, el servidor cuando está en marcha está a la espera de peticiones de los clientes que constan en su configuración. 

Cuando uno de estos clientes enciende el equipo o pone en marcha su dispositivo *y no sabe cuál es su IP*, de forma automática envia una petición de tipo `DHCP Discover` al servidor que es el encargado de enviar una respuesta de tipo `DHCP Offer` por lo que oferta y asigna lo siguiente: la dirección IP, la puerta de enlace (gateway) y los servidores DNS que apliquen.

De esta forma se establece una comunicación de cliente a servidor (y viceversa), como bien he comentado anteriormente. Por lo tanto, una vez aceptada la respuesta del servidor, el cliente envia de nuevo una respuesta de tipo `DHCP Request` asumiendo la configuración ofrecida por el servidor.

Paralelamente, el servidor, mantiene en su pequeña caché de este servicio y/o protocolo toda una lista de aquellos clientes con las IP asignadas en este *challenge*, con este fin, evita que se repita el proceso con una *doble asignación* de IP con otro cliente.

Si aún así no ha quedado claro, te dejo este esquema que he hecho, el cual me ha costado años de dedicación y 500 horas de cursos de Photoshop CS6:

![img-description](/images/dhcp_explain.png)
_Comunicación entre cliente y servidor bajo el protocolo DHCP_

1. El cliente enciende su dispositivo y 'está perdido' necesita su dirección IP, por lo que hace una petición `DHCP Discover` al servidor.
2. El servidor 'encamina' al cliente bajo una respuesta `DHCP Offer` que incluye su dirección IP, la *gateway* y los servidores DNS.
3. El cliente acepta la oferta del servidor bajo una petición `DHCP Request`, asume la configuración y se le asigna lo anteriormente dicho.
4. El servidor guarda la información del proceso en su caché y *recuerda* la IP asignada al cliente para futuros *challenges*.

## Métodos de asignación bajo el protocolo DHCP

Tenemos dos tipos distintos de asignación DHCP: el manual y el automático.

- **Manual** --> Bajo este método, la dirección IP se asigna en base a la MAC del dispositivo lo cual asegura que una máquina en concreto tenga una IP estática asignada por el mismo servidor DHCP. Por lo que el mismo servidor DHCP siempre estará enviando una configuración estática al cliente dependiendo de su MAC.
- **Automático** --> Esta manera consta en la asignación automática del conjunto de direcciones IP, dependiendo de la manera que vayan enviando las peticiones los clientes expuestos en la configuración DHCP y recibiendo las respuestas por parte del servidor. Se puede bifurcar en dos categorias basadas en el tiempo: un lapso de tiempo fijado o permanente y/e infinito.
    - Tiempo fijado --> Cuando un cliente queda obsoleto o fuera de la red gestionada por el mismo servidor DHCP, la configuración expira y se 'libera'. ¿Qué quiere decir esto? Básicamente, la dirección IP que tenía fijada este mismo cliente, queda disponible para otros futuros clientes de la red. Por lo que esta IP se agrupa con las demás disponibles para que el mismo servidor pueda ofrecerla. En cambio, si el cliente quiere mantener su misma IP, bajo esta configuración, tiene que 'renegociar' con el servidor en el lapso de tiempo ajustado para persistirla.
    - Tiempo infinito --> De forma sencilla, se mantiene de forma permanente la IP asignada por el servidor al cliente.

![img-description](/images/manual-auto.jpeg)
_Es algo como así, la verdad..._

# Nuestro servidor DHCP en Ubuntu

## Introducción

Antes de empezar, cabe recalcar que estaré utilizando el siguiente laboratorio para *testear* y montar nuestro DHCP Server con un cliente Windows:

- Ubuntu Server 20.04.1 LTS (Focal Fossa) [DHCP Server] (*) Ojo, cabe recalcar que las versiones Desktop y Server son ligeramente distintas, te recomiendo <a href="https://www.makeuseof.com/tag/difference-ubuntu-desktop-ubuntu-server/">leer este artículo.</a>
- Windows XP Professional SP2

También configuraremos mediante las opciones de red del mismo VirtualBox, una red interna que compartan tanto nuestro Ubuntu Server como el cliente Windows, quedándose así:

![img-description](/images/ubuntuserver.png)
_Configuración de red de Ubuntu Server_

![img-description](/images/windows.png)
_Configuración de red de Windows XP_

## Instalación del servidor DHCP

En nuestra máquina con Ubuntu, ejecutaremos la siguiente orden para instalar el *package* `isc-dhcp-server`:

```shell
sudo apt install isc-dhcp-server
```
{: .nolineno }

## Configurar nuestro servidor DHCP

Siendo precavidos, tenemos que tener en cuenta que nuestro fichero de configuración es ``/etc/dhcp/dhcpd.conf``, por lo que haremos un respaldo:

```shell
sudo mv /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd_backup.conf
```
{: .nolineno }

Tras hacer este pequeño *backup* de la configuración inicial, crearemos el archivo de configuración con el mismo nombre y comenzaremos a editar el fichero de configuración con cualquier IDE, yo optaré por ``nano``:

```shell
sudo touch /etc/dhcp/dhcpd.conf
sudo nano /etc/dhcp/dhcpd.conf
```
{: .nolineno }

Una vez abierto, copiaremos el siguiente contenido dentro del archivo de configuración:

```shell
default-lease-time 600;
max-lease-time 7200;
authoritative;
 
subnet 192.168.1.0 netmask 255.255.255.0 {
 range 192.168.1.10 192.168.1.25;
 option routers 192.168.1.254;
 option domain-name-servers 192.168.1.1, 192.168.1.2;
}
```

Para ser más exhaustivos investigaremos cada opción establecida en el fichero de configuración, para ver qué estamos poniendo:

- ``default-lease-time``: Es el lapso de tiempo de concesión, es decir, el tiempo predefinido a conceder la petición o inicializar el *challenge* (``DHCP Offer``) en caso de que el cliente no indique ningún tiempo predeterminado.
- ``max-lease-time``: Es el lapso de tiempo máximo de concesión, en este caso está predefinido a 7200 segundos o mejor dicho 2 horas.
- ``authoritative``: Indica que es el servidor DHCP *oficial* para la red local, quién tiene la autoridad básicamente.
- En este caso el servidor está configurado para servir las IP en un rango del 192.168.1.10 al 192.168.1.25, o para simplificar, 15 hosts máximos en nuestra red.
- El servidor enviará al cliente la información de asignar la IP 192.168.1.254 como su puerta de enlace predeterminada y las IP 192.168.1.1 y 192.168.1.2 como servidores DNS.

Una vez definamos esta configuración, tenemos que tener bien en claro que si queremos ajustar una IP estática a nuestro cliente y queremos garantizarla, como bien he explicado antes en los tipos de ajustes DHCP, debemos de utilizar su MAC. Siendo una máquina virtual nuestro cliente Windows, podemos obtener la MAC muy fácil desde la configuración de red del mismo hipervisor, en este caso VirtualBox:

![img-description](/images/windowsmac.png)
_MAC de nuestro cliente Windows_

Ojo con esto último, ya que deberemos de elegir la MAC de la interfaz de red con la cual queramos configurar nuestro servidor DHCP. También es de tener en cuenta el hostname de nuestra máquina cliente ya que configuraremos por cuantos clientes tengamos un bloque distinto en el fichero ``/etc/dhcp/dhcpd.conf``.

![img-description](/images/hostname.png)
_Hostname de nuestro cliente Windows (windows)_

Avisadas ya estas dos cosillas, podemos meter este bloque debajo de nuestra configuración del servidor DHCP:

```shell
host windows {
 hardware ethernet 08:00:27:a7:60:28;
 fixed-address 192.168.1.10;
}
```
Explicando cada opción del bloque:

- ``host``: Indicamos el hostname, en este caso *windows*.
- ``hardware ethernet``: Indicamos la MAC de la máquina, que hemos sustraido del hipervisor.
- ``fixed-address``: Ponemos la IP estática a nuestro gusto que servirá el servidor al cliente Windows.

Por lo que nuestro fichero de configuración ``/etc/dhcp/dhcpd.conf`` tiene que asemejarse de tal forma:

```shell
default-lease-time 600;
max-lease-time 7200;
authoritative;
 
subnet 192.168.1.0 netmask 255.255.255.0 {
 range 192.168.1.10 192.168.1.25;
 option routers 192.168.1.254;
 option domain-name-servers 192.168.1.1, 192.168.1.2;
}

host windows {
 hardware ethernet 08:00:27:a7:60:28;
 fixed-address 192.168.1.10;
}
```

Claro que si queremos que la IP se asigne de forma dinámica, podemos omitir los bloques estáticos anteriormente explicados, siempre y cuando se compartan la misma interfaz: tanto el servidor DHCP como el/los cliente/s DHCP. Por lo que se ofrecerá una IP del rango ajustado en la configuración, (x.x.x.10 al x.x.x.25).

Finalmente, como el protocolo DHCP permite alojar distintas interfaces, podemos definir una interfaz concreta a la escucha, en mi caso será la interfaz que aloje la red interna que hemos creado `intnet`. 

En mi equipo servidor (Ubuntu), me indica que la interfaz tiene el nombre de 'enp0s8' haciendo uso de ``ip a``:

![img-description](/images/interfaz.png)
_Interfaz de la red interna 'intnet'_

Por lo que deberemos de indicarle dicha interfaz en el fichero ``/etc/default/isc-dhcp-server``:

```shell
sudo nano /etc/default/isc-dhcp-server
```

![img-description](/images/interface.png)
_Establecemos enp0s8 como interfaz a la escucha en el DHCP Server_

Tras hacer esto, deberemos de configurar manualmente la IP estática para nuestro servidor DHCP, en la misma configuración de interfaces. Esto podemos hacerlo por GUI o via comando en el fichero  ``/etc/netplan`` (en otras distros. se puede llegar al caso de que esté en ``/lib/netplan``, ``/run/netplan`` e inclusive ``/etc/network/interfaces``):

![img-description](/images/serverconf.png)
_Configuración de red en el servidor DHCP_


## Comprobación de la configuración y funcionamiento

Una vez hecho, reiniciamos el servicio DHCP con la siguiente instrucción y vemos el estado del servicio:

```shell
sudo systemctl restart isc-dhcp-server.service
sudo systemctl status isc-dhcp-server.service
```

![img-description](/images/post.png)
_Estado del servicio isc-dhcp-server.service exitoso_

Tras acabar todo esto, se nos queda una cara tal como así:

![img-description](/images/lol.jpeg)
_Efectivamente, es mi cara ahora mismo tras escribir todo esto_

Pero... podemos comprobar que todo ha ido correctamente y efectivamente tanto el servidor Ubuntu se comunica con nuestro cliente y viceversa:

![img-description](/images/windowsip.png)
_IP asignada por reserva DHCP_

![img-description](/images/pingwindows.png)
_Comunicación exitosa de nuestra máquina Windows (cliente) al servidor Ubuntu_

- Un pequeño *disclaimer* que hay que señalar: por temas de bloqueo de paquetes ICMP debido al firewall de Windows y su AV, no podremos hacer ping predeterminadamente de un sistema Linux a un Windows. Deberemos de deshabilitar el AV y funcionalidades para evitar el bloqueo ICMP.

# Conclusión

Recapitulando todo el *post*, podemos extraer que es una gran manera de gestionar nuestra red ya sea empresarial o la doméstica si aplica, que nos da un control general sobre los dispositivos conectados (clientes) a nuestro servidor. Como también, podemos realizar diferentes tareas de adminsitración, control y compartición de recursos que pueden aplicar a cualquier máquina-cliente con cualquier tipo de sistema operativo e incluso cualquier dispositivo que tenga conexión a la red.

-- Huesos







