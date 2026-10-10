# Guía avanzada: migraciones, réplicas y datos de prueba

Documentación complementaria al [README.md](README.md) con temas _good to know_
para trabajar con este entorno: cómo migrar la base de datos, cómo traer datos de
producción a local y cuándo conviene cada cosa.

## 1. Migraciones de base de datos

"Migrar" en Odoo significa dos cosas muy distintas:

| Tipo                                                 | Ejemplo                                                                       | ¿Soportado con esta configuración?                          |
| ---------------------------------------------------- | ----------------------------------------------------------------------------- | ----------------------------------------------------------- |
| **Actualización de módulos** (misma versión de Odoo) | Aplicar cambios de código de `addons/` o de una imagen `odoo:19` más reciente | ✅ Sí, sin tocar nada                                       |
| **Migración de versión mayor** (p. ej. Odoo 18 → 19) | Mover la DB `odoo` a otra versión mayor                                       | ❌ Odoo no la hace in-place; requiere herramientas externas |

### 1.1 Actualización de módulos (misma versión)

No hace falta cambiar nada en `compose.yml` ni en `odoo.conf`:

```bash
docker compose stop web
docker compose run --rm web odoo -d odoo -u all --stop-after-init
docker compose up -d
```

- Para un solo módulo: `-u mi_modulo`.
- `docker compose run` **sustituye** el `command:` de los compose (el `-d odoo` y el
  `--dev=all --workers=0` del override no interfieren).
- Las credenciales de la DB se inyectan igual que en el arranque normal (variables de
  entorno + secreto), y se sigue leyendo `/etc/odoo/odoo.conf`.
- Con `--stop-after-init` la actualización se ejecuta durante el arranque, antes de
  levantar workers, así que el `workers = 2` de `odoo.conf` no molesta.

