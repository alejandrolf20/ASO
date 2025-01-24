# Recolección centralizada de logs de sistema, mediante journald

## Enunciado

Implementa en tu escenario de trabajo de Openstack, un sistema de recolección de log mediante journald. Para ello debes, implementar un sistema de recolección de log mediante el paquete systemd-journal-remote, o similares.

---

## Introducción

Esta práctica se basa en implementar un sistema de recoleccióm de **log** mediante la herramienta **journald**; para ello, usaremos el paquete **systemd-journal-remote**, el cual debemos instalar en cada una de las máquinas de nuestro escenario.

El escenario en cuestión cuenta con 4 máquinas con nombres inspirados en la saga de anime **One Piece**:

- **Luffy**: está conectada a dos redes, una de ellas con acceso al exterior, está basada en la distribución **Debian 12**, actúa como router y en esta práctica será la **autoridad de certificación** y la máquina que actuará como **servidor de journald**.
- **Zoro**: conectada a una red interna por la cuál recibe la conexión al exterior gracias a **Luffy**, está basada en la distribución **Rocky Linux 9**.
- **Nami**: contenedor **Ubuntu 22.04** alojado en **Luffy**.
- **Sanji**: contenedor **Ubuntu 22.04** alojado en **Luffy**.

---

## Implementación

El primer paso que realizaremos será instalar el paquete **systemd-journal-remote**, en todas de ellas se realizará con el comando `apt`, salvo en **Zoro**, que utilizaremos `dnf`:

```
alejandro@luffy:~$ sudo apt install systemd-journal-remote -y
alejandro@nami:~$ sudo apt install systemd-journal-remote -y
alejandro@sanji:~$ sudo apt install systemd-journal-remote -y
[alejandro@zoro ~]$ sudo dnf install systemd-journal-remote -y

```

En **Luffy**, al ser la máquina que actuará como servidor, activaremos los dos componentes necesarios para poder recibir mensajes de registro:

```
alejandro@luffy:~$ sudo systemctl enable --now systemd-journal-remote.socket
Created symlink /etc/systemd/system/sockets.target.wants/systemd-journal-remote.socket → /lib/systemd/system/systemd-journal-remote.socket.
alejandro@luffy:~$ sudo systemctl enable systemd-journal-remote.service
```

En el resto de máquinas (clientes) activaremos el componente que **systemd** utilizará para enviar mensajes de registro al servidor:

```
[alejandro@zoro ~]$ sudo systemctl enable systemd-journal-upload.service
Created symlink /etc/systemd/system/multi-user.target.wants/systemd-journal-upload.service → /usr/lib/systemd/system/systemd-journal-upload.service.
```

Para implementar el servicio de recolección, utilizaremos un sistema de cifrado para que nadie pueda acceder a nuestros registros; para ello utilizaremos la herramienta **Easy RSA** en **Luffy** para generar los certificados y claves de todas las máquinas:

```
alejandro@luffy:~$ sudo apt install easy-rsa openssl -y
```

Esta herramienta trae un fichero de ejemplo para simplificar la generación de certificados llamado `vars.example`:

```
alejandro@luffy:~$ cd /usr/share/easy-rsa/
alejandro@luffy:/usr/share/easy-rsa$ ls
easyrsa  openssl-easyrsa.cnf  vars.example  x509-types
```

Utilizaremos este fichero para implementar los datos con los que trabajará nuestra **CA**:

```
alejandro@luffy:/usr/share/easy-rsa$ cat vars
set_var EASYRSA_REQ_COUNTRY     "ES"
set_var EASYRSA_REQ_PROVINCE    "Sevilla"
set_var EASYRSA_REQ_CITY        "Dos Hermanas"
set_var EASYRSA_REQ_ORG         "iesgn"
set_var EASYRSA_REQ_EMAIL       "alejandroliafru@gmail.com"
set_var EASYRSA_REQ_OU          "ASIR2"
```

Para comenzar a trabajar con esta herramienta, usaremos el siguiente comando para generar un directorio estructurado, dónde se almacenarán nuestros certificados y claves llamado `pki`:

