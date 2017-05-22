# Documentación docker:

https://docs.docker.com/
http://www.formandome.es/linux/docker/



## Instalación
- Dos versiones: 
  - CE (Community edition)
  - EE (Enterprise edition)

- Ya tenemos instalada la versión CE en nuestra máquina virtual
https://docs.docker.com/engine/installation/linux/ubuntu/




# Búscamos nuestra imagen para un servidor web

- Vía consola:
```
docker search php
```

- [Vía web](https://hub.docker.com/)


# Levantamos un servidor web apache:

- Fichero docker-compose.yml:

```
version: '3'
services:
  web1:
    image: php:5.6-apache
    container_name: web1
```

- Levantamos el servicio:
```
docker-compose up -d
```

- Comprobamos su ejecución: 

```
docker-compose ps

➜  demo docker-compose ps
Name              Command               State   Ports  
------------------------------------------------------
web1   docker-php-entrypoint apac ...   Up      80/tcp 
```

- No podemos acceder a el, ya que nos hace falta un puerto en local, así que tendremos que añadir el puerto en el fichero docker-compose.ytm y reiniciar el servicio:
  - Fichero docker-compose.yml
```
version: '3'
services:
  web1:
    image: php:5.6-apache
    container_name: web1
    ports:
      - "8000:80"
```

  - Parar y arrancar el servicio:

```
docker-compose down

➜  demo docker-compose up -d
Creating network "demo_default" with the default driver
Creating web1
➜  demo docker-compose ps
Name              Command               State          Ports         
--------------------------------------------------------------------
web1   docker-php-entrypoint apac ...   Up      0.0.0.0:8000->80/tcp 
```

- Si ahora vamos a http://localhost:8000
  - No sirve nada
- Podemos entrar a la máquina virtual, ejecutando un comando sobre ella (un shell bash):
```
# docker-compose exec web1 bash
cd /var/www/html
```

- Habría que añadir un fichero con datos, quizá lo más cómodo sea desde nuestra máquina ya que nuestro contenedor ¡no tiene ni el paquete vi!

- Así que otra vez lo mismo...

```
version: '3'
services:
  web1:
    image: php:5.6-apache
    container_name: web1
    volumes:
      - ./data-web1:/var/www/html
    ports:
      - "8000:80"
```


Para comprobar el funcionamiento, creamos un fichero index.php en nuestro directorio data-web1 con el siguiente contenido:
```
cat data-web1
<?php phpinfo(); ?>
```

- Comprobamos al ejecutar http://localhost:8000 que el fichero se carga... sin embargo no tenemos las extensiones de bbdd:

  - ext/mysql (not recommended)
  - ext/mysqli
  - PDO_MySQL

# Acceso a base de datos

- Necesitamos los siguientes componentes:
  - Una base de datos
    - A partir de una nueva imagen
    - Filosofía docker: un contenedor por servicio 
  - Configurar nuestro PHP con las extensiones correspondientes

## Customizar nuestra imagen php
- La imagen de PHP que tenemos es básica, necesitamos otra nueva:
  - Podemos crear una nueva imagen
  - Podemos instalar componentes adicionales a partir de la anterior (mejor)

- Pasos:
  - Crearemos un directorio para nuestra imagen
  - Crearemos un fichero Dockerfile donde damos las instrucciones para generar nuestra imagen.
  
``` 
mkdir web1
cd web1

cat Dockerfile 
FROM php:5.6-apache
RUN docker-php-ext-install mysqli
```


- Reiniciamos:
```
docker-compose down
docker-compose up -d
```


# Creamos una imagen para nuestra base de datos:

- Otra vez como antes...
  - Vía consola:
```
docker search php
```

  - [Vía web](https://hub.docker.com/)

- Por ejemplo (¡respeta tabulaciones!):

```
  db:
    hostname: db
    image: mysql:5.5
    container_name: db
    volumes:
      - ./data-db:/var/lib/mysql
      - ./init-db:/docker-entrypoint-initdb.d
    environment:
      - MYSQL_ROOT_PASSWORD=password
      - MYSQL_DATABASE=demo
```

- Fijate que:
  - Pasamos variables de entorno según la documentación de la imagen
  - El nombre del host
  - Un volumen para mapear los ficheros de la base de datos y otro para inicializarla (a partir de un fichero sql)

- Descarga el fichero con los datos:
```
mkdir init-db
wget www.formandome.es/demo.sql
docker-compose down
docker-compose up -d
```
- Y probamos que funcione....

- Comprobamos que se ha instalado bien:
```
docker-compose exec db bash
mysql -uroot -ppassword
show databases
use demo
show tables
....
```

- Vamos a probarlo "en real" cambiando nuestro fichero index.php:
```
<?php
$servername = "db";
$username = "root";
$password = "password";
$dbname = "demo";

// Create connection
$conn = new mysqli($servername, $username, $password, $dbname);
// Check connection
if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
} 

$sql = "SELECT name, continent, region FROM country";
$result = $conn->query($sql);

if ($result->num_rows > 0) {
    // output data of each row
    while($row = $result->fetch_assoc()) {
        echo "País: " . $row["name"]. " - Continente: " . $row["continent"]. " -Región: " . $row["region"]. "<br>";
    }
} else {
    echo "0 results";
}
$conn->close();
?>
```


# ¿Y si tenemos más de un sitio web? 

- No pueden ir todos por el 80
- Necesitamos un proxy inverso (virtual hosts en Apache) que redireccione:
  docker search proxy
- Añadimos la url de mi sitio web en el fichero /etc/hosts para que vaya a local host (en un entorno real no haría falta)
- Añadimos registro a /etc/hosts:
127.0.0.1       web1.com        www.web1.com

- Modificamos el fichero docker-compose.yml (la redirección de puertos la ponemos en el proxy):

version: '2'
services:
  nginx-proxy:
    image: jwilder/nginx-proxy
    container_name: nginx-proxy
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
  web1:
    build: ./web1
    container_name: web1
    volumes:
      - ./data-web1:/var/www/html
    environment:
      - VIRTUAL_HOST=web1.com,www.web1.com
  db:
    hostname: db
    image: mysql:5.5
    container_name: db
    volumes:
      - ./data-db:/var/lib/mysql
      - ./init-db:/docker-entrypoint-initdb.d
    environment:
      - MYSQL_ROOT_PASSWORD=password
      - MYSQL_DATABASE=demo




Añadimos una segunda web, en este el php sin acceso a bbdd.




- Añadimos entrada en el docker-compose.yml:
  web2:
    image: php:5.6-apache
    container_name: web2
    volumes:
      - ./data-web2:/var/www/html
    environment:
      - VIRTUAL_HOST=web2.com,www.web2.com



- Añadimos registro a /etc/hosts:
127.0.0.1       web2.com        www.web2.com

- Creamos el directorio para los ficheros de la web

mkdir data-web2
cd data-web2 
echo "<?php phpinfo(); ?>" >index.php

- Levantamos el servicio:
$ docker-compose up -d web2   
Starting web2

$ docker-compose ps        

   Name                  Command               State         Ports        
-------------------------------------------------------------------------
db            docker-entrypoint.sh mysqld      Up      3306/tcp           
nginx-proxy   /app/docker-entrypoint.sh  ...   Up      0.0.0.0:80->80/tcp 
web1          docker-php-entrypoint apac ...   Up      80/tcp             
web2          docker-php-entrypoint apac ...   Up      80/tcp   



- Observa que ha funcionado directamente sin reinciar el servidor proxy :-)

