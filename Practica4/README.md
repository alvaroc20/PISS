# Apartado 1: ENTORNO DE TRABAJO  
## 1.1 Trabajar en tres máquinas (se recomiendan virtuales): router, servidor y cliente. 
  Utilizaremos tan solo un PC, por lo que lo usaremos como router de forma nativa, y 2 maquinas virtuales, una que actuará como el servidor y otra como 
 cliente.
## 1.2 Configurar dos interfaces de red en el pc que actúe como router, una interfaz para el cliente y otra para el servidor en subredes distintas.  
  Se crean las siguientes subredes:  
```
    RED           Direccion         Direccion de los hosts            Mascara  
   Servidor      192.168.2.0     192.168.2.1 - 192.168.2.126      255.255.255.128  
   Cliente       192.168.2.128   192.168.2.129 - 192.168.2.254    255.255.255.128
```

## 1.3 Configurar tablas de enrutamiento y habilitar ip_forwarding en el router.  

  Para la configuración de las tablas de enrutamiento utilizaremos la función de ifconfig. Primero se configurarán las interfaces de red. Después de configurar las interfaces de red, se añaden las rutas. Lo haremos mediante los siguientes comandos:
  ```
  sudo ifconfig interfazRed:Nombre ipElegida netmask mascaraElegida broadcast broadcastElegido    
  sudo route add -net ipElegida netmask mascaraElegida dev nombreInterfazElegido
```  

  Para habilitar el **ip_forwarding** accederemos, dependiendo del protocolo IPv4 o IPv6, a los archivos del directorio **proc.** Tendremos que cambiar los valores del **ip_forward** a 1 para habilitarlo y 0 para desactivarlo, siendo 0 por defecto.    

# Apartado 2: MARCADO DE PAQUETES    
## 2.1 Marcar tráfico en RTP y SIP según corresponda a valores correctos de DSCP mediante la tabla mangle de iptables.   
  Configuración del Cliente:  

   Lo primero que haremos será crear la interfaz de red   
  `sudo ifconfig eth0:Client 192.168.2.130 netmask 255.255.255.128 broadcast 192.168.2.25`  
  
   Lo siguiente, sería crear una ruta para la conexión con el router.  
  `sudo route add -net 192.168.2.128/25 gw 192.168.2.129 dev eth0:Client`  

## 2.2 DSCP y prioridad RTP  
   A continuación, marcaremos el tráfico de la interfaz del cliente en RTP y SIP mediante iptables. El valor utilizado para el **dscp ** es el valor 46, haciendo así referencia, al estándar RFC2597, Expedited Forwarding, que nos asegura la entrega y que no se descarte ningún paquete.  
   La configuración SIP UDP  en el puerto 5060  
   ```
   sudo iptables -t mangle -A OUTPUT -p udp --dport 5060 -j DSCP --set-dscp 46  
   sudo iptables -A OUTPUT -p udp --dport 5060 -j ACCEPT
```  

   El flujo de datos en RTP, relacionado con el rango de puertos de Asterisk en nuestro caso 10000:20000  
   ```
   sudo iptables -t mangle -A OUTPUT -p udp --dport 10000:20000 -j DSCP --set-dscp 46  
   sudo iptables -A OUTPUT -p udp --dport 10000:20000 -j ACCEPT
```  

  Configuración del Servidor:  
   ```
   sudo ifconfig enp0s3:Servidor 192.168.2.2 netmask 255.255.255.128 broadcast 192.168.2.127  
   sudo route add -net 192.168.2.128/25 gw 192.168.2.1 dev eth0:Client
```  
   La configuración del SIP-UDP y RTP es igual que en el caso del cliente.  
   
   Para finalizar este último paso, se configurará el router, en nuestro caso, nuestro PC. Debemos modificar el ip_forward, en caso de estar activo, no modificarlo.  
   Crear 2 interfaces de red, servidor y cliente, y terminar con las rutas entre las subredes.  
   ```
sudo ifconfig wlp2s0:Cliente 192.168.2.129 netmask 255.255.255.128 broadcast 192.168.2.255  
sudo ifconfig wlp2s0:Servidor 192.168.2.1 netmask 255.255.255.128 broadcast 192.168.2.127
sudo route add -net  192.168.2.0/25 gw  192.168.2.129 dev wlp2s0:Cliente   
sudo route add -net  192.168.2.128/25 gw  192.168.2.1 dev wlp2s0:Servidor
```  

## 2.3 Parámetros de iptables utilizados.  
Los parámetros utilizados en iptables han sido los siguientes:  
**-t** para utilizar las tablas MANGLE.  
**-A** para indicar el sentido del flujo de datos OUTPUT.  
**-p** para indicar el protocolo udp.  
**--dport** indicar el puerto.  
**-j** para indicar la acción en nuestros caso ACCEPT.  
**--set-dscp** para indicar que estándar utilizar en dscp.   
   
# Apartado 3: ANÁLISIS DEL RENDIMIENTO  
## 3.1 Saturar la conexión utilizando iperf  

Desde el cliente ejecutamos:  
    `sudo iperf -c 192.168.0.2 -P 1 -p 1000 -f M -t 10`  

Desde el servidor ejecutamos:  
    `sudo iperf -s -P 0 -i 1 -p 1000 -f M`  

## 3.2 Generar tráfico SIP   
Mediante reiteradas llamadas generamos tráfico SIP. Como se puede comprobar en la imagen, el tráfico SIP utiliza Expedited Forwarding.  
El tráfico RTP contiene Expedited Forwarding también.  


![](https://github.com/alvaroc20/PISS/blob/main/Practica4/p4.jpeg)  


  
     
    

