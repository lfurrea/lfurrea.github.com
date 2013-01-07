---
layout: post
title: Analisis de Desempeno
---

{{ page.title }}

### Objetivo
El presente documento tiene como objetivo proveer una revisión del
estado actual de la infraestructura de la red de voz en las
instalaciones de Avenida Escazú para que sea utilizado como linea base
en las tareas de resolución de problemas de la red y
la planeación para mejoramiento y crecimiento.

La revisión está basada en una inspección física de los
cuartos de distribución de cableado y una descripción de la topología
lógica de la red.

De esta manera debe satisfacer el propósito de establecer la capacidad
actual de la red de voz y las áreas vulnerables en temas de seguridad,
desempeño y disponibilidad.

### Áreas de Evaluación

####Sitios y Dispositivos de Red
Cuatro edificios cada uno con su cuarto de cableado secundario
convergen en un cuarto de cableado principal en donde están instalados
los dispositivos de columna.

Cada cuarto secundario dispone de un único switch de distribución y se conecta a un switch
de columna en el cuarto principal a través de un enlace de fibra óptica.

No se efectuó revisión de la configuración de ningún switch.

Cada cuarto secundario almacena dispositivos ATA de la línea Cisco
Small Business y/o Gateways análogos FXS de la marca Grandstream en
densidad máxima de 8 puertos, que conectan con terminales telefónicas
análogas en oficinas y locales del edificio.

Uno de los edificios tiene instalada una central telefónica marca
Panasonic modelo KX-TDA-100D que majena un enlace ISDN en formato E1
y dispone de 300 DIDs. Este edificio conecta un multipar analógico de
24 pares hacia el cuarto de cableado principal.

No se efectuó revisión de la configuración de esta central telefónica.

####Enlaces
Dos enlaces en formato Cable Modem terminan en un router de perímetro
de la línea Cisco Enterprise modelo 2811.

No se efectuó revisión de la configuración de este router de perímetro.

####Servidores

Un solo servidor aislado marca HP Proliant aloja el software de central telefónica basado
en el software de código abierto Asterisk, instalado en GNU/Linux
Centos 5.7.

La totalidad del volumen de revisión y análisis inicial fue dedicado a este
servidor que representa la unidad de funcionalidad principal de la red
de telefonía IP. Se evaluaron aspectos de configuración, desempeño,
capacidad y demanda, seguridad y disponibilidad de los servicios.  

####Terminales
La mayor parte de terminales son teléfonos análogos provisionados a
través de ATAs o gateways analógicos con puertos FXS.

Hay algunos teléfonos IP en la red pero no se efectuó ninguna revisión
de su configuración.

####Aplicaciones
Las aplicaciones de voz se limitan a la funcionalidad provista por las
herramientas del software Elastix versión 2.3 que se encuentra
instalado en el servidor que aloja Asterisk.

###Ánálisis de Capacidad y Demanda

###Análisis de Desempeño

Se reportan múltiples incidentes de interrupción de los servicios de
llamadas salientes y entrantes en las terminales telefónicas.

Se habilita la recolección de datos de carga de procesamiento mediante
el promedio provisto por el sistema en /proc/loadavg en el
servidor y se analiza un histórico reciente pero que contiene
igualmente múltiples incidentes de la misma naturaleza que los
reportados en el pasado.

<img src="http://dl.dropbox.com/u/49541944/graph/jan4-load.png">

En el gráfico se observa una carga completamente normal que tiende
mayormente a niveles bajos debido a que las tareas de procesamiento no
son intensas en uso de procesador o IO de disco. La carga máxima de este servidor se
da en IO en la interfaz de red y el stack de memoria que maneja las
listas enlazadas para los canales en el software Asterisk. No se puede
corelacionar la carga de procesamiento CPU con las causas de los fallos e incidentes. 

Al revisar el estado de la interfaz de red del servidor vemos que no
sufre de ningún tipo de acumulación de errores y la cantidad está muy
por debajo del 0.5% que es el índice de umbral recomendado.

