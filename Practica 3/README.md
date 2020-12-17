# Apartado 1: Definición de conceptos básicos  
**Centralita**: Con una centralita Asterisk se puede conectar al mundo exterior a través de Voz IP o las tecnologías tradicionalesde telefonía. Además se puede utilizar virtualmente cualquier teléfono IP basado en estándares: Asterisk incluyecontroladores para SIP y otros protocolos.    
**Diaplan**: El DialPlan o Plan de Marcación es un grupo de reglas que le indican a la central PBX-IP que hacer o como manejarlos números marcados por un usuario. El dialplan hace la función de una tabla de ruteo de llamadas, cada numeroque se marca, lee la información del dialplan y después se decide hacia donde se dirigen; estos números puedeningresar o salir del sistema.  
**Contexto**: Es una sección del dialplan, puede estar directamente en el archivo /etc/asterisk/extensions.conf o en un archivoincluido en el anterior.  


# Apartado 2: Primeros pasos con Asterisk  
Lo primero que debemos hacer al instalar Asterisk es actualizar el sistema de Ubuntu.  
```
sudo apt update 
sudo apt -y upgrade
```

Ahora debemos instalar todas las dependencias necesarias.
```
sudo add-apt-repository universe
sudo apt -y install git curl wget libnewt-dev libssl-dev libncurses5-dev subversion  libsqlite3-dev build-essential libjansson-dev libxml2-dev  uuid-dev
```  
Una vez llegados a este punto, vamos a descargar Asterisk y posteriormente a configurarlo.  
Para descargarlo, iremos a la carpeta donde queramos ubicarlo, nosotros lo haremos en la siguiente carpeta: /usr/src/.  
`cd /usr/src/`  

Lo descargamos mendiante wget y descomprimimos el archivo que descargamos.  
```
wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-17-current.tar.gz
tar xvf asterisk-17-current.tar.gz
```  


Asegurarse de que todas las dependencias se resuelvan:  
`sudo contrib/scripts/install_prereq install`  

Una vez lo tengamos, aparecerá un mensaje: install completed successfully, por lo que ahora vamos a configurarlo.  
`sudo ./configure`  

