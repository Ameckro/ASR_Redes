# ASR_Redes

## 7.1.1. Servicios de almacenamiento en red: NFS
(https://raw.githubusercontent.com/Ameckro/ASR_Redes/873dafe62c07e19bd259827762b154ee21832c6c/nfs.png)


### Introducción

   Sistema de ficheros de red
   
   Los ficheros remotos se acceden como los locales (comandos y aplicaciones)
   
   Independencia de ubicación: el usuario no distingue los locales de los remotos. Administrador sí
   
   Requiere un pre-montado
    
   Configuración:
        - Cliente: ```/etc/fstab```
        - Servidor: ```/etc/exports```

### Servidor

   - Software necesario: paquetes nfs-kernel-server, nfs-common y rpcbind (portmap)[2]. Antes de instalar NFS, es recomendable asegurarse de que el servicio rpcbind está iniciado.
  
  ``` [2]	 rpcbind (portmap) proporciona un servicio que permite mapear la númeración RPC a los puertos TCP o UDP de los servicios asociados.```

   Creado por Sun. Basado en RPC: procesos rpc.nfsd y rpc.mountd.

   Configuración de los directorios exportados: fichero ```/etc/exports```. Una línea por cada directorio a exportar:
        - Directorio a exportar
        -  Máquina o subred con permisos para importar (carácter * para todos).
        - Opciones entre paréntesis: solo lectura, sincronía, ... (```man exports```)

   Ejemplo[3]:
```
    /home-global 192.168.248.0/24(sync)  # '/24' = 255.255.255.0
    /media/dvd  *(ro)                    # todos los nodos en modo solo-lectura. Exporta un CD rom
```


```[3]  Para poder compartir directorios (en el ejemplo /home-global y /media/dvd), éstos deberán ser accesibles en el sistema de ficheros local. Es decir, (1) crearlos en caso de que no existan previamente y (2) con permisos de acceso.```
Exportar directorios según la configuración: comando exportfs:
```
    $ exportfs -rv  # aplicar los cambios en la configuración
    $ exportfs -v   # mostrar las unidades exportadas
    $ man exportfs  # mas información
```

### Cliente

   Software necesario: paquetes nfs-common y rpcbind (portmap)

   Fichero /etc/fstab: una línea por cada unidad remota a importar (en fstab, con cada reboot, se monta el dir) 
    
   Dirección IP del servidor + : + directorio exportado en el servidor. Para ver los directorios exportados por un servidor:

   ``` $ showmount -e nfsservername```

   Punto de montaje local. El directorio deberá estar creado (y vacío) y con los permisos adecuados.

   Tipo del sistema de ficheros a montar: nfs

   Modificadores: ```timeo=n``` (décimas de segundo de retardo permitido antes de mensaje de error), ```intr``` interrumpir     si no hay respuesta y es tipo hard, soft acceso no persistente, ```noauto``` para carga no automática, ...
        
        
### Ejemplo:

   Configuración en /etc/fstab:

   ```192.168.248.111:/home-shared /home/remote-projects nfs  timeo=15,intr```

   Montar directamente:

   ```$ mount -t nfs 192.168.248.111:/home-shared /home/remote-projects -o timeo=15,intr```

### Seguridad

   Si no se comparten las cuentas (p.e. con NIS), puede causar problemas si entre los sistemas los identificadores de usuarios y grupos coinciden:
      La solución puede ser:
         - Usar NIS
         - Una consistencia con los UID (En principio el UID 1001 esta es posible que esté en las dos cuentas, por lo que el administrador deberia ir maquina a maquina evitando esos UID) => solución mala.
  
   Además, si desde el cliente se crea/modifica un fichero como root, en el servidor tambien se accederia como root. Para evitar eso si el cliente se usa como root, en el servidor seria cambiado por el usuario "nobody"
    
   Seguridad: la característica root_squash se pone por defecto para que el superusuario de la máquina cliente no pueda tener privilegios especiales. no_root_squash quita esta protección.
    
   Restringir acceso mediante /etc/hosts.allow (pormapper 192.168.216.0/24) y /etc/hosts.deny(pormapper ALL) .
   
   Otra opción 

```
user1@oria:/puntoB$ sudo showmount -e 192.168.216.134
Export list for 192.168.216.134:
/usr/local *
/prueba    192.168.0.0/16
user1@oria:/puntoB$ sudo mount -t nfs 192.168.216.134:/usr/local /puntoA/
user1@oria:/puntoB$ sudo mount -t nfs 192.168.216.134:/prueba /puntoB/
```

## 7.1.2. Servicio de configuración compartida: NIS

NIS (Network Information Service) permite compartir información de control entre un servidor y sus clientes. Conocido inicialmente como yellow pages, razón por la que algunos comandos comienzan con yp

../_images/domain-nis.png

### Introducción

   Función: compartir información de control en red local
   Principalmente usuarios y grupos
   Creado por Sun . Se basa en RPC
   A sustituir por LDAP
   Dominio NIS: área controlada por el servicio NIS.
       Gestión mediante los comandos domainname o nisdomainname.
   Estructura básica: nombre de dominio y broadcast
       También es posible restringir el acceso
   Configuración
       Servidor (ypserv): /etc/ypserv.conf
       Cliente (ypbind):
           Configuración cliente: /etc/yp.conf
           Configuración respecto a otros servicios: /etc/nsswitch.conf
   Comandos básicos: ypserv, ypbind e ypcat.

Puntos a destacar:

    Información que se distribuye: usuarios, grupos, hosts, servicios, .... Denominados mapas. Se obtienen a partir de los ficheros originales, convirtiéndolos a un formato de base de datos específico de la aplicación.

    El servidor y los clientes se agrupan en un concepto denominado dominio[1]. Un servidor da servicio a los clientes de su dominio.
    [1]	

    Dicho dominio puede coincidir con el definido para el servicio DNS, pero conceptualmente son distintos.

    Servidor: configurado para distribuir los mapas dentro de su dominio. Se puede configurar qué mapas se distribuyen, además de a qué clientes. Como medida de seguridad, se puede configurar servidores adicionales (que se denominarán esclavos, para apoyar al principal o maestro)

    Cliente: el cliente solicita los mapas del dominio al que pertenece. En el propio cliente se decide cómo se combina la información obtenida con la configuración del sistema local (ejemplo: usuarios obtenidos del dominio NIS con los usuarios locales).

Existe una versión más reciente denominada NIS+ con mejoras significativas (añade mayor seguridad y estructura jerárquica), sin embargo, no se encuentra muy extendida.
Servidor

Software necesario: paquetes nis y rpcbind (portmap)

Es recomendable tener un maestro y varios esclavos (al menos uno). Si el maestro falla, uno de los esclavos tomará el papel de maestro temporal.
Pasos a seguir para configurar el servidor maestro

    [Comprobar que el servicio rpcbind está en marcha] Instalar el paquete nis. Al instalar solicitará un dominio (paso 2).

    Establecer el dominio[2]. Opciones:

        Instanciarlo al instalar el paquete nis (o dpkg-reconfigure nis)

        Modificando el fichero /etc/defaultdomain (puede ser necesario reiniciar el sistema):

        $ echo "lab_domain" > /etc/defaultdomain

        Mediante el comando domainname (o nisdomainname o ypdomainname) (se perderán los cambios al reiniciar el sistema o el servicio nis).
    [2]	

    Se distingue entre mayúsculas y minúsculas.

    Tras establecer el dominio, indicar en el fichero /etc/hosts las direcciones IP y nombre de los servidores de NIS que serán master o esclavo en el dominio[3]. Éste es el único modo de resolver los nombres de los servidores, ya que NIS no utiliza DNS:

    127.0.1.1         odin.labo.sa          odin    # conexion con si mismo
    # 192.168.248.112 serv_respaldo.labo serv_respaldo

    [3]	

    Sustituir por el dominio por defecto (localdomain).

    Configurar el paquete NIS como servidor. En el fichero /etc/default/nis:

    # Are we a NIS server and if so what kind (values: false, slave, master)?
    NISSERVER=master
    # Are we a NIS client?
    NISCLIENT=false

    Tras realizar el cambio, reiniciar el servicio NIS para que aplique la modificación. Al reiniciar el servicio, los procesos asociados al servidor (ypserver entre otros) se pondrán en marcha.

    Advertencia

    Si la información requerida para exportar por el servicio aún no está disponible, el proceso servidor puede quedar bloqueado. Si esto ocurre, detenerlo mediante Ctrl+C.

    Generar los mapas que contendrán la información a exportar:

    $ /usr/lib/yp/ypinit -m    # - m =>master / - s => slaves
    At this point, we have to construct a list of the hosts which will run NIS
    servers.  odin.labo.sa is in the list of NIS server hosts.  Please continue to add
    the names for the other hosts, one per line.  When you are done with the
    list, type a <control D>.
      next host to add:  odin.labo.sa
      next host to add:
    The current list of NIS servers looks like this:

    odin.labo.sa

    Is this correct?  [y/n: y]  y
    We need a few minutes to build the databases...
    Building /var/yp/labo/ypservers...
    Running /var/yp/Makefile...
    make[1]: Entering directory `/var/yp/lab_domain'
    Updating passwd.byname...
    Updating passwd.byuid...
    Updating group.byname...
    Updating group.bygid...
    Updating hosts.byname...
    Updating hosts.byaddr...
    Updating rpc.byname...
    Updating rpc.bynumber...
    Updating services.byname...
    Updating services.byservicename...
    Updating netid.byname...
    Updating protocols.bynumber...
    Updating protocols.byname...
    Updating netgroup...
    Updating netgroup.byhost...
    Updating netgroup.byuser...
    Updating shadow.byname...
    make[1]: Leaving directory `/var/yp/lab_domain'

    odin.labo.sa has been set up as a NIS master server.

    Now you can run ypinit -s odin.labo.sa on all slave server.

    Nota

    El fichero /etc/networks debe existir previamente, de lo contrario habrá que crearlo (touch /etc/networks).

    Nota

    Los mapas se crean en base a la configuración en el fichero /var/yp/Makefile, en el que se indican los ficheros de configuración a exportar, junto con los valores, como por ejemplo, qué subconjunto de usuarios se van a exportar.

    Tras realizar el cambio, reiniciar el servicio NIS para que aplique la modificación. Al comenzar el servicio, se pondrán en marcha los procesos asociados al servicio, entre otros, ypserver:

    $ systemctl restart nis
    [ ok ] Restarting NIS (via systemctl): nis.service

    $ /etc/init.d/nis restart
    [ ok ] Stopping NIS services: ypbind ypserv ypppasswdd ypxfrd.
    [ ok ] Starting NIS services: ypserv yppasswdd ypxfrd ypbind.

    Probar la nueva configuración:

    $ ypcat passwd    # passwd baliabideko balioak ikusteko
    user3:x:1002:1002:,,,:/home/user3:/bin/bash
    user2:x:1001:1001:,,,:/home/user2:/bin/bash
    user1:x:1000:1000:,,,:/home/user1:/bin/bash

Cliente

Software necesario: paquetes nis y rpcbind (portmap). El servicio nis-common deberá estar en marcha.
Pasos:

    Instalar el paquete nis

    Configurar el dominio del mismo modo que en el caso del servidor

    Configurar el paquete NIS como cliente. En el fichero /etc/default/nis:

    # Are we a NIS server and if so what kind (values: false, slave, master)?
    NISSERVER=false
    # Are we a NIS client?
    NISCLIENT=true

    Reiniciar el servicio NIS para aplicar la modificación. Al reiniciar el servicio, el proceso ypbind se pondrá en marcha.

    Nota

    La configuración anterior —sólo se indica el dominio, pero no el servidor de ese dominio— proporciona flexibilidad y transparencia con respecto a la ubicación del servidor (la localización de dicho servicio se realiza mediante broadcast). Sin embargo, si deseamos indicar explicitamente en qué servidor se encuentra el servicio, podemos incluir la siguiente línea en el archivo /etc/yp.conf:

    domain lab_domain server 192.168.248.111
    # ypserver 192.168.248.111 # otra opción

    Comprobación importación:

    $ ypwhich              # servidor master
    $ yptest               # testea los mapas importados
    $ ypwhich -x           # mapas importados
    $ ypcat passwd         # valores importados en el mapa passwd
    $ yppoll passwd.byname # hora de la ultima actualización y versión del mapa

    Nota

    Observese que, a pesar de haber importado correctamente los mapas, el sistema todavía no los ha integrado con los suyos (es decir, todavía no los utiliza):

    $getent passwd  # valores de la bd local de passwd

    El comando getent muestra los valores de las bases de datos que utiliza el sistema: passwd, groups, hosts ....

    Para comprobar que los datos se importan pero todavía no se utilizan:

        Crea en el servidor un usuario con el identificador nisuser (adduser):

        $ adduser nisuser

        En el cliente, comprueba que el usuario se importa, pero que todavía no se utiliza:

        $ ypcat passwd | grep nisuser  # usuario importado
        nisuser:x:1002:1002::/home/nisuser:/bin/bash
        $ getent passwd | grep nisuser # no está disponible

    Integración de los datos importados: una vez que el sistema ha importado la información, es necesario indicar cómo se integrará con las bases de datos locales. Dicha configuración se establece en el fichero en /etc/nsswitch.conf:

    passwd:     compat
    group:      compat
    shadow:     compat

    hosts:      files dns
    networks:   files

    protocols:  db files
    services:   db files
    ethers:     db files
    rpc:        db files

    netgroup:   nis

    Como se puede observar, hay dos opciones para combinar la información local con la importada:

        Indicar la(s) fuente(s) y el orden de búsqueda: files para los ficheros locales, nis para los ficheros obtenidos mediante nis, ldap, .... Aparecen por orden de prioridad. Ejemplo:

        passwd:    files nis

        Tras realizar la modificación, comprobar ( ypcat y getent) que el sistema incluye ambos conjuntos de valores en su entorno de trabajo.

        Modo compat. Para que el sistema trabaje con los valores importados, estos deben ser añadidos explicitamente en la base de datos del sistema. Para utilizar este modo, deberemos establecer la siguiente configuración en el fichero /etc/nsswitch.conf:

        passwd:    compat

        Para indicar que queremos añadir todos los usuarios importados a la base de datos passwd, en el fichero /etc/passwd añadimos al final:

        +::::::    # Añadir todos los usuarios importados a traves de NIS
        Por ejemplo:
        $ echo "+:::::: # Usuarios importados por NIS" >> /etc/passwd

        Asimismo, hacemos los mismo para los ficheros /etc/group y /etc/shadow:

        $ echo "+::: # Usuarios importados por NIS" >> /etc/group
        $ echo "+:::::::: # Usuarios importados por NIS" >> /etc/shadow

        Nota

        Observese que el número de caracteres : es el mismo que el de cualquier otra línea del fichero correspondiente.

        Ahora podemos comprobar (getent passwd) que los valores importados han sido añadidos.

        Mediante este método se puede discriminar por usuarios o grupos. Ejemplo: en lugar de la línea anterior (añadir todos los usuarios, añadimos las siguientes líneas):

        +domuser2::::::    # permitimos el acceso a domuser2
        -domuser1::::::    # denegamos el acceso a domuser1
        +::::::/bin/false  # cuentas válidas, pero sin poder
                           # iniciar shell

Seguridad

    Gestión de contraseñas. Si se desea cambiar la contraseña de un usuario desde uno de los sistemas que ha importado dicho usuario, no será suficiente cambiar la contraseña en local (comando passwd), sino que dicha contraseña deberá ser cambiada en el servidor master. Para ello se deberá utilizar el comando yppasswd proporcionado por NIS.

    Se puede restringir el acceso al servicio NIS mediante la configuración del fichero /etc/ypserv.securenets en los servidores maestros y esclavos[4]:

    255.0.0.0        127.0.0.0         # acceso para localhost
    255.255.255.0    192.168.248.0    # acceso desde red local

    [4]	

    Adicionalmente, también se puede utilizar tcpwrappers y/o netfiltering

    También se puede reforzar la seguridad mediante la configuración del fichero /etc/ypserv.conf

    Nota

    NIS no proporciona un servicio de autenticación, sino un servicio de replicación de información de control. Si bien dicha replicación se realiza de manera activa, en ocasiones las modificaciones en el servidor maestro no se propagan (los mapas no se modifican y, por lo tanto, no son distribuidos). Para estas situaciones particulares, se puede forzar la actualización de este modo (en el maestro):

    $ cd /var/yp
    $ make

    Otra opción, aunque más ineficiente, podría ser volver a generar todos los mapas mediante el comando ypinit anteriormente mencionado

Práctica: NFS + NIS

A continuación un ejemplo de cómo ambos servicios se complementan. En una red de varios puestos de tipo Unix / Linux va a trabajar un conjunto de usuarios. Se desea tener la posibilidad de que un usuario pueda acceder a sus archivos (y configuración del sistema) independientemente del puesto de trabajo desde el que acceda.

Para implementar este sistema se puede recurrir a la siguiente configuración:

    Servicio de albergue de las carpetas home en una ubicación global. Solución: compartir una carpeta de nombre /home_global por medio de NFS
    Utilizar un sistema de gestión de usuarios en red: NIS.

A fin de probar el sistema, crear las siguientes cuentas en el sistema que alberga el servicio NIS:

$ adduser --home /home-global/domuser1 --uid 1111 domuser1
$ adduser --home /home-global/domuser2 --uid 1112 domuser2
$ adduser --home /home-global/domuser3 --uid 1113 domuser3

Comprobar el funcionamiento del sistema iniciando sesión en un sistema cliente con uno de los usuarios anteriores. Al iniciar sesión el usuario siempre accede a su entorno de trabajo (su home, incluyendo archivos de inicialización, correo, ...), independientemente del puesto desde el que está accediendo. Asegurarnos que los permisos son correctos y que no se producen accesos indebidos.

...

