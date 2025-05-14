# Directus Development Environment with code-server and Docker

Este repositorio contiene los archivos y pasos necesarios para levantar un entorno de desarrollo de Directus dentro de un contenedor Docker basado en code‑server (Coolify).

---

## Tabla de contenidos

1. [Requisitos](#requisitos)
2. [Dockerfile](#dockerfile)
3. [Variables de entorno de code-server](#variables-de-entorno-de-code-server)
4. [Construcción de la imagen Docker](#construcción-de-la-imagen-docker)
5. [Configuración de Directus](#configuración-de-directus)

   * [Clonar el proyecto](#clonar-el-proyecto)
   * [Archivo `.env` con SQLite](#archivo-env-con-sqlite)
   * [Creación de carpetas y permisos](#creación-de-carpetas-y-permisos)
   * [Bootstrap y arranque](#bootstrap-y-arranque)
6. [Uso](#uso)
7. [Consideraciones adicionales](#consideraciones-adicionales)

---

## Requisitos

* Docker
* (Opcional) Docker Compose
* Conexión a internet para descargar imágenes base y dependencias

---

## Dockerfile

```dockerfile
FROM ghcr.io/coder/code-server:latest

##########################
# 1) Instalación de SO   #
##########################
USER root
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    curl \
    git \
    build-essential \
    ca-certificates \
    postgresql-client \
  && apt-get clean

##########################
# 2) NVM + Node.js + npm #
##########################
ENV NVM_DIR=/home/coder/.nvm
RUN mkdir -p $NVM_DIR \
 && curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash \
 # Inicializar NVM e instalar LTS de Node
 && . $NVM_DIR/nvm.sh \
 && nvm install --lts \
 && nvm alias default 'lts/*' \
 # Hacer accesibles node/npm globalmente
 && ln -s "$NVM_DIR/versions/node/$(ls $NVM_DIR/versions/node)/bin/node" /usr/local/bin/node \
 && ln -s "$NVM_DIR/versions/node/$(ls $NVM_DIR/versions/node)/bin/npm"  /usr/local/bin/npm \
 # Ajustar permisos para el usuario coder
 && chown -R coder:coder $NVM_DIR

##########################
# 3) Herramientas JS     #
##########################
# Usamos Corepack para manejar pnpm/yarn en la versión del proyecto
RUN . $NVM_DIR/nvm.sh \
 && nvm use default \
 && npm install -g corepack \
 && corepack enable \
 && corepack prepare pnpm@latest --activate \
 && npm install -g @directus/cli

##########################
# 4) Configuración user  #
##########################
USER coder
# Añadir NVM a su shell por defecto
RUN echo 'export NVM_DIR="$HOME/.nvm"'            >> ~/.bashrc \
 && echo '[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"' >> ~/.bashrc \
 && echo 'nvm use default'                       >> ~/.bashrc

# Carpeta de trabajo donde clonarás Directus
WORKDIR /home/coder/project

##########################
# 5) (Opcional) CMD por  #
#    defecto para dev    #
##########################
# Si quieres que al arrancar el contenedor Directus se instale y arranque,
# descomenta la siguiente línea. De lo contrario ejecuta manualmente:
#    $ directus bootstrap && directus start
#
# ENTRYPOINT [ "sh", "-c", ". $NVM_DIR/nvm.sh && nvm use default && cd /home/coder/project && directus bootstrap && directus start" ]
```

---

## Variables de entorno

### code-server

Para configurar la autenticación y el acceso sudo en code-server, define estas variables de entorno en tu `docker run` o `docker-compose`:

```yaml
environment:
  - PASSWORD=password            # Contraseña de acceso a la interfaz web
  - SUDO_PASSWORD=sudo_password  # Contraseña para usar sudo en el terminal
```

### Base de datos para Directus

#### PostgreSQL

Si usas PostgreSQL como servicio externo:

```yaml
environment:
  - DB_CLIENT=pg
  - DB_HOST=db
  - DB_PORT=5432
  - DB_DATABASE=directus
  - DB_USER=directus
  - DB_PASSWORD=secret123
```

#### MySQL / MariaDB

Para MySQL o MariaDB:

```yaml
environment:
  - DB_CLIENT=mysql
  - DB_HOST=db
  - DB_PORT=3306
  - DB_DATABASE=directus
  - DB_USER=directus
  - DB_PASSWORD=secret123
```

---

## Construcción de la imagen Docker

```bash
docker build -t directus-code-server .
```

---

## Configuración de Directus

### Clonar el proyecto

```bash
# Dentro de code-server (WORKDIR /home/coder/project)
git clone https://github.com/SlackDevelopers/directus.git .
```

### Archivos `.env`

Puedes usar SQLite, PostgreSQL o MySQL/MariaDB como backend. Crea el archivo `.env` con la configuración deseada:

#### 1) SQLite

```bash
mkdir -p database
cat > .env <<EOF
DB_CLIENT="sqlite3"
DB_FILENAME="./database/data.db"
SECRET="una_clave_muy_larga_y_secreta"
PUBLIC_URL="http://localhost:8055"
PORT=8055
ADMIN_EMAIL="admin@example.com"
ADMIN_PASSWORD="admin123"
EOF
```

#### 2) PostgreSQL

```bash
cat > .env <<EOF
DB_CLIENT="pg"
DB_HOST="db"
DB_PORT=5432
DB_DATABASE="directus"
DB_USER="directus"
DB_PASSWORD="secret123"
SECRET="una_clave_muy_larga_y_secreta"
PUBLIC_URL="http://localhost:8055"
PORT=8055
ADMIN_EMAIL="admin@example.com"
ADMIN_PASSWORD="admin123"
EOF
```

#### 3) MySQL / MariaDB

```bash
cat > .env <<EOF
DB_CLIENT="mysql"
DB_HOST="db"
DB_PORT=3306
DB_DATABASE="directus"
DB_USER="directus"
DB_PASSWORD="secret123"
SECRET="una_clave_muy_larga_y_secreta"
PUBLIC_URL="http://localhost:8055"
PORT=8055
ADMIN_EMAIL="admin@example.com"
ADMIN_PASSWORD="admin123"
EOF
```

### Creación de carpetas y permisos

```bash
# Dentro de /home/coder/project\mkdir -p database uploads extensions
chown -R coder:coder database uploads extensions
chmod -R 755 database uploads extensions
```

### Bootstrap y arranque

```bash
# Cargar NVM y usar Node.js
. ~/.nvm/nvm.sh && nvm use default

# Ejecutar bootstrap (solo primera vez)
npx directus bootstrap

# Levantar el servidor
npx directus start
```

El servidor quedará corriendo en `http://localhost:8055`.


## Uso

1. Navega a `http://localhost:8055`.
2. Ingresa con el usuario y contraseña definidos en `ADMIN_EMAIL` / `ADMIN_PASSWORD`.
3. Empieza a crear colecciones, roles y contenido.

---

## Consideraciones adicionales

* **Spatialite**: si necesitas soporte de geometría, instala `libsqlite3-mod-spatialite`.
* **Puerto en uso**: modifica `PORT` en `.env` o libera el puerto.
* **Persistencia**: todos los datos de Directus quedan en `./database/data.db`, sube o respalda ese fichero.

---

¡Listo! Ya tienes un entorno completo para desarrollar y contribuir a Directus dentro de code-server y Docker.

# Desde el WORKDIR de tu proyecto
. ~/.nvm/nvm.sh && nvm use default

# Si usas la CLI oficial:
npx directus build
npx directus start

# O, en monorepo:
pnpm install
pnpm --filter app build
pnpm --filter api cli bootstrap
pnpm --recursive dev