```
alejandro@luffy:/usr/share/easy-rsa$ sudo ./easyrsa init-pki
* Notice:

  init-pki complete; you may now create a CA or requests.

  Your newly created PKI dir is:
  * /usr/share/easy-rsa/pki
```

El siguiente paso, será crear nuestra **CA**, para ello usaremos el siguiente comando:

```
alejandro@luffy:/usr/share/easy-rsa$ sudo ./easyrsa build-ca nopass
* Notice:
Using Easy-RSA configuration from: /usr/share/easy-rsa/vars

* WARNING:

  Move your vars file to your PKI folder, where it is safe!

* Notice:
Using SSL: openssl OpenSSL 3.0.15 3 Sep 2024 (Library: OpenSSL 3.0.15 3 Sep 2024)

Using configuration from /usr/share/easy-rsa/pki/2c4b91b3/temp.af7be5c2
..+...+.......+.....+.+.....+.........+....+.........+.....+......+.+...+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..............+......+...........+............+.+..+...+....+.....+............+.+.....+.......+...+.........+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+.+...+.....+......+.+.....+...+......................+......+...+.....+......+...+...+.......+......+.....+.+...........+...+......+.+.........+........+.............+.....+....+.....+......+.......+..+...............+....+............+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
..+...+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*...+........+.+......+...+......+..+...+....+..+....+..............+.+.....+.+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*....+.........+....+..+...+.......+...+..+......+....+........+...+....+...+.....+......+.........+.+.........+.....+...+...+.+..............................+........+.+.....+.+...+......+......+.....+...............+.+.........+...+........+...+.+......+...+.........+........+.......+..+...+.+.....+....+.....+..........+..+.........+.......+.........+.....+.+..+............+.........+....+...+........................+......+..+.+..+...+.+........+.......+......+..+.+..+...+.........+..........+..+.+..............+...+.......+........+...+...+.......+...+.....+..........+.....+.+.....+.........+.......+..+............+......+.............+...............+.....+.+...+............+........+.......+.....+.+.....+..................+.+...........+...+....+...+...+..+.............+..+..........+............+..+.........+.......+.....+.......+...+..+...+.......+......+...........+...+...+.......+..+......+.............+.....+......+.+..............+.+.........+.....+......+...+.....................+....+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:luffy.alejandrolf.gonzalonazareno.org

* Notice:

CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/usr/share/easy-rsa/pki/ca.crt
```

El resultado de la instrucción anterior generará un certificado llamado `ca.crt`, el cual ha usado las variables que declaramos anteriormente en el fichero `vars`; además, hemos introducido un nombre para nuestra **CA**, en este caso ha sido **luffy.alejandrolf.gonzalonazareno.org**.

Tras construir nuestra **CA**, generaremos la clave privada y la solicitud de firma de cada una de las máquinas de nuestro escenario, incluyendo **Luffy**.

Comenzaremos con **Zoro**:

```
alejandro@luffy:/usr/share/easy-rsa$ sudo ./easyrsa gen-req zoro nopass
* Notice:
Using Easy-RSA configuration from: /usr/share/easy-rsa/vars

* WARNING:

  Move your vars file to your PKI folder, where it is safe!

* Notice:
Using SSL: openssl OpenSSL 3.0.15 3 Sep 2024 (Library: OpenSSL 3.0.15 3 Sep 2024)

.+....+...........+...+....+...+.........+.....+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..+......+......+.+........+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*...+.........+......+..........+...+......+...+.....+...+...+....+.....+.+.....+....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..+...+...+....+..+.+...+.........+...+...+..+.............+......+...+....................+..................+...+.+........+...+...+.+......+.....+.......+..+........................+.+...+.....+...+...+......+....+..............+.+.....+.+......+..+...+....+...+......+...........+....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [zoro]:zoro.alejandrolf.gonzalonazareno.org
* Notice:

Keypair and certificate request completed. Your files are:
req: /usr/share/easy-rsa/pki/reqs/zoro.req
key: /usr/share/easy-rsa/pki/private/zoro.key
```

