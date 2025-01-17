# Systemd-Boot

## Enunciado

Los desarrolladores de Debian han propuesto el uso de systemd-boot para instalaciones UEFI de Debian Trixie, que se lanzará en 2025. Opción disponible, de momento, en instalaciones debian 13 en modo experto. El objetivo es agregar soporte de arranque seguro firmado a Debian para intentar resolver el problema relacionado con UEFI y Secure Boot con sistemas Debian. Proponen utilizar un gestor de arranque llamado “systemd-boot” para mejorar el proceso de arranque de Debian en sistemas UEFI.


2. **Cambiar el tradicional gestor de arranque grub por systemd boot en una máquina virtual con debian 12.**

```
• Valora las ventajas y desventajas de este cambio.
    • Indica que versiones basadas en GNU/Linux están adoptando a systemd-boot como gestor de arranque por defecto.
```

```
• En máquina virtual basada en Debian 12, sustituye el gestor de arranque grub por systemd-boot, en el menú de arranque deberá aparecer las siguientes opciones:
```
```
Arranque de debian 12.
    Acceso a Firmware de la máquina virtual.
    Acceso a la shel EFI.
```

---

## 1.- Instala en máquina virtual, debian 13 con systemd-boot, y familiarízate con este nuevo gestor de arranque.

![53](img/53.png)

En esta parte de la práctica aprenderemos a instalar la nueva versión de Debian llamada **Trixie**, que corresponde a **Debian 13**; además usaremos el gestor de arranque `systemd-boot` en un sistema **UEFI**.

