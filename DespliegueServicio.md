# Despliegue de servicio

**Cloud Computing: Servicios y Aplicaciones**

**Pablo García Alvarado** – <pablog.alvarado@proton.me>  
Escuela Técnica Superior de Ingeniería Informática y de Telecomunicación  
Universidad de Granada

<p align="center">
  <img src="https://github.com/user-attachments/assets/dde05572-856d-429a-b17a-f6b6e36ed2bd"
       alt="image"
       width="500" />
</p>

**29/04/2026, Granada**

---

## Índice

1. [Introducción](#introducción)
2. [Entorno de desarrollo y de producción utilizado](#entorno-de-desarrollo-y-de-producción-utilizado) 
3. [Tarea 1](#tarea-1)  
4. [Tarea 2](#tarea-2)  
5. [Tarea 3](#tarea-3)  
6. [Conclusiones](#conclusiones)  
7. [Bibliografía](#bibliografía)

---

## Introducción

El objetivo de esta práctica es adquirir experiencia en el despliegue y gestión de servicios interconectados mediante contenedores, utilizando herramientas como Podman, podman-compose y Kubernetes.

A lo largo de la práctica se pretende comprender cómo diseñar e implementar distintas arquitecturas de servicios en función de los requisitos del sistema, prestando especial atención a aspectos como la persistencia de datos, la autenticación de usuarios, la escalabilidad, la gestión de réplicas, el balanceo de carga y la monitorización.

Además, se busca trabajar con configuraciones orientadas a la alta disponibilidad y al uso concurrente por parte de múltiples usuarios, acercando el entorno de trabajo a escenarios reales de administración y despliegue de servicios en infraestructuras modernas.

A lo largo de la práctica se han trabajado distintos despliegues de servicios mediante contenedores y Kubernetes. En la primera parte se desplegó una arquitectura basada en LDAP, ownCloud y una base de datos. Posteriormente, se amplió el trabajo hacia escenarios de balanceo, escalabilidad, HAProxy y despliegue en Kubernetes mediante Minikube, utilizando recursos como pods, deployments, services y réplicas.

---

## Entorno de desarrollo y de producción utilizado

La práctica se ha desarrollado utilizando un entorno basado en contenedores, trabajando principalmente desde terminal mediante `podman` y `podman-compose`.

El entorno utilizado ha sido el siguiente:

- Sistema operativo local: Windows, utilizando terminal Git Bash para la conexión y gestión de ficheros.
- Entorno de despliegue: servidor `galeon.ugr.es` proporcionado para la práctica.
- Gestor de contenedores: Podman.
- Herramienta de composición de servicios: podman-compose.


---



## Tarea 1

El objetivo de esta primera tarea ha sido desplegar una arquitectura de servicios interconectados mediante contenedores usando `podman` y `podman-compose`. En concreto, se ha configurado un entorno formado por ownCloud, una base de datos MariaDB, un servidor LDAP y un servicio Redis.

La finalidad principal era disponer de una instancia funcional de ownCloud accesible desde navegador, con persistencia de datos y autenticación de usuarios mediante LDAP. De esta forma, se trabaja una arquitectura similar a la que podría encontrarse en un entorno real, donde la aplicación principal no gestiona todos los elementos por sí sola, sino que depende de servicios externos especializados.

Los servicios desplegados en esta tarea han sido:

- `pablo_owncloud`: aplicación web ownCloud.
- `pablo_mariadb`: base de datos MariaDB para ownCloud.
- `pablo_redis`: servicio Redis usado como apoyo por ownCloud.
- `pablo_ldap`: servidor LDAP para gestionar usuarios externos.

Antes de comenzar el despliegue, se comprobó el entorno de trabajo y las versiones de las herramientas disponibles. Para ello se utilizaron comandos básicos de identificación del sistema y versiones:

```bash
hostname
whoami
pwd
podman --version
podman compose version
```

<img width="485" height="146" alt="image" src="https://github.com/user-attachments/assets/57fa6eb8-c69e-44b3-a03e-e43d753313fd" />

A continuación, se creó una estructura de directorios para organizar los ficheros de configuración, scripts, capturas y datos persistentes de los servicios:

```bash
mkdir -p practica-owncloud/{ldap,scripts,capturas,data/owncloud,data/mariadb,data/ldap/database,data/ldap/config}
cd practica-owncloud
```

Se decidió separar los datos de cada servicio en subdirectorios dentro de `data/`, de forma que la información persistente no quedara mezclada con los ficheros de configuración.

El despliegue se definió mediante un archivo podman-compose.yml. Para editarlo se hizo uso de nano. 

```yaml
version: "3.8"

services:
  mariadb:
    image: docker.io/library/mariadb:10.11
    container_name: pablo_mariadb
    environment:
      MARIADB_ROOT_PASSWORD: rootpass
      MARIADB_DATABASE: owncloud
      MARIADB_USER: owncloud
      MARIADB_PASSWORD: owncloudpass
    volumes:
      - ./data/mariadb:/var/lib/mysql:Z
    networks:
      - owncloud-net

  redis:
    image: docker.io/library/redis:7
    container_name: pablo_redis
    command: ["redis-server", "--appendonly", "yes"]
    networks:
      - owncloud-net

  ldap:
    image: docker.io/osixia/openldap:1.5.0
    container_name: pablo_ldap
    environment:
      LDAP_ORGANISATION: "Example Inc."
      LDAP_DOMAIN: "example.org"
      LDAP_ADMIN_PASSWORD: "admin"
    ports:
      - "20111:389"
      - "20112:636"
    volumes:
      - ./data/ldap/database:/var/lib/ldap:Z
      - ./data/ldap/config:/etc/ldap/slapd.d:Z
    networks:
      - owncloud-net

  owncloud:
    image: docker.io/owncloud/server:10.15
    container_name: pablo_owncloud
    depends_on:
      - mariadb
      - redis
      - ldap
    ports:
      - "20110:8080"
    environment:
      OWNCLOUD_DOMAIN: "galeon:20110"
      OWNCLOUD_TRUSTED_DOMAINS: "galeon,localhost,127.0.0.1"
      OWNCLOUD_DB_TYPE: "mysql"
      OWNCLOUD_DB_NAME: "owncloud"
      OWNCLOUD_DB_USERNAME: "owncloud"
      OWNCLOUD_DB_PASSWORD: "owncloudpass"
      OWNCLOUD_DB_HOST: "mariadb"
      OWNCLOUD_ADMIN_USERNAME: "admin"
      OWNCLOUD_ADMIN_PASSWORD: "admin"
      OWNCLOUD_REDIS_ENABLED: "true"
      OWNCLOUD_REDIS_HOST: "redis"
    volumes:
      - ./data/owncloud:/mnt/data:Z
    networks:
      - owncloud-net

networks:
  owncloud-net:
```

En esta configuración se definió una red interna llamada `owncloud-net`, compartida por todos los servicios. Esta decisión permite que los contenedores se comuniquen entre sí usando sus nombres de servicio, como `mariadb`, `redis` o `ldap`, sin necesidad de exponer todos los puertos al exterior.

Solo se publicaron los puertos necesarios para la prueba desde el navegador o desde el exterior del entorno: el puerto `20110` para acceder a ownCloud, el `20111` para LDAP y el `20112` para LDAPS. MariaDB y Redis no se publicaron hacia fuera porque únicamente deben ser utilizados por los servicios internos de la arquitectura.

Durante los primeros intentos de despliegue, algunos contenedores no se iniciaron correctamente. Para diagnosticar el problema se consultaron los logs de MariaDB, LDAP y ownCloud mediante los siguientes comandos:

```bash
podman logs pablo_mariadb --tail=80
podman logs pablo_ldap --tail=80
podman logs pablo_owncloud --tail=80
```

La revisión de los logs permitió detectar que el problema estaba relacionado con los permisos de los directorios utilizados como volúmenes persistentes. Al trabajar con Podman, los procesos que se ejecutan dentro de los contenedores utilizan usuarios internos concretos. Por tanto, si los directorios del sistema anfitrión no tienen los permisos adecuados, servicios como MariaDB, OpenLDAP u ownCloud no pueden escribir correctamente sus datos.

Para solucionarlo se detuvieron los contenedores y se ajustó la propiedad de los directorios persistentes usando `podman unshare chown`:

```bash
podman-compose down

podman unshare chown -R 999:999 ./data/mariadb
podman unshare chown -R 911:911 ./data/ldap/database ./data/ldap/config
podman unshare chown -R 33:33 ./data/owncloud

podman-compose up -d
```

Se volvió a comprobar que los servicios se hubieran terminado de levantar correctamente con `podman ps` y `podman ps -a`.

<img width="1491" height="160" alt="p1_cc_2" src="https://github.com/user-attachments/assets/1d3a85c5-aabe-4088-b2b1-03aa8ef08726" />

Una vez desplegado el contenedor LDAP, se comprobó que el servicio respondía correctamente mediante una búsqueda LDAP ejecutada dentro del propio contenedor:

```bash
podman exec pablo_ldap ldapsearch -x \
  -H ldap://localhost:389 \
  -b dc=example,dc=org \
  -D "cn=admin,dc=example,dc=org" \
  -w admin
```

Este comando permite consultar el árbol LDAP usando autenticación simple. La opción -H indica la dirección del servidor LDAP, -b establece la base de búsqueda, -D indica el usuario con el que se realiza la consulta y -w proporciona la contraseña.

A continuación, se creó una unidad organizativa llamada People, destinada a agrupar los usuarios de la organización. Para ello se creó un fichero LDIF:

```bash
nano ldap/add_ou_people.ldif
```

El contenido del fichero fue el siguiente:

```ldif
dn: ou=People,dc=example,dc=org
objectClass: top
objectClass: organizationalUnit
ou: People
description: Unidad organizativa para usuarios de la empresa
```

Posteriormente, el fichero se copió al contenedor LDAP:

```bash
podman cp ldap/add_ou_people.ldif pablo_ldap:/tmp/add_ou_people.ldif
```

Y se añadió la unidad organizativa al directorio mediante `ldapadd`:

```bash
podman exec pablo_ldap ldapadd -x \
  -D "cn=admin,dc=example,dc=org" \
  -w admin \
  -f /tmp/add_ou_people.ldif
```

La decisión de crear ou=People se tomó para mantener una estructura LDAP ordenada. En lugar de añadir los usuarios directamente bajo dc=example,dc=org, se agrupan dentro de una unidad organizativa específica, lo cual facilita su búsqueda, filtrado y posterior integración con ownCloud.

Una vez creada la unidad organizativa, se añadieron dos usuarios al directorio LDAP: pablo y lorena. Para ello se prepararon dos ficheros LDIF:

```bash
nano ldap/pablo.ldif
nano ldap/lorena.ldif
```

Estos ficheros definían los atributos necesarios para que los usuarios pudieran ser reconocidos posteriormente por ownCloud, incluyendo atributos como uid, cn, sn, uidNumber, gidNumber, homeDirectory, loginShell y la clase de objeto inetOrgPerson.

Después, los ficheros se copiaron al contenedor LDAP:

```bash
podman cp ldap/pablo.ldif pablo_ldap:/tmp/pablo.ldif
podman cp ldap/lorena.ldif pablo_ldap:/tmp/lorena.ldif
```

Y se importaron en el directorio con ldapadd:

```bash
podman exec pablo_ldap ldapadd -x \
  -D "cn=admin,dc=example,dc=org" \
  -w admin \
  -f /tmp/pablo.ldif

podman exec pablo_ldap ldapadd -x \
  -D "cn=admin,dc=example,dc=org" \
  -w admin \
  -f /tmp/lorena.ldif
```

Una vez añadidos los usuarios, se les asignó contraseña mediante ldappasswd:

```bash
podman exec pablo_ldap ldappasswd -s pablo123 \
  -w admin \
  -D "cn=admin,dc=example,dc=org" \
  -x "uid=pablo,ou=People,dc=example,dc=org"

podman exec pablo_ldap ldappasswd -s lorena123 \
  -w admin \
  -D "cn=admin,dc=example,dc=org" \
  -x "uid=lorena,ou=People,dc=example,dc=org"
```

El uso de ficheros LDIF permite definir las entradas LDAP de forma clara y reproducible. Además, separar cada usuario en un fichero distinto facilita realizar cambios o añadir nuevos usuarios en el futuro.

Para comprobar que los usuarios se habían añadido correctamente al directorio, se realizó una búsqueda LDAP sobre la unidad organizativa `People`:

```bash
podman exec pablo_ldap ldapsearch -x \
  -H ldap://localhost:389 \
  -b ou=People,dc=example,dc=org \
  -D "cn=admin,dc=example,dc=org" \
  -w admin
```

Además, se realizó una búsqueda filtrando por la clase de objeto inetOrgPerson, ya que esta clase sería utilizada posteriormente por ownCloud para identificar qué entradas del directorio debían considerarse usuarios válidos:

```bash
podman exec pablo_ldap ldapsearch -x \
  -H ldap://localhost:389 \
  -b dc=example,dc=org \
  -D "cn=admin,dc=example,dc=org" \
  -w admin \
  "(objectClass=inetOrgPerson)" uid cn dn
```

<img width="740" height="953" alt="p1_cc_1" src="https://github.com/user-attachments/assets/2a04d056-2451-4954-9431-18447e080d25" />

El resultado mostró correctamente los usuarios pablo y lorena, por lo que se confirmó que el servidor LDAP estaba funcionando y que las entradas habían sido creadas correctamente.

Una vez comprobado que el servidor LDAP contenía los usuarios correctamente, se configuró ownCloud accediendo al servicio a través de http://galeon:20110, para utilizar LDAP como sistema externo de autenticación. Para ello se accedió con el usuario administrador de ownCloud y se habilitó/configuró la autenticación LDAP desde el apartado de administración.

Como todos los contenedores forman parte de la red interna `owncloud-net`, ownCloud puede comunicarse con el servidor LDAP usando el nombre del servicio definido en `podman-compose.yml`:

```text
ldap:389
```

Esta decisión evita depender de direcciones IP internas de contenedores, que podrían cambiar al recrear el entorno. Usar el nombre del servicio hace que la configuración sea más estable y reproducible.

En la configuración de ownCloud se comprobó que la conexión con LDAP era correcta, apareciendo el estado Configuration OK. Además, en la pestaña de usuarios se configuró el filtro para que ownCloud reconociera únicamente entradas LDAP de tipo inetOrgPerson.

objectClass=inetOrgPerson

Con este filtro, ownCloud detectó los dos usuarios creados previamente en LDAP.

También se configuró el atributo utilizado por ownCloud para permitir el inicio de sesión de los usuarios LDAP. En este caso, se utilizó el atributo `uid`, de forma que los usuarios pudieran iniciar sesión con nombres como `pablo` o `lorena`.

El filtro de login utilizado fue:

```text
(&(|(objectclass=inetOrgPerson))(|(uid=%uid)))
```

Este filtro indica que ownCloud debe buscar entradas que pertenezcan a la clase inetOrgPerson y cuyo atributo uid coincida con el nombre introducido en el inicio de sesión.

Para comprobar la configuración, se utilizó la herramienta de verificación de ownCloud introduciendo el usuario pablo. El sistema devolvió el mensaje User found and settings verified, confirmando que ownCloud era capaz de localizar correctamente al usuario en LDAP.

Tras finalizar la configuración de LDAP en ownCloud, se probó el inicio de sesión con los usuarios creados en el directorio. Se comprobó que los usuarios `pablo` y `lorena` podían acceder correctamente y que ownCloud mostraba sus perfiles con el nombre completo asociado a sus entradas LDAP.

<img width="1918" height="1033" alt="p1_cc_5" src="https://github.com/user-attachments/assets/68df90fb-2b7f-4ab0-b643-0a8469b9f84c" />

<img width="1918" height="1030" alt="p1_cc_6" src="https://github.com/user-attachments/assets/b9407477-1596-4899-b3be-512cd267301c" />

La persistencia de datos se configuró mediante volúmenes enlazados a directorios locales dentro del proyecto. En concreto, se utilizaron los siguientes montajes:

```yaml
volumes:
  - ./data/mariadb:/var/lib/mysql:Z
  - ./data/ldap/database:/var/lib/ldap:Z
  - ./data/ldap/config:/etc/ldap/slapd.d:Z
  - ./data/owncloud:/mnt/data:Z
```

Esta configuración permite que los datos principales de la arquitectura no dependan únicamente del ciclo de vida de los contenedores. Si un contenedor se detiene o se recrea, los datos almacenados en los directorios locales se mantienen.

En concreto:

MariaDB conserva la base de datos utilizada por ownCloud.
LDAP conserva tanto la base de datos del directorio como su configuración.
ownCloud conserva sus datos internos en el directorio asociado.

Durante el desarrollo de la tarea se detuvieron y levantaron los servicios en varias ocasiones mediante:

```bash
podman-compose down
podman-compose up -d
```

Tras reiniciar los contenedores, se verificó que la arquitectura seguía funcionando y que los usuarios LDAP continuaban disponibles.

Como resultado final, se obtuvo una arquitectura funcional formada por cuatro servicios desplegados mediante contenedores: `ownCloud`, `MariaDB`, `Redis` y `OpenLDAP`. El servicio `ownCloud` quedó accesible desde el navegador mediante `http://galeon:20110`, mientras que el servidor LDAP quedó integrado como sistema externo de autenticación.

La tarea permitió comprobar la comunicación entre contenedores mediante la red interna `owncloud-net`, la publicación selectiva de puertos, el uso de volúmenes persistentes y la resolución de problemas relacionados con permisos en Podman. Además, se verificó que ownCloud era capaz de detectar usuarios definidos en LDAP y permitir el inicio de sesión con ellos.

Por tanto, se cumplió el objetivo de desplegar una aplicación web compuesta por varios servicios interconectados, con persistencia de datos y autenticación centralizada.

---

## Tarea 2

El objetivo de esta segunda tarea ha sido ampliar la arquitectura desplegada en la Tarea 1 para incorporar escalabilidad horizontal y balanceo de carga. Para ello se modificó el despliegue inicial, pasando de una única instancia de ownCloud a dos instancias independientes de la aplicación, ambas conectadas a los mismos servicios de apoyo: MariaDB, Redis y LDAP.

Además, se añadió un contenedor HAProxy encargado de recibir las peticiones externas y repartirlas entre las dos instancias de ownCloud. De esta forma, el usuario no accede directamente a un contenedor concreto de ownCloud, sino al balanceador, que decide a qué backend redirigir cada petición.

Los servicios principales desplegados en esta tarea fueron:

- `pablo_owncloud1`: primera instancia de ownCloud.
- `pablo_owncloud2`: segunda instancia de ownCloud.
- `pablo_haproxy`: balanceador de carga HAProxy.
- `pablo_mariadb`: base de datos compartida por ownCloud.
- `pablo_redis`: servicio Redis compartido.
- `pablo_ldap`: servidor LDAP para autenticación de usuarios.

Antes de modificar el despliegue, se realizó una copia del archivo de composición anterior para conservar la configuración de la Tarea 1:

```bash
cp podman-compose.yml podman-compose-tarea1.yml
```

Esta decisión permite mantener una copia funcional del primer despliegue y trabajar sobre una nueva configuración sin perder la versión anterior. De este modo, si la configuración con balanceo de carga generaba problemas, era posible volver al estado anterior de la práctica.

A continuación, se creó y editó el archivo de configuración de HAProxy:

```bash
nano haproxy.cfg
```

También se editó el archivo podman-compose.yml para añadir la segunda instancia de ownCloud y el nuevo servicio HAProxy:

```bash
nano podman-compose.yml
```

La configuración resultante incorporó dos servicios ownCloud, llamados owncloud1 y owncloud2, ambos basados en la misma imagen docker.io/owncloud/server:10.15. Ambos contenedores se conectaron a la misma red interna owncloud-net y utilizaron los mismos servicios de base de datos, Redis y LDAP.

```yaml
owncloud1:
  image: docker.io/owncloud/server:10.15
  container_name: pablo_owncloud1
  depends_on:
    - mariadb
    - redis
    - ldap
  environment:
    OWNCLOUD_DOMAIN: "galeon.ugr.es:20110"
    OWNCLOUD_TRUSTED_DOMAINS: "galeon,galeon.ugr.es,localhost,127.0.0.1,docker.ugr.es"
    OWNCLOUD_DB_TYPE: "mysql"
    OWNCLOUD_DB_NAME: "owncloud"
    OWNCLOUD_DB_USERNAME: "owncloud"
    OWNCLOUD_DB_PASSWORD: "owncloudpass"
    OWNCLOUD_DB_HOST: "mariadb"
    OWNCLOUD_ADMIN_USERNAME: "admin"
    OWNCLOUD_ADMIN_PASSWORD: "admin"
    OWNCLOUD_REDIS_ENABLED: "true"
    OWNCLOUD_REDIS_HOST: "redis"
  volumes:
    - ./data/owncloud:/mnt/data:z
  networks:
    - owncloud-net

owncloud2:
  image: docker.io/owncloud/server:10.15
  container_name: pablo_owncloud2
  depends_on:
    - mariadb
    - redis
    - ldap
  environment:
    OWNCLOUD_DOMAIN: "galeon.ugr.es:20110"
    OWNCLOUD_TRUSTED_DOMAINS: "galeon,galeon.ugr.es,localhost,127.0.0.1,docker.ugr.es"
    OWNCLOUD_DB_TYPE: "mysql"
    OWNCLOUD_DB_NAME: "owncloud"
    OWNCLOUD_DB_USERNAME: "owncloud"
    OWNCLOUD_DB_PASSWORD: "owncloudpass"
    OWNCLOUD_DB_HOST: "mariadb"
    OWNCLOUD_ADMIN_USERNAME: "admin"
    OWNCLOUD_ADMIN_PASSWORD: "admin"
    OWNCLOUD_REDIS_ENABLED: "true"
    OWNCLOUD_REDIS_HOST: "redis"
  volumes:
    - ./data/owncloud:/mnt/data:z
  networks:
    - owncloud-net
```

La decisión de crear dos servicios separados de ownCloud permite simular un escenario de escalabilidad horizontal. En lugar de aumentar los recursos de una única instancia, se despliegan varias instancias equivalentes de la aplicación. Todas ellas se conectan a los mismos servicios de persistencia y autenticación, de forma que el usuario pueda acceder a la misma aplicación independientemente de qué instancia atienda la petición. Esto viene bastante bien ya que en caso de que una instacia se caiga, el servicio pueda seguir estando disponible.

Posteriormente se añadió el servicio HAProxy al archivo podman-compose.yml:

```yaml
haproxy:
  image: docker.io/haproxytech/haproxy-alpine:2.4
  container_name: pablo_haproxy
  depends_on:
    - owncloud1
    - owncloud2
  ports:
    - "20110:80"
    - "20113:8404"
  volumes:
    - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro,z
  networks:
    - owncloud-net
```

En esta configuración, HAProxy publica dos puertos:

20110:80, utilizado para acceder a ownCloud a través del balanceador.
20113:8404, utilizado para acceder a la página de estadísticas de HAProxy.

La decisión de mantener el acceso externo en el puerto 20110 permite que el usuario siga accediendo a ownCloud desde la misma dirección, pero internamente las peticiones ya no llegan directamente a un contenedor ownCloud, sino al balanceador HAProxy.

Para que HAProxy pudiera repartir las peticiones entre las dos instancias de ownCloud, se creó un archivo haproxy.cfg. En este fichero se definió un frontend que escucha las peticiones HTTP y un backend con los dos servidores ownCloud.

Un ejemplo de configuración utilizada es el siguiente:

```cfg
global
    daemon
    maxconn 256

defaults
    mode http
    timeout connect 5s
    timeout client 50s
    timeout server 50s

frontend owncloud_front
    bind *:80
    default_backend owncloud_back

backend owncloud_back
    balance roundrobin
    option httpchk GET /status.php
    server owncloud1 owncloud1:8080 check
    server owncloud2 owncloud2:8080 check

listen stats
    bind *:8404
    stats enable
    stats uri /
    stats refresh 5s
```

Con esta configuración, HAProxy escucha en el puerto 80 dentro del contenedor y redirige las peticiones hacia owncloud1:8080 y owncloud2:8080. El algoritmo roundrobin reparte las peticiones de forma alterna entre las instancias disponibles. Además, la opción httpchk permite comprobar si cada backend responde correctamente.

La página de estadísticas se expone en el puerto interno 8404, que se publicó externamente como 20113. Esta interfaz permite comprobar visualmente si los servidores backend se encuentran en estado activo.

<img width="1917" height="676" alt="p1_cc_10" src="https://github.com/user-attachments/assets/0f956cad-f4a1-4c09-b1ff-d8687ed53ed5" />

Una vez modificados los ficheros, se levantó la arquitectura completa mediante:

```bash
podman-compose up -d
```

Después se comprobó el estado de los contenedores:

```bash
podman ps
```

En esta fase debían aparecer activos los contenedores de MariaDB, Redis, LDAP, las dos instancias de ownCloud y HAProxy.

<img width="1917" height="676" alt="p1_cc_10" src="https://github.com/user-attachments/assets/d7347af3-909c-4921-87cd-b829c5b46061" />

También se comprobó el acceso HTTP al servicio mediante:

```bash
curl -I http://localhost:20110
```

Este comando permite comprobar desde terminal si el balanceador está respondiendo correctamente en el puerto publicado.

Para comprobar el comportamiento del balanceador, se realizaron pruebas deteniendo y arrancando una de las instancias de ownCloud. En primer lugar, se detuvo la primera instancia:

```bash
podman stop pablo_owncloud1
podman ps
```

Después se comprobó si HAProxy seguía pudiendo redirigir las peticiones hacia la segunda instancia disponible. Esta prueba permite verificar que la arquitectura es más tolerante a fallos que el despliegue con una sola instancia, ya que el servicio puede seguir respondiendo aunque una de las réplicas deje de estar disponible.

Posteriormente, se volvió a arrancar la instancia detenida:

```bash
podman start pablo_owncloud1
podman ps
```

<img width="1918" height="1082" alt="p1_cc_12" src="https://github.com/user-attachments/assets/647e8789-0527-4f4b-853e-1ff619e97d90" />

Como problema encontrado, cuando se paraba owncloud1, este cambio se veía reflejado dentro de la tabla de HAProxy, pero al volver arrancar esa instancia, owncloud no volvía a estar como Up y se quedaba en rojo en la tabla.

Como resultado final, se consiguió ampliar la arquitectura inicial incorporando dos instancias de ownCloud y un balanceador HAProxy. El acceso externo al servicio se realiza a través del puerto 20110, donde escucha HAProxy, que redirige las peticiones hacia owncloud1 u owncloud2.

También se habilitó una página de estadísticas de HAProxy en el puerto 20113, desde la cual se puede consultar el estado de los backends y comprobar si las instancias de ownCloud están disponibles.

Esta tarea permitió trabajar conceptos de escalabilidad horizontal, balanceo de carga, comprobación de disponibilidad y tolerancia a fallos. Además, se comprobó la importancia de configurar correctamente la red interna, los dominios de confianza de ownCloud y los health checks de HAProxy.

Con ello, la arquitectura pasó de tener una única instancia de ownCloud a disponer de dos instancias balanceadas, compartiendo los servicios de base de datos, Redis y LDAP.

---

## Tarea 3

El objetivo de esta tercera tarea ha sido desplegar servicios mediante Kubernetes, utilizando Minikube como entorno local de ejecución y Podman como driver. A diferencia de las tareas anteriores, donde el despliegue se realizó directamente con `podman` y `podman-compose`, en esta parte se ha trabajado con recursos propios de Kubernetes, como `Pods`, `Deployments`, `Services` y manifiestos YAML.

La finalidad de esta tarea era comprender cómo Kubernetes permite definir aplicaciones de forma declarativa, gestionar réplicas, exponer servicios y comprobar el estado de los recursos desplegados. De esta forma, se pasa de una arquitectura basada en contenedores individuales a una arquitectura orquestada, donde Kubernetes se encarga de mantener el estado deseado del sistema.

Para organizar los manifiestos de Kubernetes se creó una carpeta específica para esta tarea:

```bash
mkdir -p tarea3-docker-k8s
cd tarea3-docker-k8s
```

Dentro de esta carpeta se definieron los siguientes ficheros:

```bash
nano namespace.yml
nano mariadb.yml
nano redis.yml
nano ldap.yml
nano owncloud.yml
```

Cada fichero se utilizó para describir una parte de la arquitectura:

namespace.yml: define el espacio de nombres owncloud-k8s.
mariadb.yml: define los recursos necesarios para la base de datos MariaDB.
redis.yml: define el despliegue del servicio Redis.
ldap.yml: define el despliegue del servidor LDAP.
owncloud.yml: define el despliegue y exposición del servicio ownCloud.

La decisión de separar los manifiestos en varios ficheros permite mantener la configuración más ordenada y facilita aplicar, modificar o depurar cada componente de forma independiente.

Antes de aplicar los manifiestos, se comprobó la disponibilidad de `kubectl` y el estado del contexto actual:

```bash
kubectl version --client
kubectl get nodes
kubectl config current-context
kubectl config get-contexts
kubectl cluster-info
```

Posteriormente se inició Minikube usando Podman como driver. Durante las pruebas se probaron distintas opciones de arranque, incluyendo el driver docker, el driver none y la configuración rootless. Finalmente, se utilizó un perfil específico llamado pablo-k8s con Podman como driver y containerd como runtime:

```bash
minikube start -p pablo-k8s \
  --driver=podman \
  --container-runtime=containerd \
  --cpus=2 \
  --memory=3000mb \
  --disk-size=10g
```

El uso de un perfil propio permite aislar este clúster de otros posibles entornos de Minikube. Además, se asignaron recursos concretos de CPU, memoria y disco para garantizar que el entorno tuviera capacidad suficiente para ejecutar los servicios definidos.

Una vez iniciado Minikube, se comprobó el estado del nodo y el contexto activo:

```bash
kubectl get nodes
kubectl config current-context
kubectl get storageclass
```

Para desplegar la arquitectura en Kubernetes se definieron varios ficheros `.yml`, cada uno asociado a un componente concreto del sistema. Los ficheros creados fueron:

```bash
ldap.yml
mariadb.yml
namespace.yml
owncloud.yml
redis.yml
```

En primer lugar, se definió un namespace propio para la práctica:
Namespace
```yml
apiVersion: v1
kind: Namespace
metadata:
  name: owncloud-k8s
```

LDAP
El fichero `ldap.yml` define un `Deployment` y un `Service` para OpenLDAP. El `Deployment` utiliza la imagen `docker.io/osixia/openldap:1.5.0` y configura las variables de entorno necesarias para inicializar el directorio LDAP:

```yaml
[...]
env:
  - name: LDAP_ORGANISATION
    value: "Example Inc."
  - name: LDAP_DOMAIN
    value: "example.org"
  - name: LDAP_ADMIN_PASSWORD
    value: "admin"
  - name: LDAP_TLS
    value: "false"
[...]
```

Además, se expone el puerto 389, correspondiente al servicio LDAP sin TLS. La decisión de desactivar TLS mediante LDAP_TLS=false se tomó para simplificar el despliegue dentro del entorno de prácticas y evitar problemas adicionales de certificados durante las pruebas.

El Service asociado permite que otros pods del namespace, como ownCloud, puedan comunicarse con LDAP usando el nombre del servicio.

MariaDB
El fichero `mariadb.yml` define los recursos necesarios para la base de datos. En primer lugar, se creó un `PersistentVolumeClaim` llamado `mariadb-pvc` con una capacidad solicitada de `1Gi`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb-pvc
  namespace: owncloud-k8s
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
[...]
```

Este volumen persistente permite que los datos de MariaDB no dependan exclusivamente del ciclo de vida del pod. Si el pod se elimina y Kubernetes lo vuelve a crear, la base de datos puede conservar sus datos en el volumen asociado.

Después se definió un Deployment con la imagen docker.io/library/mariadb:10.11, configurando las variables de entorno necesarias para crear la base de datos de ownCloud:

```yml
[...]
env:
  - name: MARIADB_ROOT_PASSWORD
    value: rootpass
  - name: MARIADB_DATABASE
    value: owncloud
  - name: MARIADB_USER
    value: owncloud
  - name: MARIADB_PASSWORD
    value: owncloudpass
[...]
```

El contenedor expone el puerto 3306 y monta el volumen persistente en /var/lib/mysql. También se definió un Service para que ownCloud pueda conectarse a la base de datos usando el nombre mariadb.

Redis
El fichero `redis.yml` define un `Deployment` y un `Service` para Redis. Se utilizó la imagen `docker.io/library/redis:7` y se expuso el puerto `6379`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: owncloud-k8s
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
[...]
```

Redis se utiliza como servicio auxiliar para ownCloud, al igual que en las tareas anteriores. El Service permite que ownCloud pueda conectarse a Redis usando el nombre redis dentro del namespace.

Owncloud
El fichero `owncloud.yml` define el despliegue principal de la aplicación ownCloud. Al igual que MariaDB, se creó un `PersistentVolumeClaim`, en este caso llamado `owncloud-pvc`, con una capacidad de `2Gi`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: owncloud-pvc
  namespace: owncloud-k8s
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

Este volumen se monta en /mnt/data, que es el directorio utilizado por ownCloud para almacenar sus datos.

El Deployment utiliza la imagen docker.io/owncloud/server:10.15 y define las variables necesarias para conectar ownCloud con MariaDB y Redis:

```yml
[...]
env:
  - name: OWNCLOUD_DOMAIN
    value: "localhost:20117"
  - name: OWNCLOUD_TRUSTED_DOMAINS
    value: "localhost"
  - name: OWNCLOUD_DB_TYPE
    value: "mysql"
  - name: OWNCLOUD_DB_NAME
    value: "owncloud"
  - name: OWNCLOUD_DB_USERNAME
    value: "owncloud"
  - name: OWNCLOUD_DB_PASSWORD
    value: "owncloudpass"
  - name: OWNCLOUD_DB_HOST
    value: "mariadb"
  - name: OWNCLOUD_REDIS_ENABLED
    value: "true"
  - name: OWNCLOUD_REDIS_HOST
    value: "redis"
[...]
```

La variable OWNCLOUD_DB_HOST se configuró como mariadb, ya que Kubernetes permite resolver los servicios internos por su nombre. Del mismo modo, Redis se referencia mediante el nombre redis.

Finalmente, se definió un Service para ownCloud, exponiendo el puerto 8080, que posteriormente se redirigió al exterior mediante kubectl port-forward.

Una vez definidos los ficheros anteriores, se aplicaron al clúster en el siguiente orden:

```bash
kubectl apply -f namespace.yml
kubectl apply -f mariadb.yml
kubectl apply -f redis.yml
kubectl apply -f ldap.yml
kubectl apply -f owncloud.yml
```

El orden seguido tiene sentido porque primero se crea el namespace donde se alojarán todos los recursos. Después se despliegan los servicios de apoyo, como MariaDB, Redis y LDAP, y finalmente ownCloud, que depende de ellos para funcionar correctamente.

Para comprobar el estado de los recursos desplegados se utilizaron:

```bash
kubectl get pods -n owncloud-k8s
kubectl get svc -n owncloud-k8s
kubectl get pvc -n owncloud-k8s
```

Uno de los problemas encontrados estuvo relacionado con el manifiesto `ldap.yml`. Durante las pruebas fue necesario editarlo varias veces y recrear tanto el deployment como el service de LDAP:

```bash
nano ldap.yml
kubectl delete deployment ldap -n owncloud-k8s
kubectl delete service ldap -n owncloud-k8s
kubectl apply -f ldap.yml
```

El problema se resolvió revisando los nombres usados en el manifiesto. Finalmente, el deployment se definió con el nombre openldap y la etiqueta app: openldap, asegurando que el Service seleccionara correctamente los pods correspondientes mediante el selector adecuado.

Una vez desplegados los recursos en Kubernetes, se utilizó `kubectl port-forward` para exponer temporalmente el servicio `owncloud` fuera del clúster. Para ello se redirigió el puerto `20117` del servidor al puerto `8080` del servicio de ownCloud dentro del namespace `owncloud-k8s`:

```bash
kubectl port-forward --address 0.0.0.0 \
  -n owncloud-k8s \
  service/owncloud \
  20117:8080
```

El parámetro --address 0.0.0.0 permite que el puerto redirigido escuche en todas las interfaces de red, facilitando el acceso desde el navegador usando la dirección del servidor. Tras ejecutar el comando, se observó que Kubernetes aceptaba conexiones entrantes al puerto 20117, como indica la salida Handling connection for 20117.

<img width="1093" height="200" alt="p1_cc_15" src="https://github.com/user-attachments/assets/b95b3e66-fa43-487f-991f-ddd8a893620b" />

Es importante señalar que, en esta tarea, el objetivo principal fue trasladar los servicios principales a un entorno orquestado con Kubernetes y trabajar con recursos como `Deployments`, `Services`, `PersistentVolumeClaims` y `port-forward`. La integración completa de LDAP dentro de ownCloud se realizó y verificó principalmente en el despliegue con `podman-compose` de la Tarea 1. En Kubernetes se priorizó la definición declarativa de los servicios, la persistencia y la exposición de ownCloud, por lo que este despliegue debe entenderse como una adaptación progresiva de la arquitectura al entorno Kubernetes.

Como conclusión de la Tarea 3, se consiguió trasladar la arquitectura de servicios a un entorno Kubernetes utilizando Minikube y manifiestos `.yml`. Se desplegaron los componentes principales dentro del namespace `owncloud-k8s`, incluyendo MariaDB, Redis, OpenLDAP y ownCloud, trabajando con recursos como `Deployments`, `Services` y `PersistentVolumeClaims`. Además, se comprobó el estado del clúster y de los recursos mediante comandos como `kubectl get pods`, `kubectl get svc`, `kubectl get pvc`, `kubectl logs` y `kubectl describe`.

Esta tarea permitió comprender mejor la diferencia entre gestionar contenedores directamente con `podman-compose` y desplegarlos mediante un orquestador como Kubernetes. Mientras que en las tareas anteriores los servicios se definían en un único archivo de composición, en Kubernetes cada componente se describe mediante recursos independientes y el propio clúster se encarga de mantener el estado deseado. También se trabajó la exposición del servicio mediante `kubectl port-forward`, comprobando que ownCloud recibía peticiones externas. En conjunto, la tarea sirvió para familiarizarse con el despliegue declarativo, la organización mediante namespaces, la persistencia en Kubernetes y la depuración de errores en pods y servicios.

---

## Conclusiones

En esta práctica he aprendido a desplegar y gestionar servicios interconectados mediante contenedores, empezando con `podman-compose` y avanzando después hacia un entorno orquestado con Kubernetes. A lo largo del trabajo he podido ver de primera mano (y no solo con lo que ya sabía de teoría) cómo una aplicación real no depende de un único contenedor, sino de varios servicios que colaboran entre sí, como una base de datos, un sistema de autenticación, un servicio auxiliar de caché y un balanceador de carga.

También he comprendido mejor la importancia de la persistencia de datos, la configuración de redes internas y la publicación controlada de puertos. Los problemas encontrados, especialmente los relacionados con permisos en volúmenes, configuración de HAProxy y manifiestos de Kubernetes, me han ayudado a entender mejor cómo diagnosticar errores usando comandos como `podman logs`, `podman ps`, `kubectl get`, `kubectl logs` y `kubectl describe`.

Lo que más me ha gustado de la práctica ha sido ver cómo los servicios iban funcionando progresivamente: primero ownCloud con LDAP, después dos instancias balanceadas con HAProxy y finalmente el despliegue en Kubernetes. Me ha parecido especialmente interesante comprobar cómo herramientas como HAProxy y Kubernetes permiten acercar el despliegue a escenarios más reales, donde se busca escalabilidad, tolerancia a fallos y una gestión más ordenada de los servicios.

En general, la práctica me ha resultado útil porque combina administración de sistemas, redes, contenedores y despliegue de aplicaciones. Además, me ha permitido entender mejor la lógica detrás de las arquitecturas modernas basadas en servicios y la importancia de documentar bien tanto las decisiones tomadas como los problemas encontrados durante el proceso.

---

## Bibliografía

1. Guion de la práctica 1 de Cloud Computing: Servicios y Aplicaciones, última vez consultado el 29 de abril de 2026 en https://github.com/j-m-benitez/cc2526/blob/main/practice1/README.md#entrega-de-la-pr%C3%A1ctica-a-traves-de-prado-documentaci%C3%B3n-y-evaluaci%C3%B3n-de-la-pr%C3%A1ctica

2. Documentación oficial de Podman, última vez consultada el 29 de abril de 2026 en https://podman.io/docs

3. Repositorio oficial de podman-compose, última vez consultado el 29 de abril de 2026 en https://github.com/containers/podman-compose

4. Documentación oficial de ownCloud Server, última vez consultada el 29 de abril de 2026 en https://doc.owncloud.com/server/

5. Documentación de ownCloud sobre autenticación de usuarios mediante LDAP, última vez consultada el 29 de abril de 2026 en https://doc.owncloud.com/server/next/admin_manual/configuration/user/user_auth_ldap.html

6. Documentación oficial de OpenLDAP, última vez consultada el 29 de abril de 2026 en https://www.openldap.org/doc/

7. Documentación oficial de MariaDB Server, última vez consultada el 29 de abril de 2026 en https://mariadb.com/kb/en/documentation/

8. Documentación oficial de Redis, última vez consultada el 29 de abril de 2026 en https://redis.io/docs/latest/

9. Documentación oficial de HAProxy, última vez consultada el 29 de abril de 2026 en https://www.haproxy.com/documentation/

10. Documentación oficial de Kubernetes, última vez consultada el 29 de abril de 2026 en https://kubernetes.io/docs/

11. Documentación oficial de Minikube, última vez consultada el 29 de abril de 2026 en https://minikube.sigs.k8s.io/docs/

12. Documentación de Kubernetes sobre Deployments, última vez consultada el 29 de abril de 2026 en https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

13. Documentación de Kubernetes sobre Services, Load Balancing and Networking, última vez consultada el 29 de abril de 2026 en https://kubernetes.io/docs/concepts/services-networking/service/
