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
	bookstore_net:
```
En este caso, se elimina por completo la sección de mysql dentro del docker compose y, se establecen variables de entorno para la conexión a la base de datos (esquema, usuario, contraseña y dirección de la BD).

Además, para cada instancia, se utiliza el servicio de `nginx` el cual se configura de la siguiente manera:
```nginx


server {
	listen 80;
	server_name bookstore-alp.freeddns.org;

	location / {
		proxy_pass http://localhost:5000;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	}
}
```
### Versión monolítica con Docker Swarm
Docker Swarm permite la orquestación de contenedores, permitiendo escalar y replicar los contenedores de las aplicaciones, puntos cruciales para la tolerancia a fallos y alta disponibilidad.
Esta versión modifica el archivo `docker-compose.yml` para agregar una propiedad de `deploy` a cada servicio. Este define como se debe comportar cada contenedor en cuanto a cantidad de réplicas, y cuantas de estas por instancia, entre otros aspectos.
Para el proyecto, se definió 8 réplicas de la aplicación en `flask` y 3 replicas de `nginx` con la condición de repartirse 1 máximo en cada instancia. 
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
Se agregó además, un servicio de `nginx` como un contenedor que reside en la misma red de las apps de `flask`. El propósito de esto último recae en poder redirigir mediante proxy inverso las peticiones que le llegue a cualquier máquina que haga parte del *swarm* a un contenedor de aplicación `flask` cualquiera. Esto es gracias al DNS interno del swarm que, al acceder mediante el nombre del servicio de la aplicación flask, devuelve una coincidencia de cualquiera de las réplicas, utilizando round-robin. Esto se especifica en el `default.conf` de `nginx`:

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
Donde `flaskapp` es el nombre del servicio que hace parte de la misma red interna.

Para ejecutar el docker swarm, basta con ejecutar el siguiente comando en alguna máquina para ser manager:

    docker swarm init --advertise-addr [IP MAQUINA]

Es necesario especificar la IP de la máquina en la cual corre la instancia.

Se generará un token para que otros nodos se unan como workers al swarm. Si se desea agregar más managers, dentro un nodo manager se debe ejecutar el siguiente comando:

    docker swarm join-token manager
Devolverá el token para unir un nodo como manager del swarm.

Por último, para generar el servicio y sus réplicas (especificadas en el docker-compose), se ejecuta el siguiente comando:

    docker stack deploy --compose-file docker-compose.yml <nombre_servicio>

# 4. Arquitectura de despliegue y componentes

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
Para ejecutar el servidor, basta con clonar el repositorio y utilizar el archivo `docker-compose.yml` para ejecutar el entorno requerido, cuyos pasos se encuentran en la sección 3.

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
   
