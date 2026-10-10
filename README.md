# Odoo 19 + PostgreSQL con Docker Compose

Entorno de desarrollo de **Odoo 19** con **PostgreSQL 18** usando Docker Compose.

- `compose.yml` → configuración base (la que se usa en producción).
- `compose.override.yml` → configuración de desarrollo local (no está en el repo: cada uno crea la suya).

Al ejecutar `docker compose up`, Docker **fusiona automáticamente** ambos archivos.

## Estructura del proyecto

```
odoo-test/
├── compose.yml                 # Base (producción)
├── compose.override.yml        # Desarrollo (NO se sube al repo, la creas en el paso 2)
├── config/
│   ├── odoo.conf.template      # Plantilla de configuración (SÍ está en el repo)
│   └── odoo.conf               # Tu configuración local (NO se sube, la creas en el paso 3)
├── addons/                     # Tus módulos personalizados (la creas en el paso 4)
├── filestore/                  # Adjuntos de Odoo (bind mount, solo en desarrollo; paso 4)
├── sessions/                   # Sesiones web (bind mount, solo en desarrollo; paso 4)
└── secrets/
    └── odoo-db-password.txt    # Contraseña de PostgreSQL (NO se sube, la creas en el paso 1)
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
    command: --dev=all --workers=0 -d odoo
```

### 3. Crea la configuración local de Odoo

Usa `config/odoo.conf.template` como base para crear `config/odoo.conf` y sustituye
`admin_passwd` por una contraseña maestra fuerte. Este archivo no se sube al repositorio.

### 4. Crea las carpetas de datos locales

```bash
mkdir -p addons filestore sessions
```

> **Por qué:** Git no guarda carpetas vacías, así que tras clonar el repo no existen.
> Si no las creas, Docker las genera al montar los volúmenes (en Linux quedan como
> propiedad de `root` y no podrás añadir módulos en `addons/` sin `sudo`).

### 5. Inicializa la base de datos (solo la primera vez)

```bash
docker compose run --rm web odoo -d odoo -i base --stop-after-init
```

### 6. Arranca el entorno

```bash
docker compose up -d
```

La primera vez tarda un poco porque se descargan las imágenes. La base de datos `odoo`
ya la creaste en el paso 5; si te lo saltaste, Odoo arrancará pero mostrará un error
porque no tiene ninguna base de datos que servir (vuelve al paso 5).

### 7. Accede a Odoo

Abre http://localhost:8069

- **Usuario:** `admin`
- **Contraseña:** `admin` (es la contraseña por defecto de una instalación recién
  inicializada; cámbiala desde las preferencias del usuario en cuanto entres)

## Comandos útiles

| Comando | Descripción |
|---|---|
| `docker compose up -d` | Arranca los contenedores (modo desarrollo) |
| `docker compose down` | Detiene y elimina los contenedores |
| `docker compose down -v` | Igual, pero **borra también los volúmenes** (la base de datos se pierde) |
| `docker compose logs -f web` | Muestra los logs de Odoo en tiempo real |
| `docker compose config` | Muestra la configuración fusionada final (muy útil para depurar) |
| `docker compose -f compose.yml up -d` | Arranca **solo con la configuración de producción** (ignora el override) |
| `docker compose run --rm web odoo -d odoo -i base --stop-after-init` | Inicializa la base de datos (solo la primera vez) |

## ¿Cómo funciona la fusión de archivos?

`compose.yml` define la base y `compose.override.yml` la modifica para desarrollo:

- **`volumes`**: se fusionan por la ruta destino. `./filestore:/var/lib/odoo/filestore` y
  `./sessions:/var/lib/odoo/sessions` se **añaden** al volumen `odoo-web-data:/var/lib/odoo`.
- **`command`**: se **sustituye** entero → en desarrollo se añade `--dev=all --workers=0`.

### ¿Qué hace `--dev=all`?

Activa el modo desarrollador de Odoo: recarga automáticamente el código Python y las
plantillas al modificar los archivos en `addons/`, sin reiniciar el contenedor.
`--workers=0` es necesario para que esta recarga funcione.

> En producción **no** se usa `--dev=all` (consume más recursos y expone información de depuración).

## Una sola base de datos

La configuración está pensada para trabajar con **una única base de datos** (`odoo`),
como en un despliegue real: Odoo arranca con `-d odoo` y `config/odoo.conf` incluye
`list_db = False`, así que no aparece el selector de bases de datos al entrar.

Si en clase quieres que cada ejercicio use su propia base de datos:

1. Quita `-d odoo` del `command` (en `compose.yml` y en tu `compose.override.yml`).
2. Cambia a `list_db = True` en `config/odoo.conf`.
3. Gestiona las bases de datos desde http://localhost:8069/web/database/manager
   (la contraseña maestra es el `admin_passwd` de tu `config/odoo.conf`).

## Añadir tus propios módulos

Coloca tus módulos dentro de la carpeta `addons/`. Se montan en el contenedor en
`/mnt/extra-addons`, que ya forma parte del `addons_path` de Odoo. Tras añadir un módulo
nuevo, actualiza la lista de aplicaciones desde la interfaz de Odoo (modo desarrollador →
*Apps → Update Apps List*).

> **Aviso esperado:** mientras `addons/` esté vacía verás en los logs
> `option addons_path, invalid addons directory '/mnt/extra-addons', skipped`.
> Odoo 19 solo acepta una carpeta de addons si contiene al menos un módulo válido
> (con `__manifest__.py` y `__init__.py`); el aviso desaparece en cuanto creas el primero.

## ¿Dónde están mis datos?

| Dato | Dónde vive | ¿Sobrevive a `down -v`? |
|---|---|---|
| Base de datos | Volumen Docker `odoo-db-data` | ❌ No |
| Otros datos de Odoo | Volumen Docker `odoo-web-data` | ❌ No |
| Adjuntos (filestore) | `./filestore` (en tu repo) | ✅ Sí |
| Sesiones | `./sessions` (en tu repo) | ✅ Sí |
| Módulos propios | `./addons` (en tu repo) | ✅ Sí |

## Documentación adicional

- [README-avanzado.md](README-avanzado.md) → migraciones de base de datos (actualizar
  módulos y cambios de versión), réplica de producción en local (filestore, neutralización),
  réplica vs datos de prueba, y qué volúmenes usa cada fichero compose.
