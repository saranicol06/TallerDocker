📘 Taller Docker: Construcción, Contenedores, Compose y Balanceo
👩‍💻 Autora: Sara Nicol Zuluaga, Axel Daniel Bedoya
#️⃣ 1. Introducción

Este taller tiene como objetivo aprender a:

- Crear un Dockerfile desde cero

- Construir imágenes propias

- Crear una aplicación Python con Flask

- Conectarla a una base de datos Redis

- Ejecutar la aplicación dentro de contenedores

- Usar docker-compose para orquestar múltiples servicios

- Implementar un balanceador de carga usando Traefik

- Publicar una imagen en Docker Hub

#️⃣ 2. Preparación del entorno

Se creó la siguiente estructura:

```
friendlyhello/
│-- app.py
│-- Dockerfile
│-- requirements.txt
│-- docker-compose.yaml
│-- data/        (generada automáticamente por Redis)
```

<img width="1495" height="576" alt="image" src="https://github.com/user-attachments/assets/09bc74a3-dfa9-413a-9b59-9a80c565cbf5" />


Todos los archivos se trabajaron en:

C:\Users\saran\Sites\friendlyhello

#️⃣ 3. Creación de la aplicación Flask

Archivo: app.py

```python 
from flask import Flask
from redis import Redis, RedisError
import os
import socket

redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello {name}!</h3>" \
            "<b>Hostname:</b> {hostname}<br/>" \
            "<b>Visits:</b> {visits}"
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```

<img width="1903" height="1088" alt="image" src="https://github.com/user-attachments/assets/84d01675-b51a-4bdf-84b5-bce4fb893b5e" />

La aplicación devuelve:

- Un saludo

- El hostname del contenedor

- El número de visitas almacenado en Redis

#️⃣ 4. Dependencias

Archivo: requirements.txt

```
Flask
Redis
```
<img width="1705" height="374" alt="image" src="https://github.com/user-attachments/assets/260a2b02-5cb5-48cd-a8fc-3f03dcc36105" />


#️⃣ 5. Creación del Dockerfile

Archivo: Dockerfile

```
FROM python:3-slim
WORKDIR /app
COPY . /app
RUN pip install --trusted-host pypi.python.org -r requirements.txt
EXPOSE 80
ENV NAME World
CMD ["python", "app.py"]
```

<img width="1353" height="365" alt="image" src="https://github.com/user-attachments/assets/11f21fb8-c0ad-41fb-bcd5-c21d9b699e20" />


Este Dockerfile:

- Usa Python 3-slim como base

- Copia el código al contenedor

- Instala dependencias

- Expone el puerto 80

- Ejecuta la aplicación

#️⃣ 6. Construcción de la imagen

Comando:

```
docker build -t saranicol06/friendlyhello .
```
<img width="2343" height="388" alt="image" src="https://github.com/user-attachments/assets/7cb25ab8-0e29-4ef3-a467-2c7e35d1ad98" />

Se verificó con:

```
docker images
```

<img width="2123" height="573" alt="image" src="https://github.com/user-attachments/assets/19edd9f8-ee67-4ee6-998e-b0b6b24bbde4" />


#️⃣ 7. Subida de imagen al Docker Hub

Iniciar sesión:

```
docker login
```
<img width="2342" height="428" alt="image" src="https://github.com/user-attachments/assets/e3b83807-6fd3-49af-8087-962350acee14" />

Etiquetar la imagen:

```
docker tag friendlyhello saranicol06/friendlyhello
```

Publicar:

```
docker push saranicol06/friendlyhello
```
<img width="2008" height="793" alt="image" src="https://github.com/user-attachments/assets/d2221fdf-4f4d-4378-a001-eeb6f5f2bc87" />


#️⃣ 8. Creación del archivo docker-compose.yaml

Archivo: docker-compose.yaml

```
version: "3.8"

services:
  web:
    image: saranicol06/friendlyhello:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.web.rule=Host(`localhost`)"
      - "traefik.http.routers.web.entrypoints=web"
      - "traefik.http.services.web.loadbalancer.server.port=80"
    depends_on:
      - redis

  redis:
    image: redis:latest
    volumes:
      - "./data:/data"
    command: ["redis-server", "--appendonly", "yes"]
    labels:
      - "traefik.enable=false"

  traefik:
    image: traefik:v2.3
    command:
      - "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedByDefault=false"
      - "--entrypoints.web.address=:4000"
    ports:
      - "4000:4000"
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    labels:
      - "traefik.enable=true"
```

<img width="1987" height="1526" alt="image" src="https://github.com/user-attachments/assets/1f40bf14-8dd1-4b84-ab9d-752233f4d1d3" />

Este archivo crea 3 servicios:

Servicio	Función
web	Aplicación Flask dentro de un contenedor
redis	Base de datos en memoria
traefik	Balanceador de carga y reverse proxy

#️⃣ 9. Levantar toda la aplicación

```
docker compose up -d
```

<img width="2303" height="527" alt="image" src="https://github.com/user-attachments/assets/0fbfd912-face-4a58-847e-7fe68917485c" />


Ver contenedores:

```
docker ps
```

<img width="2330" height="854" alt="image" src="https://github.com/user-attachments/assets/2e9d0f47-45f5-4713-8ef4-a2e1d4ad2f65" />


#️⃣ 10. Probar la aplicación

✔ App corriendo:
http://localhost:4000

<img width="2879" height="1622" alt="image" src="https://github.com/user-attachments/assets/7028ae9e-4dc8-46f6-8e26-84a33f05dfe0" />


✔ Dashboard de Traefik:
http://localhost:8080

<img width="2879" height="1531" alt="image" src="https://github.com/user-attachments/assets/21afcfba-753f-4c7f-b25c-da526a8433dd" />


#️⃣ 11. (Opcional) Probar balanceo de carga

Escalar la aplicación:

```
docker compose up -d --scale web=5
```

<img width="2340" height="618" alt="image" src="https://github.com/user-attachments/assets/d09535fc-8bbc-4e61-bc7c-a533f84e283d" />

Al recargar varias veces http://localhost:4000
 verás hostnames diferentes, uno por contenedor.
 

#️⃣ 12. Conclusiones

Docker permite crear aplicaciones aisladas, portables y reproducibles.

Un Dockerfile define exactamente cómo construir una imagen.

Docker Compose permite orquestar varios servicios fácilmente.

Redis se integró como servicio externo sin instalar nada en Windows.

Traefik se usó como balanceador para manejar múltiples instancias.

La imagen final se publicó en Docker Hub correctamente.



