
```
Curso       : 202021
Área        : Sistemas operativos, Servicio de Directorio LDAP, Autenticación
Descripción : Configurar autenticación a través del servicio de directorio 389-DS
Requisitos  : Partimos de 389-DS en OpenSUSE
Tiempo      :
```

# Cliente para autenticación LDAP

Con autenticacion LDAP prentendemos usar la máquina servidor LDAP, como repositorio centralizado de la información de grupos, usuarios, claves, etc. Desde otras máquinas conseguiremos autenticarnos (entrar al sistema) con los usuarios definidos no en la máquina local, sino en la máquina remota con LDAP. Una especie de *Domain Controller*.

En esta actividad, vamos a configurar otra MV (GNU/Linux OpenSUSE) para que podamos hacer autenticación en ella, pero usando los usuarios y grupos definidos en el servidor de directorios LDAP de la MV1.

# 1. Preparativos

* Supondremos que tenemos una MV1 (serverXX) con DS-389 instalado, y con varios usuarios dentro del DS.
* Necesitamos MV2 con SO OpenSUSE ([Configuración MV](../../global/configuracion/opensuse.md))

Comprobamos el acceso al LDAP desde el cliente:
* Ir a MV cliente.
* `nmap -Pn IP-LDAP-SERVERXX | grep -P '389|636'`, para comprobar que el servidor LDAP es accesible desde la MV2 cliente. En caso contrario revisar que el cortafuegos está abierto en el servidor y que el servicio está activo.
* `ldapsearch -H ldap://IP-LDAP-SERVERXX:389 -W -D "cn=Directory Manager" -b "dc=ldapXX,dc=curso2021" "(uid=*)" | grep dn`, comprobamos que los usuarios del LDAP remoto son visibles en el cliente.

# 2. Configurar autenticación LDAP

> Enlaces de interés:
> * https://doc.opensuse.org/documentation/leap/archive/15.3/security/html/book-security/cha-security-ldap.html
> * [Configurar_servidor_de_autenticacion_usando_YaST](https://es.opensuse.org/Configurar_servidor_de_autenticacion_usando_YaST)
> * https://luiszambrana.com.ar/2020/12/09/gestion-de-usuarios-con-openldap/?s=09

## 2.1 Crear conexión con servidor

Vamos a configurar de la conexión del cliente con el servidor LDAP.

* Ir a la MV cliente.
* No aseguramos de tener bien el nombre del equipo y nombre de dominio (`/etc/hostname`, `/etc/hosts`)
* Ir a `Yast -> LDAP y Kerberos`. En el caso de que no nos aparezca esta herramienta, la podemos instalar con el paquete `yast2-auth-client`.
* Configurar como la imagen de ejemplo:
    * BaseDN: `dc=ldapXX,dc=curso2021`
    * DN de usuario: `cn=Directory Manager`
    * Contraseña: CLAVE del usuario cn=Directory Manager

![opensuse422-ldap-client-conf.png](./images/opensuse422-ldap-client-conf.png)

* Pulsar el botón para `Probar conexión`.
* Aceptar.

## 2.2 Comprobar con comandos

Ir a la MV cliente:
* Vamos a la consola.
* `id mazinger`, consultar información del usuario.
* `getent passwd mazinger`, consultamos más datos del usuario
* `cat /etc/passwd | grep mazinger`, nos aseguramos que el usuario NO es local.
* `su -l mazinger`, entrar con el usuario definido en LDAP.

# 3. Crear usuarios usando otros comandos

> Para que funcionen bien los siguientes comandos el fichero /root/.dsrc
debe estar correctamente configurado.

Ir a la MV del servidor:
* `dsidm localhost user list`, consultar la lista de usuarios.
* Crear usuario robot1:
```
dsidm localhost user create --uid robot1 \
   --cn robot1 --displayName 'robot1' --uidNumber 2101 --gidNumber 100 \
  --homeDirectory /home/robot1
```
* Poner la clave al usuario:
```
dsidm localhost account reset_password \
  uid=robot1,ou=people,dc=ldapXX,dc=curso2122
```
* `dsidm localhost user list`, consultar la lista de usuarios.

Ir a la MV cliente:
* Abrir terminal con nuestro usuario normal (NO usar root).
* `su robot1`, entrar como ese usuario.

# 4. Usando Yast

## 4.1 Crear usuario LDAP usando Yast

En este punto vamos a escribir información dentro del servidor de directorios LDAP.
Este proceso se debe poder realizar tanto desde el Yast del servidor, como desde el Yast
del cliente.

* Ir a la MV cliente.
* `Yast -> Usuarios Grupos`.
* Set filter: `LDAP users`.
* Bind DN: `cn=Directory Manager,dc=ldapXX,dc=curso2021`.
* Crear el grupo `villanos` (Estos se crearán dentro de la `ou=groups`).
* Crear usuario `baron` (Se creará dentro de la `ou=people`).
* Incluir los usuarios `robot` y `drinfierno` en el grupo de `villanos`.

## 4.2 Comprobamos

* Ir a la MV cliente.
* `ldapsearch -H ldap://IP-LDAP-SERVER -W -D "cn=Directory Manager" -b "dc=ldapXX,dc=curso2021" "(uid=NOMBRE-DEL-USUARIO)"` comando para consultar en la base de datos LDAP la información del usuario con uid concreto.
* Iniciar sesión de entorno gráfico con algún usuario LDAP.


# IDEAS para le próximo curso
Squid con soporte ldap - openSUSE Wiki
https://es.opensuse.org/Squid_con_soporte_ldap

---
# ANEXO

Hasta ahora hemos usado el protocolo no seguro LDAP, ahora vamos a implementar
la conexión segura LDAPS.

## Generar Certificados

* https://www.linuxito.com/seguridad/598-como-crear-un-certificado-ssl-autofirmado-en-dos-simples-pasos
* https://www.adictosaltrabajo.com/2003/08/07/iisssl/
* https://www.linuxito.com/gnu-linux/nivel-alto/994-como-implementar-ldap-sobre-ssl-tls-con-openldap

Opción 1:
* Crear certificado autofirmado: `openssl req -newkey rsa:1024 -x509 -nodes -out server.pem -keyout server.pem -days 365`.
* Export firma PKCS12: `openssl pkcs12 -export -out server.p12 -in server.pem`.

Opción 2:
* `openssl genrsa -des3 -out server.key 1024`
* `openssl rsa -in server.key -out server.pem`
* `openssl req -new -key server.key -out server.csr`
* `openssl x509 -req -days 360 -in server.csr -signkey server.key -out server.crt`

## [PENDIENTE] de completar la información sobre los certificados.

Esto no funciona, pero es un intento de crear certificado y firma para LDAPS.

* Crear certificado autofirmado: `openssl req -newkey rsa:1024 -x509 -nodes -out server.pem -keyout server.pem - days 265`.
* Export firma PKCS12: `openssl pkcs12 -export -out server.pfx -in server.pem`.
