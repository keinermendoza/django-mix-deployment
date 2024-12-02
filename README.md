# Django Deploy con Docker + static (NGINX desde el HOST):

La idea es combinar la practicidad de Docker para conteinerizar nuestra aplicacion, pero al mismo tiempo mantener la posibilidad de servir diferentes apps desde una misma maquina.

Asumiré que sabes como usar Docker junto con un archivo docker-compose.yml, así como cargar variables de entorno y crear volumenes.

Solo como una vista general implicito lo siguiente:

- Tenemos acceso a una VPN con sistema operativo LINUX (ubuntu) a la que podemos conectarnos via SSH
- Hemos instalado NGINX
- Sabemos crear una imagen de nuestra app y subirla a dockerhub (no está incluído aqui)
- nuestra app esta en la carpeta project
- nuestro projecto (django) se llama **project**
- nuestra app (django) se llama **core**
- existe un archivo **.env.prod**  con todas las variables de entorno que necesita nuestro projecto 
- existe un archivo **.env.database** con todas las variables que necesita nuestra base de datos 
- existe un archivo **entrypoint.sh** para esperar que el contenedor de la base de datos esté listo para recibir conexiones

la estructura de nuestro projecto será la siguiente:

> [!IMPORTANT] Recordando que esta estructura es para producción y que la imagen de nuestra app para este punto ya fue creada y pusheada a dockerhub
```
project/
├── .env.prod
├── .env.database
├── entrypoint.sh
├── docker-compose.yml # a continuación lo crearemos
└── static/ # n subfolders más adelante la crearemos
```
 
## django project
Aqui muestro algunas configuraciones que hice en el projecto **antes de crear la imagen y enviarla a dockerhub**.

> [!IMPORTANT] A modo de recordatorio no olvidar usar las varaibles de entorno y setear la variable STATIC_ROOT

```python
# project/settings.py

SECRET_KEY = environ.get('SECRET_KEY')
DEBUG = bool(int(environ.get('DEBUG', '0')))
ALLOWED_HOSTS = environ.get('ALLOWED_HOSTS').split(' ')
CSRF_TRUSTED_ORIGINS =  environ.get('CSRF_TRUSTED_ORIGINS').split(' ')

DATABASES = {
    'default': {
        'ENGINE': environ.get('ENGINE', 'django.db.backends.sqlite3'),
        'HOST': environ.get('SQL_HOST'),
        'USER': environ.get('USER'),
        'NAME': environ.get('NAME', BASE_DIR / 'db.sqlite3'),
        'PASSWORD': environ.get('PASSWORD'),
        'PORT': environ.get('SQL_PORT'),
    }
}

STATIC_URL = 'static/'

## IMPORTANTE!
STATIC_ROOT = BASE_DIR / "static"
```
> [!WARNING]No estoy seguro de si es necesario agregar estas urls, editar este comentario
>

```python

# project/urls.py
from django.conf import settings
from django.conf.urls.static import static
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('core.urls')),
# ]
] + static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)
```

## Archivo docker-compose.yml

El siguiente es un archivo docker-compose.yml para producción.

> [!WARNING] NO olvidar remplazar las variables:
> - <mi_usuario_en_dockerhub>
> - <nombre_de_mi_projecto>
> - <nombre_de_mi_volumen>



```yaml
services:
  app:
    image: <mi_usuario_en_dockerhub>/<nombre_de_mi_projecto>:v1.0.0
    command:
      - /bin/sh
      - -c
      - |
        . /usr/src/app/entrypoint.sh
        gunicorn project.wsgi:application --bind 0.0.0.0:8010
    volumes: # importante
      - ./static:/usr/src/app/static
    ports:
      - 8010:8010
    env_file: './.env.prod'
    restart: unless-stopped
    depends_on:
      - db


  db:
    image: postgres:16-bullseye
    env_file: './.env.database'
    restart: unless-stopped
    volumes:
      - <nombre_de_mi_volumen>:/var/lib/postgresql/data/

volumes:
  <nombre_de_mi_volumen>:

```
El proposito del volumen es permitir que los archivos estaticos dentro del contenedor sean accesibles desde el host para que aspi **NGINX** pueda servirlos.

Una vez creado el archivo anterior en nuestro servidor ejecutamos nuestros servicios docker como de costumbre

```bash
sudo docker compose up
```

> [!WARNING] Asumiendo que la carpeta en la que estamos ejecutando nuestro archivo docker-compose.yml es la carpeta project

Entramos en nuestro app

```bash
sudo docker exec -it project-app-1 bash
```

Una vez alli colectamos los archivos estaticos como de costumbre

```bash
python manage.py collectstatic 
```
En este punto hemos terminado la configuracion relacionada con nuestra app

## NGINX

Crear un archivo desde donde servir nuestro sitio web
> [!IMPORTANT] Nombraré el archivo misitio.conf 
```
sudo nano /etc/nginx/sites-enabled/misitio.conf
```

Una vez abierto el archivo pegaré la siguiente configuración

> [!WARNING] Recordar remplazar la variable ip_o_domain con la ip de la maquina o preferiblemente con el dominio que hayamos comprado
 
```
server {
    server_name  <ip_o_domain>;
    charset utf-8;

	location = /favicon.ico { 
        access_log off; log_not_found off; 
        }
        client_max_body_size 75M;
        # esto es opcional, si lo quieres usar debes crear manualmente el directorio y ambos archivos, NGINX escribirá en ellos.
	    # access_log /var/log/nginx/misitio/misitio_access.log;
        # error_log /var/log/nginx/misitio/misitior_error.log;     
 	
    # IMPORTANTE
	location /static {
		alias /home/ubuntu/auth_provider/static;		
	}

	location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_pass http://localhost:8010; 		
    }
}

```

ahora creamos un link simbólico

```
sudo ln -s /etc/nginx/sites-available/misito.conf /etc/nginx/sites-enabled/
```

eliminamos el archivo default
```
sudo rm /etc/nginx/sites-available/default.conf
```
testeamos la configuracion
```
sudo nginx -t
```

En este punto deberiamos recibir una respuesta como: 
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Ahora tenemos que darle permiso a NGINX de leer los archivos en la carpeta static de nuestro proyecto, si lo dejamos tal como está recibiremos una respuesta **HTTP 403** cada vez que nuestra app intente cargar alguno de nuestros estaticos.  

```
sudo usermod -a -G tu_usuario www-data
sudo chown -R :www-data /ruta/hasta/project/static
```

si no sabes cual es tu usuario recuerda puedes hacerle esa pregunta existencial con el comando
```bash
whoami
```

reiniciamos NGINX
```
sudo systemctl start nginx
```
Para este punto tu sitio debe estar corriendo en docker mientras srive los archivos estaticos desde NGINX corriendo en el HOST.

### FIN