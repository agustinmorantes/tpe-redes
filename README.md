# TPE Redes Grupo 5 Zabbix - How-to

## VPC
En la consola de AWS ingresar al wizard para creación de VPC.

Crear una VPC con CIDR 10.0.0.0/16, con 3 subredes públicas:
- 10.0.1.0/24 en AZ1
- 10.0.2.0/24 en AZ2
- 10.0.3.0/24 en AZ3


## Security Groups
Creamos 3 security groups:

Uno para el Zabbix Server, uno para los Zabbix Proxies, y uno para los Web Servers


### Zabbix Server

Agregar allow inbound rules para HTTP/S, SSH y TCP 8080 a todas las ipv4 (0.0.0.0/0).

Agregar allow inbound rules para todo el tráfico proveniente de la VPC (10.0.0.0/16).

#### Integración OpenVPN

Para permitirle a los chicos del grupo de OpenVPN enviar las métricas al server, expusimos el puerto 10051 del Zabbix Server a internet en el security group (allow 10051 0.0.0.0/0).


### Zabbix Proxy

allow SSH 0.0.0.0/0

allow any 10.0.0.0/16


### Web Servers

allow RDP 0.0.0.0/0
allow SSH 0.0.0.0/0
allow HTTP 0.0.0.0/0

allow any 10.0.0.0/16



## Instancias EC2

Creamos las 5 instancias linux de la misma forma:

1. Seleccionamos la AMI de Ubuntu 24.04 para las instancias de linux, o Windows Server 2025 base para la instancia de Windows.
2. Instancia t2.micro
3. Creamos un par de claves ed25519/RSA para poder ingresar via SSH
4. En la pestaña network:
    
    a. Seleccionamos nuestra VPC
    
    b. Seleccionamos la subnet correspondiente
    
    c. Asignamos la IP correspondiente a la instancia que estamos creando según nuestro diseño
    
    d. Asignamos el security group correspondiente previamente creado.
    
    e. Marcamos auto-assign public IP


## Setup Zabbix Server

