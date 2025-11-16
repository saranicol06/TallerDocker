Taller Docker: Construcción, Contenedores, Compose y Balanceo
Autores: Sara Nicol Zuluaga, Axel Daniel Bedoya
Fecha: 16/11/2025

#️ 1. Introducción

Este taller tiene como objetivo aprender a:

Crear un Dockerfile desde cero

Construir imágenes propias

Crear una aplicación Python con Flask

Conectarla a una base de datos Redis

Ejecutar la aplicación dentro de contenedores

Usar docker-compose para orquestar múltiples servicios

Implementar un balanceador de carga usando Traefik

Publicar una imagen en Docker Hub

#️2. Preparación del entorno

Se creó la siguiente estructura:

friendlyhello/
│-- app.py
│-- Dockerfile
│-- requirements.txt
│-- docker-compose.yaml
│-- data/        (generada automáticamente por Redis)


Todos los archivos se trabajaron en:

C:\Users\saran\Sites\friendlyhello

#️ 3. Creación de la aplicación Flask

Archivo: app.py

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


La aplicación devuelve:

Un saludo

El hostname del contenedor

El número de visitas almacenado en Redis

#️ 4. Dependencias

Archivo: requirements.txt

Flask
Redis

#️ 5. Creación del Dockerfile

Archivo: Dockerfile

FROM python:3-slim
WORKDIR /app
COPY . /app
RUN pip install --trusted-host pypi.python.org -r requirements.txt
EXPOSE 80
ENV NAME World
CMD ["python", "app.py"]


Este Dockerfile:

Usa Python 3-slim como base

Copia el código al contenedor

Instala dependencias

Expone el puerto 80

Ejecuta la aplicación

#️ 6. Construcción de la imagen

Comando:

docker build -t saranicol06/friendlyhello .


Se verificó con:

docker images

#️ 7. Subida de imagen al Docker Hub

Iniciar sesión:

docker login


Etiquetar la imagen:

docker tag friendlyhello saranicol06/friendlyhello


Publicar:

docker push saranicol06/friendlyhello

#️ 8. Creación del archivo docker-compose.yaml

Archivo: docker-compose.yaml

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


Este archivo crea 3 servicios:

Servicio	Función
web	Aplicación Flask dentro de un contenedor
redis	Base de datos en memoria
traefik	Balanceador de carga y reverse proxy
#️ 9. Levantar toda la aplicación
docker compose up -d


Ver contenedores:

docker ps

#️ 10. Probar la aplicación

✔ App corriendo:
http://localhost:4000

✔ Dashboard de Traefik:
http://localhost:8080

#️ 11. (Opcional) Probar balanceo de carga

Escalar la aplicación:

docker compose up -d --scale web=5


Al recargar varias veces http://localhost:4000
 verás hostnames diferentes, uno por contenedor.

#️ 12. Conclusiones

Docker permite crear aplicaciones aisladas, portables y reproducibles.

Un Dockerfile define exactamente cómo construir una imagen.

Docker Compose permite orquestar varios servicios fácilmente.

Redis se integró como servicio externo sin instalar nada en Windows.

Traefik se usó como balanceador para manejar múltiples instancias.

La imagen final se publicó en Docker Hub correctamente.

✔ Taller completado exitosamente.
