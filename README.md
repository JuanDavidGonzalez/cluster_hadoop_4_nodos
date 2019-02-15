# Cluster Hadoop
Los archivos  almacenados en este repositorio, corresponden al configuración de un cluster hadoop con las siguientes caracteristicas:  
- 5 Nodos: 1 master y 4 workers (master, node-1, node-2, node-3, node-4)  
- Almacenamiento Heterogeneo (RAMDISK, SSD, HDD)  
- Memoria limite contenedores YARN 64GB  
- Factor replica 3

### Requisitos
- 5 servidores o máquinas virtuales con distribución Linux (Ubuntu recomendado)
- Usuario hadoop en todos los nodos
- Java 8

### Configuración archivo hosts
Para que cada nodo se comunique usando nombres, editar el archivo **/etc/hosts** y agregar la dirección IP de los 5 servidores. No olvide reemplazar la IP de muestra con su IP:  

    192.0.2.1 master  
    192.0.2.2 node-1  
    192.0.2.3 node-2  
    193.0.2.4 node-3  
    192.0.2.5 node-4
    
### Instalar JAVA 8
Se debe instalar Java 8 en todos los nodos.

    $ sudo add-apt-repository ppa:webupd8team/java 
    $ sudo apt-get update
    $ sudo apt-get install oracle-java8-installer
    
Verificar Instalación Java:
	
    $ java -version
      
### Configuración Conexión entre los nodos  
El nodo maestro utilizará una conexión ssh para conectarse a los otros nodos mediante autenticación de par de claves.  

Se debe iniciar sesión en el nodo master con el usuario hadoop y generar una clave ssh:

	$ ssh-keygen -b 4096
        
Para copiar la clave a los otros nodos, jecutar los siguientes comandos e ingresar la  contraseña del usuario hadoop cuando se solicite. Si pregunta si desea agregar o no la clave a los hosts conocidos (known hosts), se debe indicar que si (yes):

    ssh-copy-id -i $HOME/.ssh/id_rsa.pub hadoop@master
    ssh-copy-id -i $HOME/.ssh/id_rsa.pub hadoop@node-1
    ssh-copy-id -i $HOME/.ssh/id_rsa.pub hadoop@node-2
    ssh-copy-id -i $HOME/.ssh/id_rsa.pub hadoop@node-3
    ssh-copy-id -i $HOME/.ssh/id_rsa.pub hadoop@node-4


### Montar RAMDISK
Para poder hacer uso del Almacenamiento en RAMDISK se debe crear primero dicha unidad, para crea un RAMDISK realizamos los siguientes pasos:

- Crear punto de montaje.
	
  	  $ sudo mkdir -p /mnt/RAM

- Montar RAMDisk indicando el tamaño de memoria que se desea asignar, en este caso se  asignan 30GB.

		$ sudo mount -t tmpfs -o size=30g tmpfs /mnt/RAM/  

  Ejecutar el comando **df -h**, para verificar el RAMDISK montado.  

- Posteriormente se debe hacer que el propietario del directorio donde se monto el RAMDisk, sea el usuario hadoop, por lo que se deben cambiar los permisos asignados.

		$ sudo chown -R hadoop:hadoop /mnt/RAM/     
		$ chmod 777 /mnt/RAM/
       
  Anexar la siguiente línea al archivo **/etc/fstab** para que el RAMDISK se monte automáticamente cada vez que se reinicien las máquinas.  
  
  		tmpfs /mnt/dn-tmpfs/ tmpfs nodev,uid=hadoop,gid=hadoop,noexec,nodiratime,size=30G 0 0/
  
### Montar Partición Disco

En este caso el disco SSD en el disco principal y se desea montar un disco HDD.

- Listar discos disponibles

		$ sudo lsblk
    
    Para verificar si el disco es SSD o HDD ejecutar:  
    
    	$ sudo cat /sys/block/sda/queue/rotational
    En este caso se esta verificando el disco sda, reemplazar sda, por el nombre del disco que se desea verificar.  
    
    Si la salida del comando es **1**, es un **HDD**, si la salida es **0**, es un **SSD**.
    
	En este caso se desea montar el disco sda, el disco que se desea montar debe tener por     lo menos una particion. Para crear una partición ejecutar:

      $ sudo fdisk /dev/sda
      >> opcion: m  (Ver menu)
      >> opcion: n  (nueva partición)
      >> opcion: p  (Primaria)
      >> opcion: w  (Escribir)
   
  Si se ejecuta el comando **lsblk** nuevamente se debe observar el disco con su     partición, en este caso sda1. para dar formato a la particion creada ejecutar el siguiente comando:  

	  $ sudo mkfs.ext4 /dev/sda1  


- Crear punto de montaje
	
  	  $ sudo mkdir -p /mnt/HDD
- Montar partición

	  $ sudo mount -t /dev/sda1 /mnt/HDD
    
  Ejecutar el comando **df -h** para verificar la partición montado.
  
  
  ### Configurar Variables de Entorno
  
  Editar el archivo **~.bashrc** y agregar las siguientes variables:
  
    export HADOOP_HOME=/home/hadoop/hadoop
    export HADOOP_MAPRED_HOME=/home/hadoop/hadoop
    export HADOOP_COMMON_HOME=/home/hadoop/hadoop
    export HADOOP_HDFS_HOME=/home/hadoop/hadoop
    export YARN_HOME=/home/hadoop/hadoop
    export HADOOP_COMMON_LIB_NATIVE_DIR=/home/hadoop/hadoop/lib/native
    export PATH=$PATH:/home/hadoop/hadoop/sbin:/home/hadoop/hadoop/bin
    export HADOOP_INSTALL=/home/hadoop/hadoop
	export LD_LIBRARY_PATH=/home/hadoop/hadoop/lib/native/:$LD_LIBRARY_PATH
  
## Descargar Hadoop y reemplazar archivos

Una vez realizadas todas las configuraciones previas indicadas, descargar descargar la versión más reciente disponible de hadoop, descomprimir los archivos y renombrar el directorio como **hadoop**, clonar este repositorio y reemplazar los  archivos aqui disponibles por los presentes en los archivos descargados de hadoop.  

Estos archivos son 6:  
  - core-site.xml
  - hadoop-env.sh
  - hdfs-site.xml
  - mapred-site.xml
  - workers
  - yarn-site.xml
 
 Los archivos que se deben remplazar se encuentran en **~hadoop/etc/hadoop/**
  
## Iniciar HDFS

Formatera el namenode:

	$ hdfs namenode -format
Iniciar el HDFS  

	$ start-dfs.sh
    
Crear directorio personal  en HDFS

	$ hdfs dfs -mkdir -p /user/hadoop

Crear directorios en HDFS para almacenamiento Heterogeneo

	$ hdfs dfs -mkdir RAM  
    $ hdfs dfs -mkdir HOT  
    $ hdfs dfs -mkdir HDD
    
Establecer politicas de Alamacenamiento para cada uno de los directorios creados:


	$ hdfs storagepolicies -setStoragePolicy -path RAM -policy LAZY_PERSIST
    $ hdfs storagepolicies -setStoragePolicy -path SSD -policy ALL_SSD
    $ hdfs storagepolicies -setStoragePolicy -path HDD -policy HOT
    
 De esta forma todo lo que se almacene en el directorio RAM del HDFS, sera ubicado en el RAMDISK, lo que se almacene en el directorio SSD, se ubicará en el disco SSD y lo que se almanene en el directorio HDD, se ubicara en el disco HDD.