[root@PORTAFOLIO-UC ~]# ethtool -S eth0
NIC statistics:
     rx_packets: 242203524
     tx_packets: 216673674
     rx_bytes: 41071977460
     tx_bytes: 45734172629
     rx_broadcast: 34548911
     tx_broadcast: 466575
     rx_multicast: 951302
     tx_multicast: 10
     rx_errors: 7
     tx_errors: 0
     tx_dropped: 0
     multicast: 951302
     collisions: 0
     rx_length_errors: 7
     rx_over_errors: 0
     rx_crc_errors: 0
     rx_frame_errors: 0
     rx_no_buffer_count: 0
     rx_missed_errors: 0
     tx_aborted_errors: 0
     tx_carrier_errors: 0
     tx_fifo_errors: 0
     tx_heartbeat_errors: 0
     tx_window_errors: 0
     tx_abort_late_coll: 0
     tx_deferred_ok: 0
     tx_single_coll_ok: 0
     tx_multi_coll_ok: 0
     tx_timeout_count: 0
     tx_restart_queue: 0
     rx_long_length_errors: 0
     rx_short_length_errors: 7
     rx_align_errors: 0
     tx_tcp_seg_good: 69228
     tx_tcp_seg_failed: 0
     rx_flow_control_xon: 0
     rx_flow_control_xoff: 0
     tx_flow_control_xon: 0
     tx_flow_control_xoff: 0
     rx_long_byte_count: 41071977460
     rx_csum_offload_good: 206039908
     rx_csum_offload_errors: 15
     rx_header_split: 0
     alloc_rx_buff_failed: 0
     tx_smbus: 260277
     rx_smbus: 33405312
     dropped_smbus: 0
     rx_dma_failed: 0
     tx_dma_failed: 0

Igualmente los contadores del stack de red en el SO muestran valores
dentro de los umbrales normales.

Tcp:
    200285 active connections openings
    274215 passive connection openings
    600 failed connection attempts
    1287 connection resets received
    3 connections established
    21647097 segments received
    21815462 segments send out
    6981 segments retransmited
    2 bad segments received.
    686 resets sent
Udp:
    203868971 packets received
    77704 packets to unknown port received.
    117848 packet receive errors
    213423784 packets sent 

No se pueden corelacionar las causas de los fallos a errores de
procesamiento de paquetes localmente en el servidor o el stack TCP/IP
del SO.

Al revisar detenidamente las bitácoras del servidor encontramos una
serie de eventos que tienen una gran posibilidad de estar relacionados
a los fallos reportados. 

Estos eventos se derivan de la opción de configuración en Asterisk
para las terminales SIP llamada "qualify", que básicamente implica que
cada terminal será monitoreada usado paquetes "keepalives" que
corresponden al mensaje SIP con el método OPTIONS que por defecto se
envía cada 2000ms, de esta manera si Asterisk obtiene el mensaje de
respuesta dentro del intervalo de 2000ms Asterisk mantiene un
estado para cada terminal y la marca como "con vida", si el mensaje de
respuesta al SIP OPTIONS no se obtiene dentro del intervalo de 2000ms
Asterisk cambia el estado de la terminal a "sin vida" y cualquier
intento de llamada hacia esa terminal mientras haya sido marcada "sin
vida" fallará.
 
En los gráficos siguientes, observamos los eventos en los cuales se
supera el intervalo de 2000ms graficados en dos tipos de terminales
SIP, el set en rojo corresponde a terminales en la red local ATAs o
teléfonos IP y el set en verde corresponde en su totalidad a
terminales SIP de registro con el proveedor Callmyway, que resulta ser
al menos una por cada local u oficina de Av Escazú suscrita al servicio.

<img src="http://dl.dropbox.com/u/49541944/graph/dec29.png">
<img src="http://dl.dropbox.com/u/49541944/graph/dec30.png">
<img src="http://dl.dropbox.com/u/49541944/graph/dec31.png">
<img src="http://dl.dropbox.com/u/49541944/graph/jan1.png">
<img src="http://dl.dropbox.com/u/49541944/graph/jan2.png">
<img src="http://dl.dropbox.com/u/49541944/graph/jan3.png">

Se observa por lo tanto que la densidad de estos eventos con el
proveedor Callmyway es significativa y se puede correlacionar a los
reportes de fallos e incidentes experimentados por los usuarios del
servicio. Particularmente esto explica la naturaleza azarosa de los
reportes, debido a que solamente si hubo un intento de llamada en el
instante particular en el cual Asterisk cambió el estado de la
terminal SIP a "sin vida" entonces es la causa de uno de los fallos.

Esto también explica la razón por la cual la solución temporal
provista por la empresa de soporte de la central Asterisk en la cual
reiniciaban el stack o módulo SIP del software Asterisk daba
resultados, ya que este estado transiente de "sin vida" o "con vida"
que mantiene Asterisk es obligado a reiniciarse y renovarse para todas
las terminales SIP.

###Análisis de Disponibilidad

###Análisis de Seguridad


###Conclusiones y Recomendaciones