Como vemos, se generarán dos ficheros, llamados `zoro.req` y `zoro.key`, que corresponden a la solicitud de firma y a la clave privada respectivamente.

Utilizaremos la misma instrucción con el resto de máquinas de nuestra escenario, siguiendo por **Nami**:

```alejandro@luffy:/usr/share/easy-rsa$ sudo ./easyrsa gen-req nami nopass
* Notice:
Using Easy-RSA configuration from: /usr/share/easy-rsa/vars

* WARNING:

  Move your vars file to your PKI folder, where it is safe!

* Notice:
Using SSL: openssl OpenSSL 3.0.15 3 Sep 2024 (Library: OpenSSL 3.0.15 3 Sep 2024)

.+....+.....+...+.+..+...+.......+..+...+....+......+..+.+...........+....+......+...+...+.....+...+......+....+.....+....+...............+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*...+..+......+.....................+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+.+..+............+.+..............+....+.....+....+..+...+...+....+...+..+....+..+....+...+...+.........+.........+.....+.+.........+............+........+....+..+....+..............................+..+...+.......+...+..+.......+..+...+............+.........+......+.........+....+...+.....+...+....+...+..+...+.......+.....+...+...+.+......+.....+.......+.....+.+...............+.....+.+.....+.+........+.+.....+......+.......+..+...............+.............+...+........+......+.+..+......+.........+.+....................+.+.....+.+.....+...+....+......+....................+.+...+.....+.+...........+...+................+.....+.+.....+.........+....+...........+....+...+............+.....+......+....+.....+.+..+...+.......+.........+.....+.+..............+.+...............+...+..+..........+..+.+..+....+.....+...+...............+......+.......+..+.+.................+..................+......+....+..+...+....+......+..+.............+.....+.........+.......+.....+.........+.+......+..+.+.........+...+..+...+.........+......+............+......................+......+...+.........+......+......+.....+....+.....+.+...+...............+..+.........+....+........+.........+...+..........+.....+...+.............+.....+.+......+...+......+...........+...+.......+...............+..+.+.................+....+...+...+............+..+...+......+.......+...+..+.........+...+.............+.....+.......+.....+.......+...+.....+...+...+.......+..+...............+......+......+....+.....+....+..+.+.....+..........+..............+.+..+.......+...+........+...+..........+.........+..+....+......+........+.+.....+.........+...+....+.....+................+.........+.........+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
.+..........+..+.......+...+...+..+.......+.........+......+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+...+..+.+..+....+.....+...+.+..+...+..........+.....+.+........+.+.....+.......+........+.+......+..+...+...+..........+..+............+...+....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*................+.+...............+......+......+..+.+..+.......+..+..........+........+...+...+.+...+..+.+.....+.......+..+......+...+............+.......+.....+................+.....+.+...+...+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [nami]:nami.alejandrolf.gonzalonazareno.org
* Notice:

Keypair and certificate request completed. Your files are:
req: /usr/share/easy-rsa/pki/reqs/nami.req
key: /usr/share/easy-rsa/pki/private/nami.key
```

Ahora es el turno de **Sanji**:

```
alejandro@luffy:/usr/share/easy-rsa$ sudo ./easyrsa gen-req sanji nopass
* Notice:
Using Easy-RSA configuration from: /usr/share/easy-rsa/vars

* WARNING:

  Move your vars file to your PKI folder, where it is safe!

* Notice:
Using SSL: openssl OpenSSL 3.0.15 3 Sep 2024 (Library: OpenSSL 3.0.15 3 Sep 2024)

..........+......+.+...+......+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*....+.......+..+.+...........+.+........+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*....+.+...+.....+...+...+....+..+...................+..+.............+.....+...............+....+.................+.+...+.....+.+...+..+.........+.+.....+......+..........+..+.+..+.........+...+.+..+....+...+...+..+.......+.....+................+..+.............+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
...+..+.+.....+.......+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*...........+...+..........+...+..+.......+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*........+..+...+.......+......+......+........+.+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [sanji]:sanji.alejandrolf.gonzalonazareno.org
* Notice:

Keypair and certificate request completed. Your files are:
req: /usr/share/easy-rsa/pki/reqs/sanji.req
key: /usr/share/easy-rsa/pki/private/sanji.key
```

