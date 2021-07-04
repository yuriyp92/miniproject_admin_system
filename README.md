# Miniproyecto Administración de Sistemas

El objetivo del proyecto es automatizar el despliegue de una aplicación Django con Fabric, Ansible y Docker. La aplicación utilizada a modo de ejemplo se denomina Shield y se ha desarrolado durante el módulo de Desarrollo Web. 

Se han de seguir los siguientes pasos:

## Instalación del proyecto

1. Hacer desde su perfil de GitHub un fork del siguiente repositorio.
```
https://github.com/yuriyp92/shield
```
2. Descargar en su entorno de desarrollo el repositorio.
```
git clone https://github.com/yuriyp92/shield.git
```
3. Dentro de la carpeta `shield` crear un entorno virtual.
```
python3 -m venv .venv
```
4. Activar el entorno vitual.
```
source .venv/bin/activate
```
5. Instalar las librerías necesarias que se encuentran en el fichero `requirements.txt`.
```
pip install -r requirements.txt
```
6. Generar y ejecutar las migraciones.
```
python3 manage.py makemigrations
python3 manage.py migrate
```
7. Cargar los datos de la aplicación. 
```
python3 manage.py loaddata metahumans/fixtures/initial_data.json
```
8. Crear un super usuario para poder entrar en el admin de Django. 
```
python3 manage.py createsuperuser
```
9. Ejecutar el servidor de Django para probar el funcionamiento de la aplicación.
```
python3 manage.py runserver
```

## Despliegue de la aplicación con Fabric

1. Instalar la libreria `fabric`.
``` 
pip install fabric
```
2. Crear un fichero denominado `fabfile.py` dentro de la carpeta `shield`, con un contenido que se muestra en los pasos siguientes.

