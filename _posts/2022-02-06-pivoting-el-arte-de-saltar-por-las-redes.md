---
title: Pivoting, el arte de saltar por las redes
date: 2022-02-06 13:22:30 +/-TTTT
categories: [Pentesting, Pivoting, Teoria]
tags: [pivoting, pentesting, networking, tunnels, teoria, knowledge]     # TAG names should always be lowercase
---


**Índice**<br/>
- [¿Qué es el Pivoting?](#qué-es-el-pivoting)
  - [Entrando al caso ejemplificado](#entrando-al-caso-ejemplificado)
- [Metodología del pivoting](#metodología-del-pivoting)
  - [Introducción](#introducción)
  - [Enumeración](#enumeración)
  - [Herramientas](#herramientas)
  - [Conclusión](#conclusión)

# ¿Qué es el Pivoting?

El Pivoting es una técnica de escalamiento o escalado de red, utilizada por los pentesters a la hora de enumerar diferentes redes existentes conectadas a un equipo de una red específica ya vulnerada.

¿Qué quiere decir esto último? A grosso modo, es el 'arte' o la acción de saltar entre redes y/e interfaces con conectividad a la misma red de la máquina vulnerada.

## Entrando al caso ejemplificado

Por lo tanto, de manera recursiva, mediante esta técnica, podemos ir "saltando" entre dispositivos que hayamos podido vulnerar, estos comparten la misma red del primer dispositivo conjunto a otra/s interfaz/ces.

Entonces, bajo "una sucesión de Fibonacci" podríamos llegar a nuestro punto final, según la arquitectura o topología de red de la víctima y siempre que podamos comprometer el siguiente dispositivo a pivotar.

Siguiendo un ejemplo gráfico más ameno:

![img-description](/images/pivoting_explain.png)
_Ejemplo de red dónde emplearíamos pivoting_

Principalmente, una de las cosas que han quedado claras tras ver este ejemplo es que el diseño gráfico no es lo mío, mejor se lo dejamos a otr@s...

![img-description](/images/kekw.jpeg)
_Graphic design is my passion_

Siguiendo el hilo del tema, en la imagen se nos da a entender un caso donde el atacante tiene acceso a una máquina que hace frontera o limita el acceso entre Internet y una red privada empresarial. En esta última podemos encontrar un router (R1), dos ordenadores (el fronterizo PC1 y uno incluído dentro de la red aislada PC2), un teléfono VOIP (VOIP Phone) y finalmente nuestro destino final más jugoso: el servidor (server).

# Metodología del pivoting

## Introducción

Entonces, ¿cómo procedemos? Tenemos que establecernos una metodología o unos pasos a hacer a la hora de actuar bajo esta técnica:

- Encontrar un dispositivo al cual podamos saltar y/o pivotar, valga la redundancia.
- Tras pivotar, podemos enumerar diferentes vectores de ataque en el mismo dispositivo pivotado.
- Explotar las vulnerabilidades mediante el vector de ataque y volver a pivotar.

Todo un símil a un while infinito hasta que podamos acceder a nuestro punto final, el servidor.

## Enumeración

Llegados a este punto, podemos preguntarnos, bien, ¿y... cómo empiezo? ¿qué debo de hacer? Pues como todo pentester en su principio ante una máquina debe de enumerar todo aquello de interés para poder realizar en este caso, pivoting hacia su siguiente objetivo. Cosa que una vez vulneremos una máquina podamos seguir ciertos *paths* o ciertas rutas del sistema que nos lleve a una enumeración de diferentes redes coexistentes dentro de la máquina y que compartan a lo mejor, más dispositivos en la misma red privada.

Bajo mi experiencia, recomiendo siempre echar un vistazo a los archivos hosts de la máquina ya sea Windows o Linux, podemos encontrar diferentes *guesses* o pistas de ruta a seguir que nos de a entender que hay más interfaces de red compartidas en el activo:

En máquinas Windows podemos localizar el archivo en la siguiente ruta: ``C:\Windows\System32\drivers\etc\hosts`` . Por consiguiente, en Linux podremos encontrarlo en la ya conocida ruta: ``/etc/hosts``.

Otra manera de enumerar diferentes interfaces coexistentes en la misma máquina que dan conectividad a más activos, podría ser ejecutando el comando ``ip a``, con alternativa a ``ìfconfig`` ya que este último no suele estar preinstalado en las distribuciones de Linux, por lo que necesita el *package* ``net-utils``. Más de lo mismo en las máquinas Windows, podemos hacer uso del comando que viene por defecto tanto en una cmd normal como en powershell: ``ìpconfig``. 

Más opciones viables de esto último e importante a saber es que podemos utilizar la opción ``/all`` de ``ipconfig`` en Windows para descubrir servidores DNS. En Linux, es equivalente a ``/etc/resolv.conf``.

Podemos proceder también a enumerar la caché ARP del activo, con ``arp -a`` tanto en máquinas con sistema operativo Linux o Windows. Lo que nos puede reportar nuevas direcciones IP que puedan tener conectividad con la máquina.

A relación con esto podemos ejecutar ``ip route`` o ``route -n`` (valga la redundancia) a fin de mostrar la tabla routing en distros. Linux. Al uso de echar un vistazo a las diferentes opciones de ``netstat`` como por ejemplo ``-alpn`` e inclusive ``-tnpanu`` que a grosso modo, esto, nos arrojaría las conexiones establecidas con pseudohosts en la red.

Es también un tanto notorio trastear con el directorio ``/proc`` en distros. Linux sobre todo en las rutas ``/proc/net/fib_trie`` y ``/proc/net/route`` , lo que nos devuelve estos dos ficheros es la configuración de interfaces disponibles en la misma máquina y su enrutación (ojo, esto último nos lo reporta en hexa, por lo que debemos de convertirlo para visualizar la información en su plena disponibilidad).

Por último, siempre recomiendo ser ingenioso y aprovecharnos del archivo ``/dev/tcp`` para hacer un port discovery y ``ping`` (si lo tiene instalado la misma máquina) para realizar un host discovery, de manera que con un sencillo script en bash podemos realizar este tipo de escaneo:

Port Discovery:
```shell
for puerto in $(seq 1 65535); do (timeout 1 bash -c "echo "" > /dev/tcp/IP_maquina/$puerto") && echo "$puerto OPEN" || echo "$puerto CLOSED"; done
```
{: .nolineno }

Host Discovery:
```shell
for host in $(seq 1 254); do (timeout 1 bash -c "ping -c 1 IP_maquina.$host") &>/dev/null && echo "IP_maquina.$host encontrado."; done
```
{: .nolineno }

*Podemos aplicar subprocesos en bash haciendo uso de ``&`` y ``wait`` en el mismo bucle, pero a mi gusto prefiero ser precavido y prescindir de esto.*

![img-description](/images/200iq.jpeg)
_Tu al leer todo este textardo_

## Herramientas

Después de tener una gran enumeración y una amplia información acerca de la topologia de la red víctima, podemos realizar el pivoting para compartirnos aquellos recursos que no podemos ver desde nuestra máquina. Algunas herramientas de interés, que en un futuro no tan lejano, iré explicando su funcionamiento en el blog, son:

- <a href="https://github.com/jpillora/chisel">Chisel</a>
- <a href="https://github.com/erluko/socat">Socat</a>
- <a href="https://github.com/haad/proxychains">Proxychains</a>
- SSH

Cabe destacar, que, tan solo he destacado las más famosas y por lo tanto habrán cientos de herramientas que puedan emular su funcionamiento e incluso mejorarlo.

![img-description](/images/yeah.jpeg)
_¿Hace falta algo más?_

## Conclusión

El pivoting es una técnica que beneficia tanto a nuestra red como a redes externas para realizar auditorias de pentesting. Como no, siempre se puede utilizar este mismo recurso para emularlo en tu propia red y hacer los diferentes tipos de port forwarding, según aplique (ya sea dynamic, remote o local) para 'transportarte' un recurso de tu otro ordenador al tuyo.

Desde luego, es una metodología que hay que investigar e ir con ojo, sobre todo contrarrestarla con aislamientos de red, reglas de firewall como ``iptables``, configuraciones de VPNs enfocadas a empresas y/u otras técnicas que eviten que ciertos atacantes puedan llegar a los recursos más valiosos de la empresa o incluso los nuestros.

-- Huesos

