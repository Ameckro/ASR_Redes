# ASR_Redes

## 7.1.1. Servicios de almacenamiento en red: NFS
[https://github.com/Ameckro/ASR_Redes/blob/master/nfs.png]


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
   ``` [3]  Para poder compartir directorios (en el ejemplo /home-global y /media/dvd), éstos deberán ser accesibles en el sistema de ficheros local. Es decir, (1) crearlos en caso de que no existan previamente y (2) con permisos de acceso.
```
   
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

    Si no se comparten las cuentas (p.e. con NIS), puede causar problemas si entre los sistemas los identificadores de usuarios y grupos coinciden
    
    Seguridad: la característica root_squash se pone por defecto para que el superusuario de la máquina cliente no pueda tener privilegios especiales. no_root_squash quita esta protección.
    
    Restringir acceso mediante /etc/hosts.allow y /etc/hosts.deny.


