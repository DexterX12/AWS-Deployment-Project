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
#### Versión monolítica con base de datos
Para la versión monolítica donde la base de datos reside en la misma máquina, se utilizó el proyecto con el docker compose de [*este repositorio*](https://github.com/st0263eafit/st0263-251/tree/main/proyecto2). Para ejecutar esta versión, basta con instalar `docker` y `docker-compose`. Luego, dentro del directorio raíz del proyecto, ejecutar el siguiente comando:

    docker compose up -d
Esto ejecutará el servidor de `flask` en un contenedor, seguido del servidor MySQL en otro.
#### Versión monolítica con base de datos remota
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
#### Versión monolítica con Docker Swarm
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
   
   