### Setup inicial
1. Actualizar los packages con `sudo apt update && sudo apt upgrade -y` y reiniciar la instancia si se requiere.
2. Instalar los paquetes postgres y nginx: `sudo apt install postgresql nginx`
3. Instalar Zabbix Server, Agent y GUI según las instrucciones para la última version con Ubuntu 24.04, postgresql y nginx en [zabbix.com/download](https://www.zabbix.com/download?zabbix=7.2&os_distribution=ubuntu&os_version=24.04&components=server_frontend_agent&db=pgsql&ws=nginx).

    Esto implica instalar el repo de apt de Zabbix, crear la db y el usuario en postgres, configurar el usuario y la contraseña de la db en la config del server y habilitar el servicio con systemctl.

4. Habilitar y reiniciar el servicio de nginx
    ```sh
    sudo systemctl enable --now nginx
    sudo systemctl restart nginx
    ```

5. Ahora que la GUI está habilitada, podemos ingresar desde el browser con la ip pública del server en el puerto 8080.

6. Ingresar con el usuario admin con las credenciales default y cambiar la password a una privada por seguridad.

    Credenciales default:
    - User: Admin
    - Pass: zabbix

Con esto tenemos el Zabbix Server corriendo y acceso a su GUI.

### Agregar proxies

1. En la sidebar, ingresar en Administration -> Proxies.
2. En la esquina superior derecha clickear el botón "Create proxy"
3. Ingresar el nombre del proxy. Es importante recordar este nombre ya que luego se debe usar el mismo en la configuración del proxy.
4. Marcar Proxy mode Active
5. Ingresar la ip privada del proxy.

Repetir por cada proxy que queramos agregar (en nuestro diseño son 2).

### Ingresar Hosts
En la sidebar, ingresar en Monitoring -> Hosts.

#### Zabbix Server
Ingresaremos el server como un host para poder monitorear mediante el Agent los diferentes procesos y servicios que hostea.

1. Si ya existe la entrada del server, clickear y editar la configuración del host. Sino crearla con el botón "Create host".
2. Verificar que el hostname sea correcto
3. Asignar los siguientes templates:
    - Linux by Zabbix agent
    - PostgreSQL by Zabbix agent
    - Zabbix server health
4. Asignar los host groups:
    - Zabbix servers
    - Linux servers
    - Virtual machines
5. Agregamos la interfaz del Agent con IP 127.0.0.1
6. Seleccionamos Monitored by Server

#### Proxies
Por cada proxy, crear un host:
1. Click en la esquina superior derecha "Create host".
2. Ingresar el hostname del proxy
3. Linkear los siguientes templates:
    - Linux by Zabbix agent
    - PostgreSQL by Zabbix agent
    - Zabbix proxy by Zabbix agent
4. Seleccionamos los siguientes host groups:
    - Zabbix servers
    - Linux servers
    - Virtual Machines
5. Agregamos una interfaz de tipo Agent con la IP del proxy y puerto 10050.

6. En Monitored by seleccionar Server. Incluso si el proceso del proxy no funciona queremos que el server pueda saber el estado del agent.

#### Web server (Linux)
Por cada web server agregamos un host:
1. Creamos el host con "Create host"
2. Asignamos el hostname que usaremos en la configuración del Agent.
3. Seleccioamos los templates:
    - Linux by Zabbix agent
    - Nginx by Zabbix agent
4. Asignamos host groups:
    - Applications
    - Linux servers
    - Virtual machines
5. Creamos una interfaz para el Agent con la ip de la instancia EC2 y puerto 10050.
6. Seleccionamos Monitored by Proxy y elegimos el proxy correspondiente a la subnet del web server.

#### Web server (Windows)
ídem server linux, pero con el template Windows by Zabbix agent.

#### OpenVPN server
Para este server utilizamos el modo activo del Agent, por lo que solo necesitamos asignar un hostname y el Agent corriendo en el server de OpenVPN del grupo 2 se conectará al Zabbix Server para pedir su configuración y enviar sus métricas.

1. Crear host
2. Asignar hostname (Importante que sea el mismo que asigna el grupo 2 en su configuración del Agent).
3. Asignar template "Zabbix agent active"
4. Seleccionar Monitored by Server.

Notar que no se agrega interfaz ya que el server no pedirá de forma activa, sino que escuchará de forma pasiva en su propia IP.


### Configurar custom templates, items y triggers
#### Modificar valores de los triggers
Para que los triggers se activen con los valores que propusimos en la preentrega, los cuales no son los valores default, hay que modificar el template de Linux by Zabbix Agent con los valores propuestos. Al editar el template directamente, los cambios aplican a todos los hosts que lo utilizan.
1. En la sidebar, ingresar a Data Collection -> Templates
2. Buscar el template Linux by Zabbix Agent y clickear en Triggers
3. Identificar los triggers a modificar:
    -  High CPU utilization
    -  High Memory utilization
4. Entrar al trigger y cambiar en la expresión el valor 5m por 1m que es el tiempo que definimos en la preentrega.
5. Entrar a la pestaña Discovery rules, ubicar la entrada "Mounted filesystem discovery" y clickear Trigger prototypes.
6. Modificar las entradas "Space is low" y "Space is critically low" cambiando en la expresión el valor 5m por 1m que es el valor que definimos en la preentrega.
7. En la configuración del template, ingresar a macros y cambiar los valores críticos de CPU, memoria y espacio en disco, así como cambiar el timeout a 1 minuto para la alerta de desconexión:
    - `{$AGENT.TIMEOUT}` => `1m`
    - `{$CPU.UTIL.CRIT}` => `70`
    - `{$MEMORY.UTIL.MAX}` => `70`
    - `{$VFS.FS.PUSED.MAX.CRIT}` => `90`
    - `{$VFS.FS.PUSED.MAX.WARN} => 60`

Realizar lo mismo para el template Windows by Zabbix Agent.

Con esto, todos los hosts que utilicen el template Linux by Zabbix Agent o Windows by Zabbix Agent dispararán las alertas que definimos en la preentrega sin configuración adicional.

#### Web scenarios
Vamos a crear un web scenario para chequear via HTTP que la aplicación está corriendo y que devuelve status 200.

1. Buscar el template Nginx by Zabbix agent.
2. Ingresar en la pestaña web del template.
3. Clickear en la esquina superior derecha "Create web scenario".
4. Ponerle un nombre, por ejemplo "HTTP OK"
5. Seleccionar Agent Zabbix
6. Entrar en la pestaña "Steps"
7. Clickear en "Add" para crear un nuevo step
8. Le ponemos nombre `GET /`
9. En la URL ponemos `http://{HOST.IP}`. Con esto, el proxy correspondiente accederá al web server mediante la ip correspondiente a cada host que utilice el template.
10. Ponemos required status codes: `200`
11. Agregamos un trigger en el template Nginx by Zabbix Agent con nombre `HTTP Error`
12. Marcamos severity High
13. Utilizamos la siguiente expresión 
    ```
    last(/Nginx by Zabbix agent/web.test.rspcode[HTTP OK,GET /])<>200
    ```

Con esto, los proxies chequearán cada 1 minuto via HTTP que cada uno de los servers devuelvan status 200. En caso contrario se disparará una alerta.

### Notificaciones via Discord
Para las notificaciones, utilizamos Discord. Desde nuestro canal de Discord generamos un Webhook, y colocamos la URL en la sección Media bajo el usuario Admin de Zabbix, con el tipo Discord. En "Use if severity" habilitamos todos los casilleros para recibir todas las notificaciones de cualquier alerta. Si quisieramos solo recibir alertas de determinada gravedad, podemos elegir las que queramos.

Luego en Alerts -> Actions -> Trigger Actions habilitamos la acción "Report problems to Zabbix administrators via all media".

En Alerts -> Media types habilitamos la opción De Discord.



## Setup Zabbix Proxies
1. Actualizar los paquetes e instalar postgres como hicimos para el server
2. Agregar el repo de zabbix e instalar los paquetes del Agent y Proxy como se indica en la documentación [zabbix.com/download](https://www.zabbix.com/download?zabbix=7.2&os_distribution=ubuntu&os_version=24.04&components=proxy&db=pgsql&ws=).
3. Editar la config en `/etc/zabbix/zabbix_proxy.conf`:
    - `Server=10.0.1.10`
    - `Hostname=`{Proxy hostname que usamos en el server}
4. Editar la config en `/etc/zabbix/zabbix_agentd.conf` y agregar nuevamente la ip del server y el hostname
5. Habilitar y reiniciar los servicios:
    ```bash
    sudo systemctl enable --now zabbix-agent zabbix-proxy postgresql
    sudo systemctl restart zabbix-agent zabbix-proxy postgresql
    ```

Con esto debería figurar el proxy ya funcionando en la UI del server.


## Setup Web Servers
### Linux
1. Actualizar los paquetes con APT.
2. Agregar el repo de zabbix e instalar el Zabbix Agent, como indica la documentación [zabbix.com/download](https://www.zabbix.com/download?zabbix=7.2&os_distribution=ubuntu&os_version=24.04&components=agent&db=&ws=).
3. Editar la configuración del Zabbix Agent en `/etc/zabbix/zabbix_agentd.conf` y agregar la ip del server y hostname como hicimos para los proxies.
4. Instalar nginx: `sudo apt install nginx`
5. Editar la configuración en `/etc/nginx/sites-available/default` y agregar el siguiente bloque dentro de la configuración del server:
    ```nginx
        location /basic_status {
                stub_status;
                allow 127.0.0.1;
                allow ::1;
                deny all;
        }
    ```

    Esta configuración le expone métricas de nginx al Zabbix Agent de forma local.
6. Habilitar y reiniciar el servicio nginx:
    ```bash
    sudo systemctl enable --now nginx
    sudo systemctl restart nginx
    ```
### Windows
El proceso en windows es el mismo solo que utilizando las herramientas propias de Windows.

Utilizamos Winget como package manager para instalar el Zabbix Agent, y como la versión de Winget de nginx no nos permitía configurarla correctamente utilizamos una versión descargada de la web oficial.


## Setup adicional monitoreo PostgreSQL

Para que el Agent del Server y de los Proxies pueda monitorear la base de datos, es necesario crear un usuario de monitoreo con los permisos correspondientes en la DB, y colocar las queries necesarias en cada host. Los archivos y pasos necesarios están descriptos en detalle en el repo del template [PostgreSQL by Zabbix Agent](https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/db/postgresql?at=4.4.4rc1)

## Monitoreo OpenVPN

Para monitorear el servidor OpenVPN, debimos crear un template custom ya que no hay uno oficial. Para monitorear el server los chicos del grupo 2 instalaron en su VM el zabbix agent, y lo configuraron en modo activo ya que no podían exponerlo directamente a internet debido a que su implementación está detras de múltiples NAT. Para no tener que pedirles que cambien la configuración si cambiaba la IP pública del Zabbix Server, utilizamos el servicio de DuckDNS para tener un dominio dinámico gratuito.

### Creación de template
1. En el Side menu entramos en la sección Data collection -> Templates
2. Clickeamos el botón en la esquina superior derecha para crear el template
3. Le asignamos el nombre "OpenVPN by Zabbix Agent Active"
4. Lo colocamos en los template groups:
    - Templates
    - Templates/Applications

### Items nuevos
Para los items utilizamos la key built-in de zabbix `log[...]` y `log.count[...]`, que nos permiten leer logs en el filesystem y parsearlos con regex. También utilizamos items "Calculated", para realizar agregaciones sobre los mismos, como en el caso del ancho de banda utilizado. 

Dentro del template vamos a la pestaña Items.

#### Connected clients
1. Creamos un nuevo item con el boton Create Item
2. Le ponemos de nombre "Connected Clients"
3. Seleccionamos el tipo "Zabbix Agent (active)"
4. Utilizamos la siguiente key:
    ```
    log.count[/var/log/openvpn/openvpn-status.log,"^CLIENT_LIST",ANSI,100,all,0]
    ```
5. Dejamos el resto como default y guardamos.

#### Total bytes received
Creamos un nuevo item con los mismos settings pero en la key usamos lo siguiente:
```
log[/var/log/openvpn/openvpn-status.log,"^CLIENT_LIST\W+debian1\W+((\d+\.?)+(:\d+)?)\W+((\d+\.?)+(:\d+)?)\W+(\d+)\W+(\d+)",ANSI,100,,\7]
```

#### Total bytes sent
Lo mismo que el anterior pero cambiamos el valor seleccionado en la key por \8:
```
log[/var/log/openvpn/openvpn-status.log,"^CLIENT_LIST\W+debian1\W+((\d+\.?)+(:\d+)?)\W+((\d+\.?)+(:\d+)?)\W+(\d+)\W+(\d+)",ANSI,100,,\8]
```

#### Download rate
1. Creamos un nuevo item, pero utilizamos el tipo "Calculated"
2. Type of information: "Numeric (unsigned)"
3. Units: B/s
4. Usamos la siguiente formula:
```
rate(//log[/var/log/openvpn/openvpn-status.log,"^CLIENT_LIST\W+debian1\W+((\d+\.?)+(:\d+)?)\W+((\d+\.?)+(:\d+)?)\W+(\d+)\W+(\d+)",ANSI,100,,\7],2m)
```

#### Upload rate
Lo mismo pero con la siguiente formula:
```
rate(//log[/var/log/openvpn/openvpn-status.log,"^CLIENT_LIST\W+debian1\W+((\d+\.?)+(:\d+)?)\W+((\d+\.?)+(:\d+)?)\W+(\d+)\W+(\d+)",ANSI,100,,\8],2m)
```

### Triggers
En la pestaña triggers del template, vamos a crear los siguientes 3 triggers:

#### High bandwidth usage
Utilizamos la siguiente expresión:
```
min(/OpenVPN by Zabbix Agent Active/download_rate,1m)>10M or min(/OpenVPN by Zabbix Agent Active/upload_rate,1m)>10M
```
Seteamos severity Average y el resto de los settings default.

#### No clients connected
La expresión es la siguiente:
```
last(/OpenVPN by Zabbix Agent Active/log.count[/var/log/openvpn/openvpn-status.log,"^CLIENT_LIST",ANSI,100,all,0])<1
```
Severity High, el resto default.

#### No data transmitted
La expresión:
```
max(/OpenVPN by Zabbix Agent Active/upload_rate,1m)=0 and max(/OpenVPN by Zabbix Agent Active/download_rate,1m)=0
```
Severity High, el resto default.