Y por último, el turno de **Luffy**:

```
alejandro@luffy:/usr/share/easy-rsa$ sudo ./easyrsa gen-req luffy nopass
* Notice:
Using Easy-RSA configuration from: /usr/share/easy-rsa/vars

* WARNING:

  Move your vars file to your PKI folder, where it is safe!

* Notice:
Using SSL: openssl OpenSSL 3.0.15 3 Sep 2024 (Library: OpenSSL 3.0.15 3 Sep 2024)

.+..+.........+.+............+........+...+.......+.....+....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..........+.........+...............+.+...+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..................+..+.......+.....+.+...+..+...+.+........+............+.+...+.....+....+...............+.....+...+....+...+.....+...+.........+.+...+.....+.........+.+.....+....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
.+..............+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+..+.......+......+.........+........+...+...+......+...............+....+.........+...+..+...+....+...+......+......+...+.....+.+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*....+.............+.....+...+.+.....................+..+...+....+........+...+..................+.........+...............+...+....+...+.....+...+...+....+..+.+..................+.....+......+..........+...+..+.+...+..+.........+..................+.+..+...+.+........+.+......+......+..+......+...............+.+...+.....+...+...+......+....+...........+.+.....+.........+.........+....+...+...+......+.........+...............+......+...........+...+....+........+...+....+.....+.......+.....+...+......+.........+...+............+......+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [luffy]:luffy.alejandrolf.gonzalonazareno.org
* Notice:

Keypair and certificate request completed. Your files are:
req: /usr/share/easy-rsa/pki/reqs/luffy.req
key: /usr/share/easy-rsa/pki/private/luffy.key
```

Una vez generados todos los archivos, llega el momento de firmar las solicitudes de firma como buena **CA** que somos; para ello, usaremos la siguiente instrucción para firmar la de **Luffy**;

```
alejandro@luffy:/usr/share/easy-rsa$ sudo ./easyrsa sign-req server luffy
* Notice:
Using Easy-RSA configuration from: /usr/share/easy-rsa/vars

* WARNING:

  Move your vars file to your PKI folder, where it is safe!

* Notice:
Using SSL: openssl OpenSSL 3.0.15 3 Sep 2024 (Library: OpenSSL 3.0.15 3 Sep 2024)


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a server certificate for 825 days:

subject=
    commonName                = luffy.alejandrolf.gonzalonazareno.org


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes

Using configuration from /usr/share/easy-rsa/pki/606a3ac2/temp.540b094a
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'luffy.alejandrolf.gonzalonazareno.org'
Certificate is to be certified until Apr 28 16:30:03 2027 GMT (825 days)

Write out database with 1 new entries
Database updated

* Notice:
Certificate created at: /usr/share/easy-rsa/pki/issued/luffy.crt
```

Como vemos, se generá un certificado ya firmado por nuestra **CA** llamado `luffy.crt`.

Lo siguiente que haremos será firmar el certificado de cada una de las máquinas restantes, continuando por **Zoro**:

```
alejandro@luffy:/usr/share/easy-rsa$ sudo ./easyrsa sign-req server zoro
* Notice:
Using Easy-RSA configuration from: /usr/share/easy-rsa/vars

* WARNING:

  Move your vars file to your PKI folder, where it is safe!

* Notice:
Using SSL: openssl OpenSSL 3.0.15 3 Sep 2024 (Library: OpenSSL 3.0.15 3 Sep 2024)


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a server certificate for 825 days:

subject=
    commonName                = zoro.alejandrolf.gonzalonazareno.org


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes

Using configuration from /usr/share/easy-rsa/pki/6b3cc4ff/temp.15048ea9
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'zoro.alejandrolf.gonzalonazareno.org'
Certificate is to be certified until Apr 28 16:30:12 2027 GMT (825 days)

Write out database with 1 new entries
Database updated

* Notice:
Certificate created at: /usr/share/easy-rsa/pki/issued/zoro.crt
```

