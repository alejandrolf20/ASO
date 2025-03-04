# Instalación y configuración inicial de OpenLDAP

## Enunciado

Realiza la instalación y configuración básica de OpenLDAP en una unidad de tu escenario de OpenStack, utilizando como base el nombre DNS asignado de tu proyecto. Deberás crear un usuario llamado asoprueba y configurar una máquina cliente basada en Debian y Rocky para que pueda validarse en servidor ldap configurado anteriormente con el usuario asoprueba.

---

## Introducción

En esta práctica, llevaremos a cabo la instalación y configuración inicial de OpenLDAP en una máquina dentro de nuestro entorno de OpenStack, utilizando el nombre DNS asignado al proyecto. OpenLDAP es un servicio de directorio ampliamente utilizado para la gestión centralizada de autenticaciones y permisos en entornos empresariales.

El objetivo principal es configurar un servidor LDAP funcional y asegurar que un cliente Debian y otro basado en Rocky puedan autenticarse en él utilizando el usuario `asoprueba`. Para garantizar la persistencia y accesibilidad de los datos de los usuarios, sus directorios personales (home) estarán alojados en un servidor **NFS**.

A lo largo de la práctica, se abordarán los siguientes puntos clave:

- Instalación y configuración de **OpenLDAP** en un servidor.
- Creación del usuario `asoprueba` dentro del servicio LDAP.
- Configuración de un cliente Debian y un cliente Rocky para autenticarse en LDAP.
- Montaje de los directorios personales de los usuarios en un servidor **NFS**.

Esta práctica permitirá comprender los fundamentos de la autenticación centralizada mediante LDAP, así como la integración con otros servicios esenciales como NFS, facilitando la administración de usuarios en entornos distribuidos.