**Scripts de migración en módulos propios:** dentro de un módulo puedes crear
`migrations/<version>/{pre|post|end}-*.py` con una función `migrate(cr, version)`;
Odoo los ejecuta automáticamente durante el `-u`. Es el mecanismo oficial
([documentación](https://www.odoo.com/documentation/19.0/developer/reference/upgrades/upgrade_scripts.html)).

### 1.2 Migración de versión mayor (p. ej. 18 → 19)

Cambiar `image: odoo:19` por otra versión **no** migra la base de datos: Odoo no
incluye migrador entre versiones mayores. Opciones reales:

1. **Servicio oficial de Odoo** ([upgrade.odoo.com](https://upgrade.odoo.com),
   requiere Enterprise): subes un dump y te devuelven otro migrado. Los módulos
   personalizados deben estar ya adaptados a la versión destino.
2. **OCA OpenUpgrade** (Community, open source): fork de Odoo con scripts de
   migración. La rama `19.0` ya existe. Requeriría cambios en la configuración:
   - Añadir `openupgrade_framework` y `openupgrade_scripts` al `addons_path`.
   - Añadir `server_wide_modules = base,web,openupgrade_framework` a `odoo.conf`
     (o `--load` por CLI).
   - Ejecutarlo siempre sobre una **copia** de la DB, nunca sobre producción.
3. Las migraciones son versión a versión: saltar de 16 a 19 implica encadenar
   16→17→18→19.

### 1.3 ¿Es distinto en local y en producción?

El procedimiento es **idéntico**; solo cambia la invocación de compose:

|                          | Local                                                      | Producción                                                                 |
| ------------------------ | ---------------------------------------------------------- | -------------------------------------------------------------------------- |
| Ficheros compose         | `compose.yml` + `compose.override.yml` (se fusionan solos) | `docker compose -f compose.yml ...` (el override no se sube al repo)       |
| `--dev=all`, `workers=0` | Irrelevante: `run` sustituye todo el comando               | `workers = 2` del conf aplica, pero no afecta a `-u ... --stop-after-init` |
| Antes de migrar          | Backup opcional                                            | **Backup obligatorio** → `stop web` → migrar → `up -d`                     |

## 2. Traer datos de producción a local (réplica)

### 2.1 El filestore SÍ tiene sentido; las sesiones NO

Un dump SQL (`pg_dump`) **no incluye los adjuntos**. Odoo guarda los binarios
(imágenes, documentos...) en `<data_dir>/filestore/<nombre_db>/` y la DB solo guarda
metadatos (`ir_attachment`). Por eso el gestor de bases de datos de Odoo ofrece
_"zip (includes filestore)"_ frente a _"pg_dump (without filestore)"_.

|               | ¿Copiar de prod a local? | ¿Tiene sentido?                                                                                          |
| ------------- | ------------------------ | -------------------------------------------------------------------------------------------------------- |
| **Filestore** | ✅ Sí                    | ✅ Necesario para una réplica fiel (si no, imágenes/adjuntos rotos)                                      |
| **Sesiones**  | ✅ Técnicamente posible  | ❌ No: son efímeras, caducan solas y es llevar tokens de autenticación vivos a una máquina de desarrollo |

> En Odoo 19 la contraseña maestra (`admin_passwd`) **ya no sirve** para iniciar
> sesión como cualquier usuario (verificado en el código fuente de `res_users.py`:
> solo contraseña de usuario o API keys). Para entrar en la réplica, resetea una
> contraseña (paso 4).

### 2.2 Procedimiento completo

**En producción** (el filestore vive dentro del volumen con nombre `odoo-web-data`):

```bash
# Dump de la base de datos
docker compose exec db pg_dump -U odoo odoo > backup.sql

# Extraer el filestore del volumen
docker run --rm -v odoo-test_odoo-web-data:/data -v "$PWD:/backup" \
  alpine tar czf /backup/filestore.tgz -C /data filestore
```

Copia ambos archivos a tu máquina (`scp`, `rsync`...).

**En local:**

```bash
# 1. Descomprimir: debe quedar ./filestore/odoo/ (la subcarpeta = nombre de la DB)
tar xzf filestore.tgz

# 2. Restaurar la base de datos
docker compose exec db psql -U odoo -d postgres -c "DROP DATABASE odoo WITH (FORCE);"
docker compose exec db psql -U odoo -d postgres -c "CREATE DATABASE odoo OWNER odoo;"
cat backup.sql | docker compose exec -T db psql -U odoo -d odoo

# 3. NEUTRALIZAR (paso crítico, ver 2.3)
docker compose run --rm web odoo neutralize -d odoo

# 4. Resetear la contraseña de admin para entrar
docker compose run --rm web odoo shell -d odoo
>>> env['res.users'].browse(2).password = 'admin'
>>> env.cr.commit()
```

`./sessions/` no se toca: se llenará con sesiones locales nuevas al iniciar sesión.

### 2.3 Neutralizar ≠ anonimizar

`odoo neutralize` desactiva acciones planificadas (cron), correo saliente, pasarelas
de pago, métodos de envío, sincronización bancaria y tokens IAP, y muestra un banner
rojo de "base de datos neutralizada"
([documentación](https://www.odoo.com/documentation/19.0/administration/neutralized_database.html)).
Sin neutralizar, la copia local podría **enviar correos reales a clientes** o ejecutar
procesos automáticos contra servicios externos con credenciales de producción.

Pero la réplica **sigue conteniendo datos personales reales** (empleados, clientes,
nóminas...). Llevarla a máquinas de desarrollo (o peor, de alumnos) puede ser un
problema de RGPD. Si necesitas datos realistas sin sensibilidad, anonimiza tras
restaurar (SQL u `odoo shell` para alterar nombres/emails) o usa datos demo.

## 3. ¿Réplica o datos de prueba?

### 3.1 ¿Dónde vive cada cosa?

**Los usuarios viven en la base de datos**: `res.users` (login, contraseña hasheada,
grupos) + su `res.partner` asociado (nombre, email...). Nada de los usuarios está en
el código ni en el filestore.

| Qué                                                             | Dónde vive                                                                | ¿Sobrevive a restaurar solo la DB? |
| --------------------------------------------------------------- | ------------------------------------------------------------------------- | ---------------------------------- |
| Usuarios, contraseñas (hash), permisos                          | **DB** (`res_users`, `res_groups`)                                        | ✅                                 |
| Datos de negocio (facturas, productos, CRM...)                  | **DB**                                                                    | ✅                                 |
| Configuración (ajustes, secuencias, cron, servidores de correo) | **DB** (`ir_config_parameter`, datos de módulos)                          | ✅                                 |
| Menús, vistas, informes, reglas de seguridad                    | Definidos en **código** (XML/CSV) y cargados en DB; se regeneran con `-u` | ✅                                 |
| Lógica de negocio                                               | **Código** (`addons/` y núcleo de Odoo)                                   | n/a                                |
| Adjuntos e imágenes                                             | **Filestore**                                                             | ❌ rotos sin él                    |
| Sesiones de login                                               | **Directorio sessions** (efímero)                                         | irrelevante                        |

### 3.2 Cuándo usar cada cosa

**Datos demo/fake para el día a día:**

- Desarrollo de módulos: la DB se crea con datos demo por defecto
  (`--without-demo=all` los desactiva); es rápida, desechable y determinista.
- Tests automáticos (`--test-enable`): generan sus propios datos; la réplica no
  aporta nada.
- Clase / compartir: sin datos sensibles y todo el mundo con el mismo dataset.
- Pruebas de carga: Odoo incluye el comando `populate`
  (`odoo populate --models res.partner --size medium`) para generar datos sintéticos
  en masa.

**Réplica neutralizada para validar antes de producción:**

- Probar scripts de migración de módulos propios: los datos reales acumulan formas
  raras (NULLs heredados, registros huérfanos, datos anteriores a una restricción)
  que los datos demo nunca tienen.
- Depurar errores que solo ocurren con datos reales.
- Validación previa al despliegue (es exactamente lo que Odoo.sh hace con sus ramas
  de staging).

### 3.3 Recomendación práctica

1. **Desarrollo** (tu `compose.override.yml`): DB desechable con datos demo.
2. **Tests automáticos**: `--test-enable`, fixtures autogenerados.
3. **Puerta de staging** antes de subir a producción: restaurar dump + filestore →
   `odoo neutralize` → `-u mi_modulo` → probar → desplegar.

## 4. ¿Qué datos usa cada fichero compose?

Solo con `compose.yml` (producción), **todo el dato vive en los dos volúmenes con
nombre**; las carpetas `./filestore` y `./sessions` del repo se ignoran por completo:

| Dato                                             | Producción (`-f compose.yml`)           | Local (con override)                                                 |
| ------------------------------------------------ | --------------------------------------- | -------------------------------------------------------------------- |
| PostgreSQL                                       | Volumen `odoo-db-data`                  | Mismo volumen `odoo-db-data`                                         |
| Filestore                                        | Dentro del volumen `odoo-web-data`      | `./filestore` (bind mount que **tapa** el subdirectorio del volumen) |
| Sesiones                                         | Dentro del volumen `odoo-web-data`      | `./sessions` (bind mount)                                            |
| Código (`addons/`), config (`config/`), secretos | Bind mount del repo: **igual en ambos** | Igual                                                                |

Dos matices importantes:

1. **Los volúmenes son mundos separados.** Los del servidor de producción y los de tu
   máquina nunca se sincronizan (incluso llevan el prefijo del directorio del
   proyecto, p. ej. `odoo-test_odoo-web-data`). El único puente es copiar a mano (el
   procedimiento de réplica de la sección 2).
2. **Ojo al alternar en local:** la DB es la misma en ambos modos (`odoo-db-data`),
   pero el filestore cambia de sitio. Si subes adjuntos con `docker compose up` y
   luego arrancas con `docker compose -f compose.yml up` (como sugiere el README
   para probar la config de producción), Odoo servirá la **misma base de datos con
   otro filestore** (probablemente vacío): imágenes rotas sin que se haya perdido
   nada. Y al revés igual.
