
## Materia: ST0263-251
Integrantes del proyecto:

* Delvin Rodríguez Jiménez - djrodriguj@eafit.edu.co
* Wendy Benítez Gómez - wdbenitezg@eafit.edu.co
* Fredy Cadavid Franco - fcadavidf@eafit.edu.co

Profesor

Edwin Nelson Montoya Múnera - emontoya@eafit.edu.co


# Despliegue en AWS

# 1. Breve descripción de la actividad

## 1.1 **Aspectos desarrollados**
Durante el desarrollo del proyecto se cumplió con éxito la implementación de los Objetivos 1 y 2, enfocados en el despliegue en producción, escalabilidad, seguridad y disponibilidad del sistema BookStore en la nube. A continuación, se detallaremos los logros alcanzados:
#### Despliegue en Producción en AWS
> - La aplicación monolítica BookStore fue desplegada exitosamente en una máquina virtual en AWS.
> - Se configuró un subdominio para la aplicación: [https://bookstore-alp.freeddns.org/](https://bookstore-alp.freeddns.org/ "https://bookstore-alp.freeddns.org/") 
> - Se implementó un certificado SSL usando ZeroSSL.
> - Se utilizó NGINX como proxy inverso, permitiendo separar el tráfico seguro y servir contenido estático.
> - Se empleó docker-compose para levantar la aplicación y base de datos en contenedores separados (Flask + MySQL)

#### Escalamiento en Nube
Se implementaron dos enfoques distintos de escalamiento horizontal:
Opción 1: Escalamiento con Máquinas Virtuales (Auto Scaling Group)
> - Se configuró un Auto Scaling Group (ASG) con un mínimo de 2 y un máximo de 3 instancias EC2.
> - Se integró un Load Balancer que distribuye el tráfico entre las instancias activas.
> - Se estableció una política de escalado dinámico basada en carga (tráfico).
> - La base de datos fue separada usando Amazon RDS, proporcionando persistencia y disponibilidad.
> - Se usó Amazon EFS como sistema de archivos compartido para servir archivos estáticos en todas las instancias.

Opción 2: Escalamiento con Contenedores (Docker Swarm)
> - Se realizó una versión del despliegue usando Docker Swarm con 5 réplicas de la app.
> - Se configuró un Load Balancer dedicado al clúster de contenedores, aislado del ASG.
> - El tráfico fue gestionado de forma segura (HTTPS) y balanceado entre los contenedores disponibles.
> 
## 1.2 **Aspectos no desarrollados**
No se cumplió con el Objetivo 3 (inicialmente) propuesto por el docente, el cual consistía en:

> Objetivo 3 (50%): Realizar una reingeniería de la app BookStore Monolítica, para dividirla en tres microservicios independientes:
> Microservicio 1: Autenticación (registro, login, logout)
> Microservicio 2: Catálogo (visualización de la oferta de libros)
> Microservicio 3: Compra, Pago y Entrega de libros

En lugar de migrar hacia microservicios, se optó por realizar un despliegue en contenedores usando Docker Swarm, con los siguientes beneficios:

> - Escalamiento horizontal del servicio completo por réplicas
> - Balanceo de carga entre contenedores
> - Persistencia de datos mediante Amazon RDS y EFS
> - Separación del tráfico HTTP/HTTPS con un NGINX como reverse proxy

Esto permitió simular un entorno distribuido en producción, aunque la lógica de negocio permanece centralizada en un solo contenedor por réplica (es decir, sigue siendo una app monolítica escalada, no una arquitectura basada en microservicios).

# 2. Diseño y Arquitectura
El diseño de las aplicaciones y su arquitectura se realizaron en 3 tipos de despliegue distintos:
## **1. Arquitectura Monolítica Simple**
![Monolith](https://i.imgur.com/WDgJKx3.png)
Esta arquitectura presenta el diseño más simple de implementación, puesto que todo se ejecuta en la misma máquina. Sin embargo, no es apta para entornos de alta demanda o de demanda variable, debido a los riesgos de saturación de la máquina y la disponibilidad de la misma. Algunas características de esta arquitectura:

-   **Web**: Cliente accede directamente a la aplicación.
-   **Bookstore**: Aplicación Flask y base de datos MySQL corriendo en una misma unidad (posiblemente un solo contenedor o servidor) con NGINX como proxy inverso.
-   **Limitaciones**: No hay alta disponibilidad (HA), ni balanceo de carga. Un único punto de falla.

## **2. Arquitectura Escalable en EC2 con Auto Scaling**
![ASG](https://i.imgur.com/4mDc9Kh.png)
Esta arquitectura introduce el escalamiento horizontal de la aplicación monolítica, instanciando más máquinas de forma dinámica según la carga del servicio. Aquí se implementan balanceadores de carga junto a bases de datos administradas para alta disponibilidad y redundancia en varias zonas de disponibilidad (cuando aplique). Características a tener en cuenta:
-   **ASG (Auto Scaling Group)**: Se despliegan múltiples instancias de EC2 que contienen:
    -   Nginx como reverse proxy.
    -   Contenedor de la app Flask (`Bookstore`).
-   **AWS RDS**: La base de datos está gestionada externamente por Amazon.
-   **Load Balancer**: Balancea tráfico entre las instancias.
-   **Aspectos**:
    -   Escalabilidad automática
    -   Separación de la base de datos.
    -   Redundancia entre instancias.
    

----------

## **3. Arquitectura con Docker Swarm**
![Swarm](https://i.imgur.com/F0Ls3Jk.png)

Arquitectura similar a la anterior, pero se usa Docker Swarm para la orquestación de los nodos en un clúster, con conceptos como replicación, redundancia y alta disponibilidad, además de algoritmos de elección de lider entre nodos, añadiendo el aspecto de tolerancia a fallos.
 
-   **Swarm Cluster**: Compuesto por 3 nodos **Managers** y 2 **Workers**.
-   **Cada nodo ejecuta Nginx + App Flask** (contenedorizados).
-   **Load Balancer externo**: Distribuye las peticiones entrantes entre los nodos del Swarm.
-   **AWS RDS**: Base de datos desacoplada del clúster.
-   **Aspectos**:
    -   Alta disponibilidad (3 managers).
    -   Swarm gestiona la orquestación y failover.
    -   Auto-reemplazo de servicios caídos.


# 3. Descripción del ambiente de desarrollo y técnico
#### Lenguajes y herramientas
Para este proyecto, se utilizó lo siguiente:
 - docker@**28.1.1**
 - docker-compose@**1.29.2**
 - flask@**latest**
 - flask_sqlalchemy@**latest**
 - flask_login@**latest**
 - pymysql@**latest**
 - werkzeug@**latest**
 - nginx
### Versión monolítica con base de datos
Para la versión monolítica donde la base de datos reside en la misma máquina, se utilizó el proyecto con el docker compose de [*este repositorio*](https://github.com/st0263eafit/st0263-251/tree/main/proyecto2). Para ejecutar esta versión, basta con instalar `docker` y `docker-compose`. Luego, dentro del directorio raíz del proyecto, ejecutar el siguiente comando:

    docker compose up -d
Esto ejecutará el servidor de `flask` en un contenedor, seguido del servidor MySQL en otro.

Para exponer el servidor mediante un proxy inverso como `nginx`, se debe instalar el servicio en la instancia:

    sudo apt install nginx -y

Para verificar el estado del servicio:

    sudo systemctl status nginx

Luego, para configurar la ruta donde debe apuntar, se tiene la siguiente configuración en el archivo default en la ruta `/etc/nginx/sites-available/default`:
```nginx
server {
	listen 80;
	server_name _;

	location / {
		proxy_pass http://localhost:5000;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	}
}
```
### Versión monolítica en varias VM con base de datos remota
Para esta versión se modifica el archivo `docker-compose` para solo alojar la aplicación flask en un contenedor de la siguiente forma:
```docker-compose
version: '3.8'

services:
	flaskapp:
		build: .
		environment:
		- FLASK_ENV=development
		- MYSQL_DATABASE=bookstore
		- MYSQL_USER=admin
		- MYSQL_PASSWORD=bookstore
		- MYSQL_ADDRESS=db_url
		ports:
		- "5000":"5000"
		networks:
		- bookstore_net
networks:
	bookstore_net
```
En este caso, se elimina por completo la sección de MySQL dentro del `docker-compose` y, se establecen variables de entorno para la conexión a la base de datos (esquema, usuario, contraseña y dirección de la BD).

Además, para cada instancia, se utiliza el servicio de `nginx` el cual se configura de la misma manera que en la versión monolítica de 1 sola instancia.

### Versión monolítica con Docker Swarm
Docker Swarm permite la orquestación de contenedores, permitiendo escalar y replicar los contenedores de las aplicaciones, puntos cruciales para la tolerancia a fallos y alta disponibilidad.
Esta versión modifica el archivo `docker-compose.yml` para agregar una propiedad de `deploy` a cada servicio. Este define como se debe comportar cada contenedor en cuanto a cantidad de réplicas, y cuantas de estas por instancia, entre otros aspectos.
Por ejemplo, para el proyecto, se definió 8 réplicas de la aplicación en `flask` y 3 replicas de `nginx` con la condición de repartirse 1 máximo en cada instancia.
```docker
version: '3.8'

services:
	flaskapp:
		image: dexterx11/bookstore:latest
		environment:
		- FLASK_ENV=development
		- MYSQL_DATABASE=bookstore
		- MYSQL_USER=admin
		- MYSQL_PASSWORD=bookstore
		- MYSQL_ROOT_PASSWORD=root_pass
		- MYSQL_ADDRESS=db_url
		ports:
		- "5000"
		networks:
		- bookstore_net
		deploy:
			replicas: 8
			restart_policy:
			condition: any
	nginx:
		image: dexterx11/bookstore-nginx:latest
		deploy:
			replicas: 3
			placement:
				max_replicas_per_node: 1
				restart_policy:
				condition: any
		ports:
			- target: 80
			published: 80
			protocol: tcp
			mode: ingress
		networks:
		- bookstore_net
- 
networks:
	bookstore_net:
	driver: overlay
	attachable: true
```
Se agregó además, un servicio de `nginx` como un contenedor que reside en la misma red de las apps de `flask`. El propósito de esto último recae en poder redirigir mediante proxy inverso las peticiones que le llegue a cualquier máquina que haga parte del *swarm* a un contenedor de aplicación `flask` cualquiera. Esto es gracias al DNS interno del swarm que, al acceder mediante el nombre del servicio de la aplicación flask, devuelve una coincidencia de cualquiera de las réplicas, utilizando round-robin.

Para que el servicio de `nginx` haga parte de la red interna, se agrega la configuración de `mode: ingress` dentro de la sección `ports` (véase el `docker-compose.yml` anterior). Luego de esto, nginx podrá acceder al servicio DNS del `swarm` haciendo referencia al nombre del servicio de la aplicación estipulado en el `docker-compose`; esto se especifica en el `default.conf` de `nginx`:

```nginx
server {
	listen 80;
	server_name bookstore-alp.freeddns.org;

	location / {
		proxy_pass http://flaskapp:5000;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	}
}
```
Donde `flaskapp` es el nombre del servicio de `flask` que hace parte de la misma red interna.

Para ejecutar el docker swarm, basta con ejecutar el siguiente comando en alguna máquina para ser manager:

    docker swarm init --advertise-addr [IP MAQUINA]

Es necesario especificar la IP de la máquina en la cual corre la instancia.

Se generará un token para que otros nodos se unan como workers al swarm. Si se desea agregar más managers, dentro un nodo manager se debe ejecutar el siguiente comando:

    docker swarm join-token manager
Devolverá el token para unir un nodo como manager del swarm.

Antes de generar los servicios y sus réplicas, se debe construir la imagen con base en el `Dockerfile` de la aplicación `flask`y el servicio de `nginx`. Se puede realizar de esta manera:

    docker build -t <username>/<image>:latest
Donde `username` es tu nombre de usuario e `image` el nombre de la imagen, seguido por la versión de esta, donde se le especifica por defecto como la más reciente. El `Dockerfile` de la aplicación en `flask` se encuentra en la raíz del proyecto, mientras que la del servicio `nginx` se encuentra en la carpeta del mismo nombre. Se debe ejecutar el mismo comando para cada imagen distinta.

Con esto las imagenes se han creado en el entorno local. Luego de esto, se debe hacer referencia al mismo en el archivo `docker-compose`. Para la aplicación de `flask`, se especificaría de la siguiente forma:
```docker
version: '3.8'

services:
	flaskapp:
		image: <username>/<image>:latest
		environment:
			...
```
Por último, para generar el servicio y sus réplicas (especificadas en el docker-compose), se ejecuta el siguiente comando:

    docker stack deploy --compose-file docker-compose.yml <nombre_servicio>


# 4.  Despliegue y componentes

## 1. **Contenedores y Orquestación Local**
Inicialmente, BookStore se ejecuta localmente mediante Docker, con dos contenedores gestionados por docker-compose:
>  - Un contenedor para la aplicación **Flask**.
>  - Un contenedor con una base de datos **MySQL**.

## 2. **Despliegue en la nube con AWS**
### **Despliegue en Máquinas Virtuales**
En el entorno de producción, la aplicación es desplegada en la nube sobre la infraestructura de AWS con los siguientes componentes clave:

#### a. Auto Scaling Group (ASG)
Se configura un grupo de autoescalado para garantizar que la aplicación escale horizontalmente según la carga del tráfico. Este grupo:
> - Está conectado a un **Target Group** asociado al balanceador de carga.
> - Define un mínimo de 2 instancias y un máximo de 3 instancias EC2.
> - Permite agregar o quitar máquinas virtuales automáticamente según métricas (por ejemplo, cantidad de peticiones HTTPS.).

#### b. Load Balancer (Application Load Balancer)
Un **Load Balancer** distribuye las solicitudes entrantes entre las instancias activas en el Target Group. Sus características son:
> - Escucha en puertos HTTP (80) y HTTPS (443).
> - Redirige todo el tráfico HTTP a HTTPS, forzando una conexión segura.
> - Garantiza la disponibilidad y balanceo de carga entre las instancias del Auto Scaling Group.

#### c. Certificado SSL
El dominio propio está protegido mediante un certificado **SSL** emitido por **ZeroSSL**, lo que garantiza la seguridad en las comunicaciones cifradas mediante HTTPS. El certificado fue instalado manualmente en el servidor y gestionado a través de NGINX como proxy inverso.

#### d. Base de Datos (Amazon RDS)
> -  Administración automática (backups, actualizaciones, monitoreo).
> - Alta disponibilidad dentro de una única zona de disponibilidad (por restricciones del entorno de AWS Academy).
> - Reducción de la carga de gestión manual del motor de base de datos.

#### e. Sistema de Archivos (EFS)
Se utiliza **Amazon EFS (Elastic File System)** para compartir archivos estáticos y recursos persistentes entre todas las instancias EC2 del grupo de autoescalado:
> - Montado automáticamente en las instancias a través de una configuración NFS.
> - Ideal para compartir diferentes archivos. 

#### f. Proxy Inverso con NGINX
En cada instancia EC2, se configura **NGINX como proxy inverso**, cuya función es:
> - Interceptar solicitudes entrantes.
> - Servir contenido estático eficientemente.
> - Reenviar las solicitudes dinámicas a la aplicación Flask.
> - Hacer de punto de terminación del SSL, es decir, NGINX desencripta la conexión HTTPS antes de pasar la solicitud a Flask.

## 3. **Despliegue en contenedores**
Además del despliegue tradicional con máquinas virtuales, la aplicación **BookStore** cuenta con una versión productiva basada en **contenedores orquestados con Docker Swarm**.

#### a. Orquestación con Docker Swarm
Se implementa un clúster de **Docker Swarm** para administrar múltiples instancias de la aplicación de forma coordinada. En este entorno
> - Se despliegan **5 contenedores** de la aplicación Flask como **servicios de Docker Swarm**, distribuidos entre los nodos del clúster.
> - Swarm gestiona el enrutamiento de las solicitudes, el balanceo interno de carga y la alta disponibilidad de los contenedores.
> - Los servicios son definidos y levantados a través de un archivo docker-compose.yml compatible con Swarm.
#### b. Load Balancer dedicado para contenedores
En esta arquitectura se utiliza un **Load Balancer independiente y exclusivo para el entorno de contenedores**, separado del balanceador del despliegue con Auto Scaling Groups. Este:
> - Se encarga de recibir el tráfico entrante HTTPS.
> - Redirige las peticiones hacia los contenedores activos del clúster
> - Está configurado con reglas de redirección HTTP → HTTPS y usa el mismo certificado SSL emitido por **ZeroSSL**.

## 4. Ejecución
### Monolítico - única instancia
Para desplegar este tipo de servidor, se crea una instancia (de preferencia `t2.micro` ).
Se debe configurar un grupo de seguridad para esta máquina y exponerla al internet mediante el puerto 80 y 443:
![Security Group](https://i.imgur.com/rra69x6.png)

Dentro de la instancia, se realiza la actualización de los repositorios:

    sudo apt update
    sudo apt upgrade -y

Luego, se instala `docker`, `docker-compose`, `nginx` y `zip`.

    sudo apt install docker.io
    sudo apt install docker-compose
    sudo apt install nginx
    sudo apt install zip
 
 Luego, se clona [el repositorio del proyecto base](https://github.com/st0263eafit/st0263-251/tree/main/proyecto2)
 

    git clone https://github.com/st0263eafit/st0263-251.git

Dentro de la carpeta `proyecto2` se encuentra el .zip del proyecto. Se extrae con el siguiente comando:

    unzip BookStore.zip

Después de esto, dentro de la carpeta raiz del proyecto, se ejecuta el siguiente comando:

    docker compose up -d
Con esto, la aplicación junto con la base de datos estarán ejecutándose en sus  respectivos contenedores. Para verificar el estado de estos se hace lo siguiente:

    docker ps

Si se tiene una salida similar a esta, el servicio está funcionando correctamente:
![Docker process](https://i.imgur.com/yZ5PzMx.png)

Lo siguiente es configurar el servicio de `nginx` para que este pueda enrutar las peticiones a la aplicación `flask`.

Antes de configurarlo, es necesario tener en la instancia el certificado SSL o TLS para la aplicación. Para este proyecto, se han guardado en la siguiente ruta: `/etc/ssl`.

Con el siguiente comando se edita el archivo de configuración por defecto de `nginx`:

    sudo nano /etc/nginx/sites-available/default

Se reemplaza su contenido por el siguiente:
```nginx
server {
    listen 80;
    server_name _;

    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    server_name _;

    ssl_certificate     /etc/ssl/fullchain1.pem;
    ssl_certificate_key /etc/ssl/privkey1.pem;

    location / {
        proxy_pass http://localhost:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```
Este enruta todo el tráfico HTTP hacia HTTPS, además de proveer los certificados para realizar la conexión segura. Cabe destacar que, es necesario cambiar la ruta de los archivos del certificado por la que se tenga en la instancia.

Se guarda la configuración, y se ejecuta el siguiente comando para verificar la sintaxis:

    sudo nginx -t

Luego, se reinicia el servicio de `nginx`

    sudo systemctl restart nginx

Finalmente,  si todo se ha configurado correctamente, al entrar a la instancia mediante la dirección IP, se podrá acceder a la aplicación. El navegador dará un aviso de certificado, ya que es necesario el acceso a este mediante el dominio registrado con dicho certificado.

### Con VM mediante ASG y LB

La ejecución de este despliegue requiere la creación de una imagen base para la plantilla de lanzamiento que utilizará el grupo de escalamiento. 

Antes de realizar la imagen, es necesario crear una base de datos. Para este proyecto, se utilizó una base de datos administrada, RDS. Esta se debe crear para utilizar MySQL, con la plantilla de `production`. Se debe anotar el usuario y contraseña para esta base de datos, puesto que se usará en las variables de entorno para la aplicación. Es importante recalcar que la base de datos debe estar en el mismo grupo de seguridad que el grupo de escalamiento y balanceadores, para garantizar el acceso netamente interno.

Después de crear la base de datos, se copia el endpoint DNS de la base de datos:
![endpoint](https://i.imgur.com/OWKI6zO.png)

Ahora, como se mencionó antes, se creará una imagen de despliegue. Se utilizará la instancia creada en el despliegue anterior como base, con la diferencia de que ahora el `docker-compose.yml` se verá de la siguiente manera:
```docker-compose
version: '3.8'

services:
	flaskapp:
		build: .
		environment:
		- FLASK_ENV=development
		- MYSQL_DATABASE=bookstore
		- MYSQL_USER=admin
		- MYSQL_PASSWORD=bookstore
		- MYSQL_ADDRESS=db_url
		ports:
		- "5000":"5000"
		networks:
		- bookstore_net
networks:
	bookstore_net:
```
Esto elimina la creación de un contenedor para la bases de datos, puesto que se utilizará una administrada. se debe reemplazar las variables de entorno, donde:
 * **DATABASE** es el esquema
 * **USER** es el usuario
 * **PASSWORD** es la contraseña de acceso
 * **ADDRESS** es el endpoint de la base de datos

Después de realizar los cambios al compose dentro de la instancia. Se crea la imagen:
![Image creation](https://i.imgur.com/4XWiiKD.png)
Tardará un poco en crearse la imagen, dependiendo del tamaño del volumen de la instancia base.

Lo siguiente es crear una plantilla de lanzamiento
![Launch Template](https://i.imgur.com/GybmC8i.png)
Se debe seleccionar como imagen base la que fue creada anteriormente, en la sección de "Mis AMIs" o similar. Además, se debe corroborar que las instancias se ejecutan con el mismo grupo de seguridad que la base de datos.

Luego, se crea un Auto Scaling Group con la capacidad deseada. Para este proyecto, se utilizó mínimo 2 instancias y máximo 3:
![Auto Scaling Group](https://i.imgur.com/qVodkIM.png) 

Se debe asociar la plantilla de lanzamiento previamente creada a este ASG.

Luego de esto, se debe crear un Load Balancer de tipo Aplicación, junto con un Target Group, el cual estará vacío por ahora, puesto que ahí las máquinas se ejecutarán automáticamente.  Si se elige un listener de tipo HTTPS, AWS pedirá los archivos de certificado, los cuales se deben proveer.

El esquema general debería ser similar al siguiente (sin los targets):
![Load Balancer](https://i.imgur.com/oig9PMD.png)
El Listener de puerto 80 redirige las peticiones a HTTPS.

Por último, en el Auto Scaling Group previamente creado, se conecta una integración de tipo Load Balancer y se especifica el target group de la máquina![Target Group](https://i.imgur.com/rHuZyOf.png)

Con esto, las máquinas se ejecutarán automáticamente dentro del target group, además de escalar según las reglas de escalamiento establecidas en el ASG.

Se puede acceder a cualquiera de las instancias a partir del endpoint DNS del load balancer. El navegador nuevamente dará aviso del certificado, puesto que la petición debería provenir del dominio registrado.

### Con contenedores mediante Docker Swarm
Para este despliegue, se deben crear una N cantidad de máquinas que formarán parte de un clúster de nodos Docker, llamado Swarm. La cantidad mínima recomendada son 3 instancias (nodos).

Para realizarlo, se crean las 3 (o más) instancias (de preferencia `t2.micro`), que compartan el mismo grupo de seguridad. Además, se debe instalar docker y docker compose en todas; esto último se hace automáticamente yendo a la sección de "User Data" y colocando el siguiente script de bash:

    #!/bin/bash
    sudo apt update
    sudo apt install docker.io -y
    sudo apt install docker-compose -y
    
Se instalará docker y docker-compose en cada máquina.

Luego de esto, en cualquiera de las máquinas, se replica este mismo repositorio:

    git clone https://github.com/DexterX12/AWS-Deployment-Project.git

Dentro de la carpeta del proyecto, se encuentra el docker-compose con las configuraciones establecidas en la **sección 3** (recomendable leer para identificar las variables y modificaciones posibles). Por defecto, se crearán 8 replicas de la aplicación `flask` y 3 replicas del servicio de `nginx` con máximo 1 de esta última por máquina.

Antes de iniciar el swarm, es necesario crear las imágenes para el servicio de `flask` y `nginx`, esto con la finalidad de que cada nodo pueda replicar la imagen de manera independiente. (Esto requiere utilizar una cuenta de Docker, donde se utilizará Docker Hub para subir las imágenes. [Aquí lo explican](https://www.howtogeek.com/devops/how-to-login-to-docker-hub-and-private-registries-with-the-docker-cli/)).




## 5. Nombres de dominio
Los 3 despliegues se encuentran en el siguiente subdominio: `bookstore-alp.freeddns.org`, con la separación de cada uno de ellos dado de la siguiente manera:
* [monolith.bookstore-alp.freeddns.org](https://monolith.bookstore-alp.freeddns.org): Este enruta hacia la aplicación monolítica que corre únicamente en una sola máquina
* [bookstore-alp.freeddns.org](https://bookstore-alp.freeddns.org): Este enlace es asignado al DNS del load balancer para el objetivo 2 del Auto Scaling Group
* [swarm.bookstore-alp.freeddns.org](https://swarm.bookstore-alp.freeddns.org): Este enlace se asignó al DNS del load balancer del clúster de Docker Swarm


# Referencias
* _«Swarm mode»_. (2025, 29 de enero). Docker Documentation. https://docs.docker.com/engine/swarm/
* Habibullah. (2023, 2 de septiembre). Containerization of Python Flask Nginx in docker - Habibullah - Medium. _Medium_. https://medium.com/@habibullah.127.0.0.1/containerization-of-python-flask-nginx-in-docker-7c451b3328b7
* Joy. (2024, 21 de abril). _Load balancing with Docker Swarm & Nginx_. DEV Community. https://dev.to/joykingoo/load-balancing-with-docker-swarm-nginx-ef5
* _Setting up Docker Swarm High Availability in Production | Better Stack Community_. (2023, 22 de diciembre). https://betterstack.com/community/guides/scaling-docker/ha-docker-swarm/#step-7-deploying-a-highly-available-nginx-service
* William. (2023, 7 marzo). How to Create an Auto Scaling Group of EC2 Instances for High Availability. _Medium_. https://medium.com/@boltonwill/how-to-create-an-auto-scaling-group-of-ec2-instances-for-high-availability-c94e85cc8cf3
   
