# Creación de un sistema automatizado de instalación I

## Enunciado
Instalación automatizada basada en medio de almacenamiento local. Instalación basada en fichero de configuración preseed:
Creación de un sistema automatizado de instalación en distribución debian 12 bookworm.
Se deberá configurar el sistema para que se responda automáticamente a todos los items en la instalación. Las diferentes contraseñas deberán codificarse para que no aparezcan en texto plano. Se trabajará con el siguiente esquema de particiones/volúmenes. La partición efi, FAT 32, y la partición /boot, ext4, serán particiones fuera del esquema de volúmenes lógicos, el resto de volúmenes, seguirá el siguiente esquema lvm, volúmenes lógicos /, home y var. Los tamaños serán los apropiados en cada caso teniendo en cuenta el tamaño del disco empleado.