Usaremos la misma instrucción para **Sanji** y **Nami**; una vez tengamos los certificados ya firmados, debemos **enviarlos** a las máquinas correspondientes junto con el certificado de laa **CA**, llamado `ca.crt` y la clave privada correspondiente a cada máquina.

Como **Luffy** es la máquina que actuará como servidor, debemos unir su certificado firmado y su clave privada en un solo archivo; para ello usamos la siguiente instrucción que generará un fichero llamado `combined.pem`:

```
alejandro@luffy:~$ sudo cat luffy.crt luffy.key > combined.pem
```

Crearemos un directorio, dónde almacenaremos los archivos que necesitaremos, para encontrarlos de una mánera más sencilla:

```
alejandro@luffy:~$ sudo mkdir -p /etc/systemd/journal-remote/
alejandro@luffy:~$ sudo mv /home/alejandro/luffy.key /etc/systemd/journal-remote/
alejandro@luffy:~$ sudo mv /home/alejandro/luffy.crt /etc/systemd/journal-remote/
alejandro@luffy:~$ sudo mv /home/alejandro/combined.pem /etc/systemd/journal-remote/
```

Estos archivos, deben de tener los siguientes permisos y propietarios:

```
alejandro@luffy:~$ sudo ls -l /etc/systemd/journal-remote/
total 20
-rw-r--r-- 1 root systemd-journal-remote 6581 Jan 23 18:46 combined.pem
-rw-r----- 1 root systemd-journal-remote 4877 Jan 23 16:30 luffy.crt
-rw-r----- 1 root systemd-journal-remote 1704 Jan 23 16:29 luffy.key
```

Para continuar, configuraremos el servicio en **Luffy**, indicando las rutas de los archivos anteriores:

```
alejandro@luffy:~$ cat /etc/systemd/journal-remote.conf
[Remote]
Seal=false
SplitMode=host
ServerKeyFile=/etc/systemd/journal-remote/luffy.key
ServerCertificateFile=/etc/systemd/journal-remote/luffy.crt
TrustedCertificateFile=/etc/systemd/journal-remote/combined.pem
```

Una vez configurado, reiniciaremos el servicio y comprobaremos que funciona correctamente:

```
alejandro@luffy:~$ sudo systemctl restart systemd-journal-remote.service
alejandro@luffy:~$ sudo systemctl status systemd-journal-remote.service
● systemd-journal-remote.service - Journal Remote Sink Service
     Loaded: loaded (/lib/systemd/system/systemd-journal-remote.service; indirect; preset: disabled)
     Active: active (running) since Thu 2025-01-23 18:59:33 UTC; 1s ago
TriggeredBy: ● systemd-journal-remote.socket
       Docs: man:systemd-journal-remote(8)
             man:journal-remote.conf(5)
   Main PID: 9110 (systemd-journal)
     Status: "Processing requests..."
      Tasks: 1 (limit: 2309)
     Memory: 1.7M
        CPU: 57ms
     CGroup: /system.slice/systemd-journal-remote.service
             └─9110 /lib/systemd/systemd-journal-remote --listen-https=-3 --output=/var/log/journal/remote/

Jan 23 18:59:33 luffy systemd[1]: Started systemd-journal-remote.service - Journal Remote Sink Service.
Jan 23 18:59:33 luffy systemd-journal-remote[9110]: Certificate checking disabled.
Jan 23 18:59:33 luffy systemd-journal-remote[9110]: microhttpd: MHD_OPTION_EXTERNAL_LOGGER is not the first option specified for the daemon. Some messages may be printed by the standard MHD logger.
```

Tras esto, configuraremos todos los clientes de la misma manera, por lo que veremos los pasos a seguir en **Zoro**.

Crearemos un directorio dónde almacenaremos los ficheros que nos transferimos desde **Luffy**:

```
[alejandro@zoro ~]$ sudo mkdir -p /etc/systemd/journal-remote/
[alejandro@zoro ~]$ sudo mv * /etc/systemd/journal-remote/
```