Lo primero que deberemos hacer será ir a la página oficial de [Debian](https://www.debian.org/devel/debian-installer/) para proceder con la descarga de la **ISO** correspondiente; podemos escoger la versión que más nos convenga, según nuestro equipo o demás necesidades; en este caso, escogeremos la versión **amd64 netinst**: 

![21](img/21.png)

Tras la correspondiente descarga, procederemos con el proceso de instalación, para ello utilizaremos **QEMU/KVM** como virtualizador.

Las características de la máquina virtual que vamos a crear no nos importaran mucho a la hora de la instalación, por lo que podemos dejar las que vienen predefinidas de por si; es importante escoger la opción que nos dirá **Personalizar configuración antes de instalar** para adecuar un pequeño ajuste antes de continuar:

![22](img/22.png)

Debemos cambiar el **firmware** que viene por defecto, ya que **Debian 13** utiliza **UEFI** como gestor de arranque junto con `systemd-boot`:

![23](img/23.png)

Para una correcta instalación de `systemd-boot` en **Debian 13**, debemos ejecutar una instalación en modo **experto**, para ello escogeremos la opción **Advanced options** y, posteriormente, **Expert install**:

![24](img/24.png)

Tras hacer las primeras configuraciones como el idioma o el teclado nos detendremos en la opción **Detectar y montar el medio de instalación**; esto es especialmente útil al realizar una instalación por red o con un medio de instalación que no es el disco duro local:

![25](img/25.png)

Cargaremos el módulo que nos aparece predefinido:

![26](img/26.png)

Seguidamente, se nos mostrará una lista de componentes adicionales pero, para la realización de la práctica, no necesitaremos ninguno:

![27](img/27.png)

Escogeremos la opción que configura la red de forma automática:

![28](img/28.png)

Seguiremos con la instalación hasta llegar a la parte del particionado de discos; esta parte es esencial para la correcta instalación de `systemd-boot`, ya que estamos configurando un sistema **UEFI**.

Escogeremos la opción de **particionado manual** y usaremos `gpt` como tipo tabla de particionado:

![29](img/29.png)

Es el esquema de particionado que utilizaremos será el siguiente:

1. **Partición EFI**: Esta partición es crucial para **UEFI**, ya que contiene los archivos del cargador de arranque, como será `systemd-boot` en este caso.

- **Tipo**: `EFI System Partition`
- **Tamaño**: 100 MB.
- **Punto de montaje**: `/boot/efi`

2. **Partición raíz (/)**: Contendrá el sistema operativo y todos los archivos de configuración.

- **Tipo**: `ext4`
- **Tamaño**: Gran parte del disco, en este caso unos 25 GB.
- **Punto de montaje**: `/`

3. **Área de intercambio (swap)**: Espacio de intercambio cuando la **RAM** está llena.

- **Tipo**: `Linux swap`
- **Tamaño**: Espacio restante, en este caso unos 150 MB.

Gráficamente quedará así:

![30](img/30.png)

Tras esto, toca la parte de la instalación del sistema base, para ello escogemos el núcleo que queramos:

![31](img/31.png)

Escogeremos la opción que que carga solo los cargadores esenciales:

![32](img/32.png)

Denegamos el análisis de medios de instalación adicionales:

![33](img/33.png)

Utilizaremos una **réplica de red**, ya que esto proporcionará los paquetes necesarios para completar la instalación:

![34](img/34.png)

Escogemos **http** como protocolo para acceder a la **réplica de red**:

![35](img/35.png)

Seleccionar **Sí** en esta opción permitirá que el instalador cargue y use los firmwares no libres que puedan ser necesarios para que nuestro hardware funcione correctamente:

![36](img/36.png)

Escogemos que usamos **software no libre**:

![37](img/37.png)

Activamos los repositorios de fuentes en APT:

![38](img/38.png)

Escogeremos las opciones que vienen marcadas por defecto, las cuales indican que Debian tendrá habilitadas las actualizaciones de seguridad y de distribución:

![39](img/39.png)

No será necesario que se realicen actualizaciones automáticas, por lo que denegaremos esta opción:

![40](img/40.png)

Elegimos los programas a instalar según nuestras necesidades:

![41](img/41.png)

Tras varios minutos en los que se descargan e instalan los paquetes del sistema, seleccionaremos el cargador de arranque `systemd-boot`:

![42](img/42.png)

Tras finalizarse la instalación, se efectuará un reinicio y se nos mostrará la siguiente pantalla, en la que escogeremos **Device Manager**:

![43](img/43.png)

Escogeremos **Secure Boot Configuration**:

![44](img/44.png)

Y desactivaremos el **Secure Boot**:

![45](img/45.png)

Esta configuración la hemos realizado para que el cargador `systemd-boot` funcione correctamente, ya que **Secure Boot** es una característica de **UEFI** diseñada para evitar que se cargue software no firmado o malicioso al iniciar el sistema, por lo que `systemd-boot` será bloquedo por el **firmware** del sistema si está característica permanece activa.

Tras guardar los cambios se iniciará el sistema correctamente:

![46](img/46.png)

Si utilizamos el comando `bootctl status` podemos verificar que el cargador de arranque `systemd-boot` se ha instalado y configurado correctamente:

![47](img/47.png)

Además, podemos utilizar la herramienta `neofetch` para comprobar la versión de nuestro sistema operativo:

![48](img/48.png)

---

## 2.- Cambiar el tradicional gestor de arranque grub por systemd boot en una máquina virtual con debian 12.

![55](img/55.png)

### Ventajas y Desventajas

### **Systemd-boot**

#### **Ventajas:**

- **Ligereza y simplicidad:**  
  systemd-boot se caracteriza por su diseño minimalista, que lo hace menos pesado que GRUB. Este enfoque reducido puede resultar en un arranque más rápido y un menor uso de recursos del sistema.

- **Integración con systemd:**  
  Diseñado específicamente para trabajar en conjunto con systemd, este gestor de arranque mejora la administración en sistemas donde systemd ya es el componente principal.

- **Velocidad de arranque optimizada:**  
  Gracias a su estructura eficiente, systemd-boot puede reducir los tiempos de inicio, ofreciendo una experiencia más fluida.

#### **Desventajas:**

- **Opciones de configuración limitadas:**  
  Al priorizar la simplicidad, systemd-boot carece de algunas funcionalidades avanzadas que GRUB proporciona, lo que puede ser un inconveniente para usuarios que requieren configuraciones complejas. Además, está diseñado exclusivamente para sistemas con EFI, limitando su compatibilidad.

- **Gestión de sistemas operativos múltiples:**  
  A diferencia de GRUB, que permite administrar varios sistemas operativos desde un menú centralizado, systemd-boot puede requerir configuraciones adicionales para lograr un resultado similar.

---

### **GRUB**

#### **Ventajas:**

- **Compatibilidad con arranques duales y múltiples sistemas:**  
  GRUB es ideal para gestionar varios sistemas operativos en una misma máquina, ya que su menú interactivo facilita la selección del sistema deseado.

- **Personalización avanzada:**  
  Ofrece una amplia gama de opciones de configuración que permiten a los usuarios ajustar el arranque según sus necesidades específicas, siendo una herramienta robusta para usuarios avanzados.

#### **Desventajas:**

- **Curva de aprendizaje más pronunciada:**  
  La gran cantidad de opciones de configuración puede resultar intimidante para usuarios principiantes. Modificar sus archivos manualmente puede ser complicado sin conocimientos previos.

- **Arranque más lento:**  
  En comparación con gestores más simples como systemd-boot, GRUB puede ser más lento al iniciar el sistema, lo que podría no ser ideal para quienes priorizan la rapidez.

---

### Distribuciones de GNU/Linux que utilizan systemd-boot por defecto

Desde su adopción inicial por Fedora en 2011 como parte del sistema de inicio systemd, varias distribuciones han optado por systemd-boot como gestor de arranque predeterminado. Entre ellas se encuentran:

- **Arch Linux**
- **Clear Linux**
- **Pop!_OS**
- **Fedora**
- **Antergos**

Estas distribuciones valoran la integración estrecha de systemd-boot con systemd, así como su enfoque en la velocidad y simplicidad.

---

### Cambio de GRUB a systemd-boot en Debian 12

Usaremos una máquina con el sistema **Debian 12** instalado y con **GRUB** como gestor de arranque:

![49](img/49.png)

La máquina en cuestión presentará el siguiente esquema de particionado:

```
usuario@usuario:~$ lsblk -f
NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sr0                                                                           
vda                                                                           
├─vda1 vfat   FAT32       9F19-0304                             469,2M     1% /boot/efi
├─vda2 ext4   1.0         2b2fea1e-c8ab-4f03-887a-0287db6c3474   12,1G    28% /
└─vda3 swap   1           263a2cfd-ce3d-4918-af5c-73cd7a488a80                [SWAP]
```

Es importante que la partición de arranque se ecuentre en formato `vfat`, ya que el gestor de arranque `systemd-boot` no es comaptible con ningún otro sistema de ficheros.

Comenzaremos usando el comando `sudo apt install systemd-boot` que instalará todos los binarios que necesitaremos para realizar el cambio de gestor de arranque.

Usaremos el siguiente comando para instalar el cargador de arranque:

```usuario@usuario:~$ sudo bootctl --path=/boot/efi install
Copied "/usr/lib/systemd/boot/efi/systemd-bootx64.efi" to "/boot/efi/EFI/systemd/systemd-bootx64.efi".
Copied "/usr/lib/systemd/boot/efi/systemd-bootx64.efi" to "/boot/efi/EFI/BOOT/BOOTX64.EFI".
Random seed file /boot/efi/loader/random-seed successfully written (32 bytes).
Not installing system token, since we are running in a virtualized environment.
Created EFI boot entry "Linux Boot Manager".
```

Para comprobar que se han creado los archivos y directorios correspondientes, usaremos el siguiente comando:

```
usuario@usuario:~$ sudo ls -l /boot/efi/loader/
total 16
drwx------ 2 root root 4096 ene 17 08:26 entries
-rwx------ 1 root root    6 ene 17 08:26 entries.srel
-rwx------ 1 root root   73 ene 17 08:26 loader.conf
-rwx------ 1 root root   32 ene 17 08:28 random-seed
```

Instalamos el cargador de inicialización de `systemd-boot`:

```
usuario@usuario:~$ sudo bootctl install
Copied "/usr/lib/systemd/boot/efi/systemd-bootx64.efi" to "/boot/efi/EFI/systemd/systemd-bootx64.efi".
Copied "/usr/lib/systemd/boot/efi/systemd-bootx64.efi" to "/boot/efi/EFI/BOOT/BOOTX64.EFI".
Random seed file /boot/efi/loader/random-seed successfully written (32 bytes).
Not installing system token, since we are running in a virtualized environment.
Created EFI boot entry "Linux Boot Manager".
```

Tras esto, editaremos el fichero `loader.conf` para configurar las opciones de arranque:

```
usuario@usuario:~$ sudo cat /boot/efi/loader/loader.conf
timeout 3
console-mode keep
editor yes
default debian.conf
```

Crearemos un fichero de configuración llamado `debian.conf`
```
usuario@usuario:~$ sudo cat /boot/efi/loader/entries/debian.conf
title Sysytemd-boot Alejandro
linux /boot/vmlinuz-6.1.0-30-amd64
initrd /boot/initrd.img-6.1.0-30-amd64
options root=UUID=2b2fea1e-c8ab-4f03-887a-0287db6c3474 rw
```

En este, hemos especificado el **kernel** que vamos a iniciar, la imagen **initrd** y el **UUID** de ña partición raíz que se montará (podemos visualizarla con el comando `blkid`).

Creamos otro fichero para la **Shell EFI**:

```
usuario@usuario:~$ sudo cat /boot/efi/loader/entries/shellefi.conf
title EFI Shell
efi /EFI/shellx64.efi
```

Descargamos el **firmware** necesario:

```
usuario@usuario:~$ wget https://github.com/holoto/efi_shell_flash_bios/blob/master/Shellx64.efi
--2025-01-17 08:40:08--  https://github.com/holoto/efi_shell_flash_bios/blob/master/Shellx64.efi
Resolviendo github.com (github.com)... 140.82.121.4
Conectando con github.com (github.com)[140.82.121.4]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: no especificado [text/html]
Grabando a: «Shellx64.efi»

Shellx64.efi                  [ <=>                                  ] 223,54K  1,40MB/s    en 0,2s    

2025-01-17 08:40:10 (1,40 MB/s) - «Shellx64.efi» guardado [228904]
```

Lo movemos a la ruta que especificamos anteriormente:

```
usuario@usuario:~$ sudo mv Shellx64.efi /boot/efi/EFI/
```

Podemos ver que se ha incorporado correctamente al directorio deseado con el siguiente comando:

```
usuario@usuario:~$ sudo ls -l /boot/efi/EFI/
total 240
drwx------ 2 root root   4096 ene 17 08:30 BOOT
drwx------ 2 root root   4096 ene 16 21:32 debian
drwx------ 2 root root   4096 ene 17 08:26 Linux
-rwx------ 1 root root 228904 ene 17 08:40 Shellx64.efi
drwx------ 2 root root   4096 ene 17 08:30 systemd
```

Desactivamos el **Secure Boot**, ya que, como comentamos en el ejercicio anterior, este es incompatible con el nuevo gestor de arranque:

```
usuario@usuario:~$ sudo mokutil --disable-validation
password length: 8~16
input password: 
input password again:
```

Actualizamos el gestor de arranque:

```
usuario@usuario:~$ sudo bootctl update
Skipping "/boot/efi/EFI/systemd/systemd-bootx64.efi", since same boot loader version in place already.
Skipping "/boot/efi/EFI/BOOT/BOOTX64.EFI", since same boot loader version in place already.
```

Comprobamos el orden de arranque con la herramienta `efibootmgr`:

```
usuario@usuario:~$ efibootmgr
BootCurrent: 0004
Timeout: 0 seconds
BootOrder: 0001,0004,0002,0000,0003
Boot0000* UiApp
Boot0001* Linux Boot Manager
Boot0002* UEFI Misc Device
Boot0003* EFI Internal Shell
Boot0004* debian
```

Como el gestor que queremos utilizar está marcado con el ID **0001**, usaremos el siguiente comando para que se use correctamente :

```
sudo efibootmgr -o 0001
```

Tras efectuar un reinicio del sistema, podremos comprobar que el gestor ha cargado correctamente:

![50](img/50.png)

Si accedemos al sistema, podremos comprobar que, el gestor de arranque es el correcto:

![51](img/51.png)

También podremos acceder a la **Shell EFI** sin ningún problema:

![52](img/52.png)