Obtendremos algo como vemos en la siguiente imagen:  
![](https://github.com/alvaroc20/PISS/blob/main/Practica%203/foto1.jpeg)   

Para acabar, debemos realizar:  
`sudo make menuselect`  

Aquí debemos configurar lo que necesitemos, en nuestro caso añadimos un par de add-ons aunque para la práctica seguramente no sean necesarios, y pulsaremos F12 para salir y guardar los cambios.  
![](https://github.com/alvaroc20/PISS/blob/main/Practica%203/foto2.jpeg)  

Para montarlo e instalarlo haremos lo siguiente:  
```
sudo make  
sudo make install
```
Para finalizar, debemos instalar las configuraciones y ejemplos.  
```
sudo make samples
sudo make config
sudo ldconfig
```
Con esto finalizaría la instalación de Asterisk.  

Para empezar a usarlo, creamos por separado un usuario y un grupo con los permisos correctos.  
```
sudo groupadd asterisk  
sudo useradd -r -d /var/lib/asterisk -g asterisk asterisk  
sudo usermod -aG audio,dialout asterisk  
sudo chown -R asterisk.asterisk /etc/asterisk  
sudo chown -R asterisk.asterisk /var/{lib,log,spool}/asterisk  
sudo chown -R asterisk.asterisk /usr/lib/asterisk  
```  

# Apartado 3: Crear centralita con dos extensiones SIP en contexto propio.  
Lo primero que debemos hacer, es modificar el archivo sip.conf, para guardar la configuración que nosotros queremos tener. Vamos a nombrar 2 teléfonos, Álvaro y Sergio, y después restablecemos el servicio de Asterisk. Trabajaremos como superusuario.  

```
gedit extensions.conf  
gedit sip.conf  
gedit pjsip.conf  
service asterisk restart  
```  
Cada uno de los usuarios, tendrá una contraseña que almacenamos en ese archivo, para nosotros, la contraseña será la misma que el nombre.  

El siguiente paso es configurar los propios teléfonos para comunicarse con Asterisk. De la forma en que hemos configurado las cuentas en el controlador del canal SIP, Asterisk esperará que los teléfonos se registren en él. El registro es simplemente un mecanismo donde un teléfono se comunica "Hey, soy el teléfono de Álvaro... aquí está mi nombre de usuario y contraseña. Oh, y si recibes alguna llamada para mí, estoy en esta dirección IP en particular."  

### extensions.conf   
[from-internal]  
exten=>6001,1,Dial(PJSIP/alvaro,20)   
exten=>6002,1,Dial(PJSIP/sergio,20)    

### sip.conf   
[general]  
transport=udp  

[friends_internal](!)  
type=friend  
host=dynamic  
context=from-internal  
disallow=all  
allow=ulaw  

[alvaro](friends_internal)  
secret=alvaro ; put a strong, unique password here instead  

[sergio](friends_internal)  
secret=sergio ; put a strong, unique password here instead   

### pjsip.conf    
[transport-udp]  
type=transport  
protocol=udp  
bind=0.0.0.0  

;Templates for the necessary config sections

[endpoint_internal](!)  
type=endpoint  
context=from-internal  
disallow=all  
allow=ulaw  

[auth_userpass](!)  
type=auth  
auth_type=userpass  

[aor_dynamic](!)  
type=aor  
max_contacts=1  

;Definitions for our phones, using the templates above

[alvaro](endpoint_internal)  
auth=alvaro  
aors=alvaro  
[alvaro](auth_userpass)  
password=alvaro ; put a strong, unique password here instead  
username=alvaro  
[alvaro](aor_dynamic)   

[sergio](endpoint_internal)  
auth=sergio  
aors=sergio  
[sergio](auth_userpass)  
password=sergio ; put a strong, unique password here instead  
username=sergio  
[sergio](aor_dynamic)     


Una vez lo tengamos, ya podemos conectarnos desde un móvil. Para comprobar si lo hemos creado bien probaremos los siguientes comandos desde el servidor.   
```
sudo asterisk -vvvr
dialplan show from-internal
module load chan_sip.so
sip show peers
```
![sip show peers](https://github.com/alvaroc20/PISS/blob/main/Practica%203/foto3.jpeg)  
![diaplan show from-internal](https://github.com/alvaroc20/PISS/blob/main/Practica%203/foto4.jpeg)  

Realizamos la llamada para comprobar que funciona.  
![](https://github.com/alvaroc20/PISS/blob/main/Practica%203/llamada_normal_alvaro.jpeg)  
![](https://github.com/alvaroc20/PISS/blob/main/Practica%203/llamada_normal.jpeg)  

# Apartado 4: Comunicación entre dos centralitas  
Para realizar este paso, necesitaremos un ordenador más, y tan solo modificaremos de la siguiente manera los archivos sip.conf y extensions.conf para que se enlace mediante un enlace trunk. Cada móvil estará registrado en una centralita distinta, por lo que para llamar de uno a otro, hace falta pasar por la otra centralita.

### sip.conf  

[general]   
transport=udp  

[default]  
transport=udp  
context=default  
allowguest=no    
qualify=yes  

[Trunk_Asterisk_A]  
type=peer  
host=192.168.100.126  
disallow=all  
allow=alaw  
context=Entrantes_Trunk_Asterisk_A  

[7001]  
type=friend  
secret=7001;  
disallow=all  
allow=alaw  
context=usuarios  
host=dynamic  

[7002]  
type=friend  
secret=7002;  
disallow=all  
allow=alaw  
context=usuarios  
host=dynamic  


### extensions.conf  
[usuarios]    

; LLAMADAS INTERNAS  
exten=>_700X,1,Dial(SIP/${EXTEN})  
same=> n,Hangup()  

; SALIENTES HACIA ASTERISK_A  
exten=>_600X,1,Answer  
same=> n, Wait(1)  
same=> n, Dial(SIP/Trunk_Asterisk_A/${EXTEN})  
same=> n, Hangup()  

; ENTRANTES DESDE ASTERISK_A   
[Entrantes_Trunk_Asterisk_A]  
exten=> 7001,1,Dial(SIP/7001)  
same=> n, Hangup()  

exten=> 7002,1,Dial(SIP/7002)  
same=> n, Hangup()  

Con esto ya podríamos realizar la llamada de forma satisfactoria.  
![](https://github.com/alvaroc20/PISS/blob/main/Practica%203/llamada_trunk_alvaro.jpeg)   
![](https://github.com/alvaroc20/PISS/blob/main/Practica%203/llamada_trunk_coletas.jpeg)  
![](https://github.com/alvaroc20/PISS/blob/main/Practica%203/llamada_trunk.jpeg)