3. Añadir librerias.
```
import sys
import os
from fabric import Connection, task
```
4. Definir una serie de variables cuya utilidad es la de ser unas constantes que permiten el despliegue de la aplicación.
```
PROJECT_NAME = "shield"
PROJECT_PATH = f"~fab /{PROJECT_NAME}"
REPO_URL = "https://github.com/yuriyp92/miniproject_admin_system.git"
VENV_PYTHON = f'{PROJECT_PATH}/.venv/bin/python'
VENV_PIP = f'{PROJECT_PATH}/.venv/bin/pip'
```
5. Añadir la tarea `development` que permite la conexion al servidor remoto.
```
@task
def development(ctx):
    ctx.user = 'vagrant'
    ctx.host = '192.168.33.10'
    ctx.connect_kwargs = {"password": "vagrant"}
```
6. Crear una función auxiliar que establece la conexión.
```
def get_connection(ctx):
    try:
        with Connection(ctx.host, ctx.user, connect_kwargs=ctx.connect_kwargs) as conn:
            return conn
    except Exception as e:
        return None
```
7. Añadir la tarea para el clonado del repositorio. 
```
@task
def clone(ctx): 
    print(f"clone repo {REPO_URL}...")   

    if isinstance(ctx, Connection):
        conn = ctx
    else:
        conn = get_connection(ctx)

    # Obtener las carpetas del directorio
    ls_result = conn.run("ls").stdout

    # Dividir el resultado para tener cada carpeta en un objeto de una lista
    ls_result = ls_result.split("\n")

    # Si el nombre del proyecto ya está en la lista de carpetas no es necesario hacer el clone 
    if PROJECT_NAME in ls_result:
        print("project already exists")
    else:
        conn.run(f"git clone {REPO_URL} {PROJECT_NAME}")
```
8. Definir la función que verifica la rama en la que se encuentra el proyecto.
```
@task
def checkout(ctx, branch=None):
    print(f"checkout to branch {branch}...")

    if branch is None:
        sys.exit("branch name is not specified")

    if isinstance(ctx, Connection):
        conn = ctx
    else:
        conn = get_connection(ctx)

    with conn.cd(PROJECT_PATH):
        conn.run(f"git checkout {branch}")
```
9. La siguiente tarea realiza un `git pull` de la rama `main` del repositorio.
```
@task
def pull(ctx, branch="main"):

    print(f"pulling latest code from {branch} branch...")

    if branch is None:
        sys.exit("branch name is not specified")

    if isinstance(ctx, Connection):
        conn = ctx
    else:
        conn = get_connection(ctx)

    with conn.cd(PROJECT_PATH):
        conn.run(f"git pull origin {branch}")
```
10. Añadir la tarea que crea un entorno virtual dentro de la carpeta del proyecto.
```
@task
def create_venv(ctx):

    print("creating venv....")

    if isinstance(ctx, Connection):
        conn = ctx
    else:
        conn = get_connection(ctx)
    with conn.cd(PROJECT_PATH):
        conn.run("python3 -m venv .venv")
        conn.run(f"{VENV_PIP} install -r requirements.txt")
```
11. Crear la función que ejecute las migraciones de Django.
```
@task
def migrate(ctx):
    print("checking for django db migrations...")

    if isinstance(ctx, Connection):
        conn = ctx
    else:
        conn = get_connection(ctx)

    with conn.cd(PROJECT_PATH):
        conn.run(f"{VENV_PYTHON} manage.py migrate")
```
12. Importar los datos de la aplicación.
```
@task
def loaddata(ctx):
    print("loading data from fixtures...")

    if isinstance(ctx, Connection):
        conn = ctx
    else:
        conn = get_connection(ctx)

    with conn.cd(PROJECT_PATH):
        conn.run(f"{VENV_PYTHON} manage.py loaddata fixtures/polls_data.json")
```
13. La función `deploy` permite conectar al servidor y realizar todas las tareas anteriores.
```
@task
def deploy(ctx):
    conn = get_connection(ctx)
    if conn is None:
        sys.exit("Failed to get connection")

    clone(conn)
    checkout(conn, branch="main")
    pull(conn, branch="main")
    create_venv(conn)
    migrate(conn)
    loaddata(conn)
```
14. Finalmente ejecutar el comando de fabric para iniciar el despliegue automático de la aplicación.
```
fab development deploy
```

## Despliegue de la aplicación con Ansible

1. Instalar la librería `ansible`
```
pip install ansible
```
2. Crear una carpeta denominada `ansible` dentro del proyecto.
```
mkdir ansible
```
3. Dentro de dicha carpeta crear los siguientes ficheros:
- **hosts**: contiene etiquetas con las direcciones IP de los servidores.
- **vars.yml**: contiene las variables del proyecto.
- **provision.yml**: contiene las tareas de aprovisionamiento de la aplicación.
- **deploy.yml**: contiene el despliegue de la aplicación.

4. Ejecutar el fichero `provision.yml`.
```
ansible-playbook -i hosts provision.yml --user=vagrant --ask-pass
```
5. Ejecutar el fichero `deploy.yml`.
```
ansible-playbook -i hosts deploy.yml --user=vagrant --ask-pass
```

## Despliegue de la aplicación con Docker

1. Instalar `Docker` en el sistema operativo siguiendo las instrucciones de la página oficial.
```
En Windows: https://docs.docker.com/docker-for-windows/install/
En Mac: https://docs.docker.com/docker-for-mac/install/
En Ubuntu: https://docs.docker.com/engine/install/ubuntu/
```
2. Comprobar que el proyecto funciona correctamente.

3. Crear un fichero denominado `Dockerfile` que contendrá lo siguiente:
```
# syntax=docker/dockerfile:1

FROM python:3.9-slim-buster

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt

COPY . .

CMD [ "python3", "manage.py" , "runserver" , "0.0.0.0:8000"]
```
4. Construir la imagen del proyecto.
```
docker build . --tag shield
```
5. Ejecutar la imagen.
```
docker run --publish 8000:8000 shield
```
