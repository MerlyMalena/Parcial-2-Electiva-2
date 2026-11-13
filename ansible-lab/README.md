# Laboratorio de Ansible con Docker

## Objetivo

Automatizar la configuración de dos servidores Ubuntu 24.04 en Docker con Ansible: consultar su nombre y sistema operativo, crear `/tmp/ansible-demo/info.txt`, generar los subdirectorios `logs`, `backup` y `config` mediante un `loop`, y mostrar un mensaje condicionado a que la distribución sea Ubuntu.

## 1. Requisitos

- Docker con Docker Compose v2 y un motor de contenedores Linux en ejecución.
- Ansible instalado en el equipo de control Linux. En Windows, ejecutar los comandos del laboratorio dentro de Ubuntu en WSL, con acceso a Docker.
- La colección `community.docker`, que proporciona la conexión `docker`. Esta conexión usa la CLI de Docker y no requiere SSH entre el equipo y los contenedores.

En Docker Desktop, la integración habitual se activa en **Settings > Resources > WSL Integration > Ubuntu > Apply & restart**.

Si Docker Desktop ya funciona en Windows y su integración con WSL no está disponible, se puede usar la CLI de Windows desde la misma terminal de Ubuntu. Este fue el método utilizado para ejecutar esta entrega:

```bash
mkdir -p /tmp/ansible-lab-bin
ln -sf "$(command -v docker.exe)" /tmp/ansible-lab-bin/docker
export PATH="/tmp/ansible-lab-bin:$PATH"
docker version
```

Mantener abierta esa terminal para los pasos siguientes. El enlace es temporal y no modifica la configuración de Docker Desktop. Requiere que `command -v docker.exe` encuentre la CLI de Windows.

Si Ansible aún no está instalado, ejecutar en Ubuntu:

```bash
sudo apt update
sudo apt install -y pipx
pipx install --include-deps ansible
pipx ensurepath
```

Abrir una nueva terminal y comprobar:

```bash
docker version
docker compose version
ansible --version
ansible-galaxy collection list community.docker
```

Si falta la colección:

```bash
ansible-galaxy collection install community.docker
```

## 2. Entrar al laboratorio

Desde la raíz del repositorio:

```bash
cd ansible-lab
```

Los comandos siguientes se ejecutan en esa carpeta.

## 3. Iniciar los contenedores

```bash
docker compose up -d
docker ps
```

Deben aparecer `server1` y `server2` en estado `Up`.

## 4. Instalar Python en ambos servidores

Ejecutar directamente desde el equipo de control; no hace falta abrir una sesión interactiva:

```bash
docker exec server1 bash -c 'apt update && DEBIAN_FRONTEND=noninteractive apt install -y python3'
docker exec server2 bash -c 'apt update && DEBIAN_FRONTEND=noninteractive apt install -y python3'
```

## 5. Comprobar el inventario y la conexión

```bash
ansible-inventory -i inventory.ini --graph
ansible docker -i inventory.ini -m ping
```

Ambos servidores deben responder con `SUCCESS` y `"ping": "pong"`.

## 6. Ejecutar el playbook

```bash
ansible-playbook -i inventory.ini --syntax-check playbook.yml
ansible-playbook -i inventory.ini playbook.yml
```

El playbook recopila los datos de los contenedores con `gather_facts`. `inventory_hostname` identifica al servidor como `server1` o `server2`; el hostname interno de Docker puede ser diferente y también se muestra en `info.txt`.

La tarea `copy` crea el archivo y escribe su contenido en una sola operación. Los módulos `file` y `copy` permiten repetir la ejecución sin cambiar recursos que ya tengan el estado solicitado.

## 7. Verificar el resultado

```bash
docker exec server1 cat /tmp/ansible-demo/info.txt
docker exec server2 cat /tmp/ansible-demo/info.txt
docker exec server1 ls -l /tmp/ansible-demo
docker exec server2 ls -l /tmp/ansible-demo
```

Cada contenedor debe tener este contenido:

```text
/tmp/ansible-demo/
├── info.txt
├── logs/
├── backup/
└── config/
```

Volver a ejecutar el playbook debe terminar con `changed=0`, `unreachable=0` y `failed=0` en ambos servidores.

## 8. Captura de la ejecución

Ejecución verificada el **13 de noviembre de 2026**, con Ansible Core 2.20.1, la colección `community.docker` 5.0.4 y dos contenedores Ubuntu 24.04.

![Captura de la salida real del playbook](evidencias/ejecucion-playbook.png)

La imagen es una captura en navegador del registro real guardado por Ansible. Se incluyen también los registros completos para revisar los resultados:

- [Primera ejecución](evidencias/ejecucion-playbook.txt): ambos servidores terminaron con `ok=7`, `changed=3`, `unreachable=0` y `failed=0`.
- [Segunda ejecución](evidencias/segunda-ejecucion.txt): ambos servidores terminaron con `ok=7`, `changed=0`, `unreachable=0` y `failed=0`.

Se comprobó también que `info.txt` identifica correctamente a cada servidor y a Ubuntu 24.04, y que existen los tres subdirectorios en ambos contenedores.

## 9. Subir a GitHub

Repositorio de entrega: [Parcial-2-Electiva-2](https://github.com/MerlyMalena/Parcial-2-Electiva-2).

Para publicar desde cero en un repositorio vacío, ejecutar desde la carpeta que contiene `ansible-lab`:

```bash
git init
git add ansible-lab
git commit -m "Agregar laboratorio Ansible con dos servidores Docker"
git branch -M main
git remote add origin https://github.com/MerlyMalena/Parcial-2-Electiva-2.git
git push -u origin main
```

## 10. Detener el laboratorio

```bash
docker compose stop
```

Para eliminar los contenedores y su red:

```bash
docker compose down
```

Al eliminarlos también se pierde el contenido creado dentro de ellos. Para repetir la práctica, volver a iniciarlos, instalar Python y ejecutar el playbook.

## Documentación

- [Conexión Docker de Ansible](https://docs.ansible.com/projects/ansible/latest/collections/community/docker/docker_connection.html).
- [Instalación de Ansible](https://docs.ansible.com/projects/ansible/latest/intro_installation.html).
