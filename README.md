# Odoo 19 + PostgreSQL con Docker Compose

Entorno de desarrollo de **Odoo 19** con **PostgreSQL 18** usando Docker Compose.

- `compose.yml` → configuración base (la que se usa en producción).
- `compose.override.yml` → configuración de desarrollo local (no está en el repo: cada uno crea la suya).

Al ejecutar `docker compose up`, Docker **fusiona automáticamente** ambos archivos.

## Estructura del proyecto

```
odoo-test/
├── compose.yml              # Base (producción)
├── compose.override.yml     # Desarrollo (NO se sube al repo, la creas tú)
├── addons/                  # Tus módulos personalizados (se montan en /mnt/extra-addons)
├── filestore/               # Adjuntos de Odoo (bind mount, solo en desarrollo)
├── sessions/                # Sesiones web (bind mount, solo en desarrollo)
└── secrets/
    └── odoo-db-password.txt # Contraseña de PostgreSQL (NO se sube al repo)
```

## Requisitos

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) instalado y en ejecución.

## Puesta en marcha (primera vez)

### 1. Crea el archivo de contraseña de PostgreSQL

```bash
mkdir -p secrets
echo "odoo" > secrets/odoo-db-password.txt
```

> **Importante:** si este archivo no existe, `docker compose up` fallará con un error del tipo
> `bind source path does not exist: .../secrets/odoo-db-password.txt`.
> En Windows puedes crear el archivo con cualquier editor de texto: debe contener solo `odoo`.

### 2. Crea el archivo `compose.override.yml`

Crea un archivo llamado `compose.override.yml` en la raíz del proyecto con este contenido:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/compose-spec/compose-spec/master/schema/compose-spec.json
# Desarrollo local: se fusiona automáticamente con compose.yml al ejecutar `docker compose up`
services:
  web:
    volumes:
      - ./filestore:/var/lib/odoo/filestore
      - ./sessions:/var/lib/odoo/sessions
    environment:
      - ENVIRONMENT=development
    command: --dev=all --workers=0 -d odoo -i base
```

### 3. Crea las carpetas de datos locales

```bash
mkdir -p filestore sessions
```

### 4. Arranca el entorno

```bash
docker compose up -d
```

La primera vez tarda un poco (descarga las imágenes y crea la base de datos `odoo`).

### 5. Accede a Odoo

Abre http://localhost:8069

- **Usuario:** `admin`
- **Contraseña:** `admin`

## Comandos útiles

| Comando | Descripción |
|---|---|
| `docker compose up -d` | Arranca los contenedores (modo desarrollo) |
| `docker compose down` | Detiene y elimina los contenedores |
| `docker compose down -v` | Igual, pero **borra también los volúmenes** (la base de datos se pierde) |
| `docker compose logs -f web` | Muestra los logs de Odoo en tiempo real |
| `docker compose config` | Muestra la configuración fusionada final (muy útil para depurar) |
| `docker compose -f compose.yml up -d` | Arranca **solo con la configuración de producción** (ignora el override) |

## ¿Cómo funciona la fusión de archivos?

`compose.yml` define la base y `compose.override.yml` la modifica para desarrollo:

- **`volumes`**: se fusionan por la ruta destino. `./filestore:/var/lib/odoo/filestore` y
  `./sessions:/var/lib/odoo/sessions` se **añaden** al volumen `odoo-web-data:/var/lib/odoo`.
- **`environment`**: se fusiona por clave → `ENVIRONMENT=development` sustituye a `production`.
- **`command`**: se **sustituye** entero → en desarrollo se añade `--dev=all --workers=0`.

### ¿Qué hace `--dev=all`?

Activa el modo desarrollador de Odoo: recarga automáticamente el código Python y las
plantillas al modificar los archivos en `addons/`, sin reiniciar el contenedor.
`--workers=0` es necesario para que esta recarga funcione.

> En producción **no** se usa `--dev=all` (consume más recursos y expone información de depuración).

## Añadir tus propios módulos

Coloca tus módulos dentro de la carpeta `addons/`. Se montan en el contenedor en
`/mnt/extra-addons`, que ya forma parte del `addons_path` de Odoo. Tras añadir un módulo
nuevo, actualiza la lista de aplicaciones desde la interfaz de Odoo (modo desarrollador →
*Apps → Update Apps List*).

## ¿Dónde están mis datos?

| Dato | Dónde vive | ¿Sobrevive a `down -v`? |
|---|---|---|
| Base de datos | Volumen Docker `odoo-db-data` | ❌ No |
| Otros datos de Odoo | Volumen Docker `odoo-web-data` | ❌ No |
| Adjuntos (filestore) | `./filestore` (en tu repo) | ✅ Sí |
| Sesiones | `./sessions` (en tu repo) | ✅ Sí |
| Módulos propios | `./addons` (en tu repo) | ✅ Sí |
