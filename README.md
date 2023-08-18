## Descripción:
> Se crearán 4 contenedores con Docker:
> 1. Un contenedor con la imagen **mysql:debian**
> 2. Un contenedor con la imagen **php:apache-bullseye**: el cual contendrá el primer servidor que alojará un simple index.php que deberá servir.
> 3. Otro contenedor con la imagen  **php:apache-bullseye**: el cual contendrá el segundo servidor que alojará el mismo index.php que el primer servidor.
> 4. Finalmente, crearemos un contenedor con la imagen **haproxy:lataest**: el cual contendrá un balanceador de carga que redirigirá las peticiones alternando entre el primer y segundo servidor sucesivamente.

#### Funcionamiento:
Con un navegador de nuestra máquina deberemos hacer una petición a localhost:8085 y este deberá recibir el sitio web que sirve el primer servidor. Luego de actualizar la página, nuestro balanceador deberá interceder y enviar la petición al segundo servidor, que servirá la misma página del servidor 1.
Ahora bien, nuestros servidores se comunicarán con una base de datos que se alojará en el contenedor de mysql. Esta base de datos tendrá algunos datos cargados a mano por nosotros mismos para que la simulación de una consulta a la base de datos por parte de nuestros servidores tenga sentido.


## Características
#### SO: Windows 10.

## Requerimientos:
1. Tener instalado **Docker Desktop**

## Procedimiento

### Primero descargamos las imágenes que necesitemos

Usamos el comando:
```
docker pull <nombre_de_la_imagen>
```

```
docker pull php:apache-bullseye
```

```
docker pull mysql:debian
```

```
docker pull haproxy:latest
```


![[Pasted image 20230816160930.png]]

### Creamos los contenedores

#### 1ro. Contenedor del gestor de bases de datos MySQL

```
docker run -d -p 3306:3306 -v "C:\Users\Usuario\Dropbox\DESCARGAS TEMPORALES CLASSROOMok\saIII-MicroServicios\contenedoresConApacheYMysql\mysqlVolumen":/var/lib/mysql --name mimsql -e MYSQL_ROOT_PASSWORD=pass mysql:debian
```

![[Pasted image 20230817104710.png]]

Vemos que estamos creando un volumen, que persistirá los datos del sistema de archivos del contendor en la ruta `/var/lib/mysql` en nuestro sistema de archivos de nuestra máquina local `C:\Users\Usuario\Dropbox\DESCARGAS TEMPORALES CLASSROOMok\saIII-MicroServicios\contenedoresConApacheYMysql\mysqlVolumen`
Esto nos permitirá crear tantas bases de datos, tablas y registros como queramos; y por más que eliminemos el contendor, estos datos no se perderán pues estarán guardados en el sistema de archivos de nuestra máquina local.

###### Importante:
La contraseña para este contenedor: **`-e MYSQL_ROOT_PASSWORD=pass` debe ser la misma que la que colocamos en nuestro fichero ``index.php``**.

##### Configuración del contenedor mysql

![[Pasted image 20230817104815.png]]

![[Pasted image 20230817105234.png]]

![[Pasted image 20230817105442.png]]

```
docker inspect mimsql
```
![[Pasted image 20230817105648.png]]


```
docker run -d -p 8088:80 --name myServer -v "C:\sitio1":/var/www/html php:apache-bullseye
```
![[Pasted image 20230817113133.png]]


```
docker exec -it myServer bash
```

```
apt-get update
```

![[Pasted image 20230817113226.png]]


```
apt-get install -y libmysqli-dev
docker-php-ext-install mysqli
docker-php-ext-enable mysqli
```



#### Creación del contenedor para el segundo servidor


```
docker run -d -p 8087:80 --name myServer2 -v "C:\sitio2":/var/www/html php:apache-bullseye
```



```
docker exec -it myServer2 bash
```

```
apt-get update
```

```
apt-get install -y libmysqli-dev
```

```
docker-php-ext-install mysqli
```

```
docker-php-ext-enable mysqli
```


#### Creación del contenedor para el balanceador

##### Crear un directorio en nuestra máquina para mapear un volumen al contenedor del balanceador.
En mi caso:
```
C:\volumenBalanceador
```
 * Dentro de este directorio crear el fichero: **haproxy.cfg**
 * En este fichero colocar las configuraciones de nuestro balanceador:
```
global
	pidfile /tmp/haproxy.pid
	log 127.0.0.1 local0 info
defaults
	log global
	mode http
	option httplog
	option dontlognull
	timeout connect 5000
	timeout client 50000
	timeout server 50000
frontend http_front
	bind 0.0.0.0:80
	default_backend http_back
backend http_back
	balance roundrobin
	cookie JSESSIONID prefix indirect nocache
	server MyServer 172.17.0.4:80 check
    server MyServer2 172.17.0.3:80 check

```

	# **Asegurarse de dejar una línea en blanco al final del fichero para que exista un salto de línea en la última línea escrita.** 




 * Otra configuración:
```
global
    maxconn 256

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend http-in
    bind *:8088
    default_backend servers

backend servers
    balance roundrobin
    server myServer 172.17.0.2:80
	server myServer2 172.17.0.4:80
```


```
global
    log 127.0.0.1 local0 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend http_front
    bind *:80
    mode http
    acl auth_ok http_auth(myrealm)
    http-request auth realm myrealm if !auth_ok
    default_backend http_back

backend http_back
    balance roundrobin
    cookie JSESSIONID prefix indirect nocache
    server server1 192.168.1.10:80 check
    server server2 192.168.1.11:80 check

listen stats
    bind :9000
    mode http
    stats enable
    stats uri /haproxy
    stats realm Strictly\ Private
    stats auth admin:yourpassword

```

```
global
    log 127.0.0.1 local0 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend http_front
    bind *:80
    mode http
    default_backend http_back

backend http_back
    balance roundrobin
    cookie JSESSIONID prefix indirect nocache
    server server1 192.168.1.10:80 check
    server server2 192.168.1.11:80 check

listen stats
    bind :9000
    mode http
    stats enable
    stats uri /haproxy
```



##### Crear el contenedor:

```
docker run -d -p 8085:80 --name balancer -v "C:\volumenBalanceador":/usr/local/etc/haproxy/ haproxy:latest
```



```
docker exec -it myServer2 bash
```

```
apt-get update
```

```
apt-get install -y libmysqli-dev
```

```
docker-php-ext-install mysqli
```

```
docker-php-ext-enable mysqli
```










