Los permisos y propietarios correspondientes son los siguientes:

```
[alejandro@zoro ~]$ ls -l /etc/systemd/journal-remote/
total 16
-rw-r-----. 1 systemd-journal-upload systemd-journal-upload 1310 Jan 23 16:57 ca.crt
-rw-r-----. 1 systemd-journal-upload systemd-journal-upload 4871 Jan 23 17:04 zoro.crt
-rw-r-----. 1 systemd-journal-upload systemd-journal-upload 1704 Jan 23 17:04 zoro.key
```

Editaremos el fichero `journal-upload.conf` con las rutas de los archivos que necesitamos:

```
[alejandro@zoro ~]$ cat /etc/systemd/journal-upload.conf 
[Upload]
URL=https://luffy.alejandrolf.gonzalonazareno.org/
ServerKeyFile=/etc/systemd/journal-remote/zoro.key
ServerCertificateFile=/etc/systemd/journal-remote/zoro.crt
TrustedCertificateFile=/etc/systemd/journal-remote/ca.crt
```

Reiniciamos el servicio y comprobamos su estado:

```
[alejandro@zoro ~]$ sudo systemctl restart systemd-journal-upload.service
[alejandro@zoro ~]$ sudo systemctl status systemd-journal-upload.service
× systemd-journal-upload.service - Journal Remote Upload Service
     Loaded: loaded (/usr/lib/systemd/system/systemd-journal-upload.service; enabled; preset: disabled)
     Active: failed (Result: exit-code) since Fri 2025-01-24 08:17:14 UTC; 1s ago
   Duration: 83ms
       Docs: man:systemd-journal-upload(8)
    Process: 40089 ExecStart=/usr/lib/systemd/systemd-journal-upload --save-state (code=exited, status=1/FAILURE)
   Main PID: 40089 (code=exited, status=1/FAILURE)
     Status: "Shutting down..."
      Error: 5 (Input/output error)
        CPU: 81ms

Jan 24 08:17:13 zoro.alejandrolf.gonzalonazareno.org systemd[1]: Started Journal Remote Upload Service.
Jan 24 08:17:14 zoro.alejandrolf.gonzalonazareno.org systemd-journal-upload[40089]: Upload to https://luffy.alejandrolf.gonzalonazareno.org:19532/upload failed: could not load PEM client certificate, OpenSSL er>
Jan 24 08:17:14 zoro.alejandrolf.gonzalonazareno.org systemd[1]: systemd-journal-upload.service: Main process exited, code=exited, status=1/FAILURE
Jan 24 08:17:14 zoro.alejandrolf.gonzalonazareno.org systemd[1]: systemd-journal-upload.service: Failed with result 'exit-code'.
```

Tras este fallo inesperado, probaremos la conexión con **Luffy** de manera manual:

```
[alejandro@zoro ~]$ openssl s_client -connect luffy.alejandrolf.gonzalonazareno.org:19532
Connecting to 172.16.0.1
CONNECTED(00000003)
depth=0 CN=luffy.alejandrolf.gonzalonazareno.org
verify error:num=20:unable to get local issuer certificate
verify return:1
depth=0 CN=luffy.alejandrolf.gonzalonazareno.org
verify error:num=21:unable to verify the first certificate
verify return:1
depth=0 CN=luffy.alejandrolf.gonzalonazareno.org
verify return:1
---
Certificate chain
 0 s:CN=luffy.alejandrolf.gonzalonazareno.org
   i:CN=luffy.alejandrolf.gonzalonazareno.org
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Jan 23 16:30:03 2025 GMT; NotAfter: Apr 28 16:30:03 2027 GMT
---
Server certificate
-----BEGIN CERTIFICATE-----
MIID2TCCAsGgAwIBAgIQatb4xvmf/6n8/xH+ZidpcDANBgkqhkiG9w0BAQsFADAw
MS4wLAYDVQQDDCVsdWZmeS5hbGVqYW5kcm9sZi5nb256YWxvbmF6YXJlbm8ub3Jn
MB4XDTI1MDEyMzE2MzAwM1oXDTI3MDQyODE2MzAwM1owMDEuMCwGA1UEAwwlbHVm
ZnkuYWxlamFuZHJvbGYuZ29uemFsb25hemFyZW5vLm9yZzCCASIwDQYJKoZIhvcN
AQEBBQADggEPADCCAQoCggEBAKTAWzEHpXHf5B7dL4dxNlBg4KvEwes+Q0fb9Tes
BHpsOp6cYelMW8NGDujBWStyS1RPZD4EIjGcVP0VcY+1szUWqjkwD6rzhNgoE6OD
EKVSrI2Tt21ykWsv4qlUjs/PY6z+oOPQaqsf1uDH2H3IaTjH49EyqhWNn42ad8Pr
nd3EVQOJvKmb/pi63ac/QG0jss2D+uciTwJFwx5AKaZRnYnbemqapZNrRfGebGU8
akHvs4wuTGMj1osYkEe/XBbWxkxGPw/l37vPSVr1NJYgOGn+GF3hEWBWfsE1jkdW
6VweM610+t/OE+V1yX+H7y12lf62wd0Tfwbu8GjcBosSHsUCAwEAAaOB7jCB6zAJ
BgNVHRMEAjAAMB0GA1UdDgQWBBRgudnbLDiBaFnnNr+A+2dCwrfWjzBrBgNVHSME
ZDBigBS4mDFHd3C9I6OybWCqDXI/ETO04aE0pDIwMDEuMCwGA1UEAwwlbHVmZnku
YWxlamFuZHJvbGYuZ29uemFsb25hemFyZW5vLm9yZ4IUfBfMjVYSmJAtFQ+sCx58
zQ/XcMkwEwYDVR0lBAwwCgYIKwYBBQUHAwEwCwYDVR0PBAQDAgWgMDAGA1UdEQQp
MCeCJWx1ZmZ5LmFsZWphbmRyb2xmLmdvbnphbG9uYXphcmVuby5vcmcwDQYJKoZI
hvcNAQELBQADggEBAF7oK6FSWkATBFpZNZXheGl9PBagxZwCnIMoJzUohui7jQWP
V/oETxCRD9O0Iw1yauO8RXDvYfFb68/zsvn3ZojMy8k/bO5Q6HZjUib6Btm0z6WW
G0FUT82FPQeLIATHpOwtG27Afy8AoqBGZfWq6raQ6onSd/1IYgRg4JPCBAupK+q8
vTMUNGIIAKpfhDyeB+xWEer0HmHb3Rc7AQ9vPbv964zd9QDU5tigt1Y1N2lEAqyU
iAVDdZNOVvWj/7wObAV8y36oHnDEPTTil1qu3O9vNi42mVgnlfXX/XzQw/ZfDSJw
Om9h5ELWRAxdmdbs5biPAOx1ORfENywcHBdsROg=
-----END CERTIFICATE-----
subject=CN=luffy.alejandrolf.gonzalonazareno.org
issuer=CN=luffy.alejandrolf.gonzalonazareno.org
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1545 bytes and written 428 bytes
Verification error: unable to verify the first certificate
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 2048 bit
This TLS version forbids renegotiation.
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 21 (unable to verify the first certificate)
---
```

Como podemos ver en la última línea. nos da el error `verify error:num=21:unable to verify the first certificate`, lo que quiere decir que el cliente no puede verificar el certificado del servidor, probablemente porque el certificado no está firmado por una autoridad de certificación (CA) de confianza.

Además, el certificado en uso tiene el mismo nombre en el campo `subject` y `issuer`:

```
subject=CN=luffy.alejandrolf.gonzalonazareno.org
issuer=CN=luffy.alejandrolf.gonzalonazareno.org
```

Esto significa que el servidor está usando un certificado autofirmado, es decir, un certificado que no ha sido firmado por una **CA** externa de confianza.

Al no tener la posibilidad de ser una **CA** de confianza, ya que la hemos creado nosotros mismos, no podemos concluir y ver el resultado final de la práctica.