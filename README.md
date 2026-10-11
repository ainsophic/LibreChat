# Egeo

**Egeo** es la distribución personalizada de LibreChat que desarrolla Ainsophic para ofrecer una experiencia de chat de IA propia, con identidad visual, configuración, endpoints y despliegue gestionados por el equipo.

> **Estado:** guía de desarrollo e instalación inicial. Antes de cada actualización importante, contrastar los comandos y archivos con la versión exacta de LibreChat que se vaya a integrar.

Egeo parte del código de [LibreChat](https://github.com/LibreChat-AI/LibreChat). Es un proyecto derivado y mantenido de forma independiente; no es una distribución oficial del equipo de LibreChat.

---

## Índice

- [1. Objetivos y principios](#1-objetivos-y-principios)
- [2. Arquitectura de repositorios y ramas](#2-arquitectura-de-repositorios-y-ramas)
- [3. Requisitos previos](#3-requisitos-previos)
- [4. Crear el repositorio de Egeo](#4-crear-el-repositorio-de-egeo)
- [5. Configurar los remotos de Git](#5-configurar-los-remotos-de-git)
- [6. Flujo diario de desarrollo](#6-flujo-diario-de-desarrollo)
- [7. Personalizar la marca Egeo](#7-personalizar-la-marca-egeo)
- [8. Configuración de la aplicación](#8-configuración-de-la-aplicación)
- [9. Conectar Ainsophic Intelligence](#9-conectar-ainsophic-intelligence)
- [10. Ejecutar Egeo en desarrollo](#10-ejecutar-egeo-en-desarrollo)
- [11. Construir y probar la imagen Docker de Egeo](#11-construir-y-probar-la-imagen-docker-de-egeo)
- [12. Actualizar Egeo desde LibreChat](#12-actualizar-egeo-desde-librechat)
- [13. Resolver conflictos de actualización](#13-resolver-conflictos-de-actualización)
- [14. Seguridad y datos](#14-seguridad-y-datos)
- [15. Pruebas antes de publicar](#15-pruebas-antes-de-publicar)
- [16. Estrategia de publicación y despliegue](#16-estrategia-de-publicación-y-despliegue)
- [17. Convenciones de trabajo](#17-convenciones-de-trabajo)
- [18. Problemas frecuentes](#18-problemas-frecuentes)
- [19. Fuentes oficiales](#19-fuentes-oficiales)

---

## 1. Objetivos y principios

Egeo debe permitir al equipo:

1. Mantener una experiencia de producto identificada como **Egeo**, sin depender de cambios manuales sobre una instalación en producción.
2. Incorporar correcciones y nuevas funcionalidades de LibreChat de manera controlada.
3. Conectar modelos propios y servicios de inferencia mediante APIs compatibles, sin exponer claves privadas en el navegador.
4. Separar cambios de marca, configuración, funcionalidades de producto y operación.
5. Construir y desplegar imágenes reproducibles desde commits identificables.

### Principios técnicos

- **Upstream primero:** conservar el historial del proyecto original y actualizarlo mediante Git.
- **Cambios pequeños:** implementar cada modificación en una rama específica.
- **Configuración fuera del código cuando corresponda:** los secretos, URLs y valores propios de cada entorno no deben quedar hardcodeados.
- **Producción desde una imagen propia:** no asumir que una imagen pública de desarrollo contiene los cambios locales de Egeo.
- **Compatibilidad verificada:** una actualización se considera terminada después de ejecutar pruebas, no solo después de resolver los conflictos de Git.
- **Seguridad por defecto:** TLS en producción, secretos persistentes, acceso administrativo restringido, copias de seguridad y control de registros.

## 2. Arquitectura de repositorios y ramas

Se recomienda tener un único repositorio Git de Egeo, con el repositorio oficial configurado como remoto `upstream`.

```text
LibreChat-AI/LibreChat (upstream)
            |
            | fetch / actualización controlada
            v
   main (espejo de upstream)
            |
            | merge + pruebas
            v
       egeo-prod  ------------------> imagen Docker de Egeo
            ^                                  |
            |                                  v
       feature/*                        entorno de staging
            |                                  |
            +---- Pull Request -----------------+
                                               |
                                               v
                                           producción
```

### Ramas

| Rama | Propósito | Regla |
|---|---|---|
| `main` | Reflejar el `main` oficial de LibreChat. | No incorporar personalizaciones de Egeo ni commits propios. |
| `egeo-prod` | Rama de integración estable de Egeo. | Recibe cambios revisados y aprobados; es la base de los despliegues. |
| `feature/<tema>` | Funcionalidad o personalización aislada. | Se crea desde `egeo-prod` y vuelve mediante Pull Request. |
| `fix/<tema>` | Corrección puntual. | Incluye una solución acotada y pruebas cuando corresponda. |
| `hotfix/<tema>` | Corrección urgente de producción. | Se integra en `egeo-prod` y luego se sincroniza con las ramas necesarias. |

Ejemplos de nombres:

- `feature/egeo-branding`
- `feature/ainsophic-endpoint`
- `feature/authentik-sso`
- `fix/login-redirect`
- `hotfix/security-patch`

**Importante:** `main` no debe ser una segunda rama de desarrollo. Si se incorporan personalizaciones allí, `git merge --ff-only upstream/main` puede dejar de funcionar y se pierde la separación simple entre código original y producto derivado.

## 3. Requisitos previos

### Herramientas necesarias

- Cuenta y repositorio en GitHub con permisos de escritura.
- Git.
- Docker Engine y Docker Compose v2 para ejecutar contenedores.
- Un editor de código y terminal.

### Para desarrollo local sin construir Docker

- La versión de Node.js indicada por la documentación de la versión de LibreChat que se haya fijado. La guía oficial consultada actualmente indica Node.js `24.16.0`; comprobarlo de nuevo al cambiar de versión.
- npm, cuya versión debe ser compatible con el `package.json` y `package-lock.json` de esa revisión.
- MongoDB accesible desde el entorno local.
- MeiliSearch y otros servicios auxiliares si se habilitan funciones que los requieran.

### Conocimientos recomendados

Git básico, ramas, Pull Requests, variables de entorno, Docker Compose, lectura de logs y resolución de conflictos.

## 4. Crear el repositorio de Egeo

### Paso 1. Crear un fork desde la fuente oficial

Abrir el repositorio oficial:

<https://github.com/LibreChat-AI/LibreChat>

Usar **Fork** desde GitHub y elegir la organización o cuenta propietaria de Egeo. Si la interfaz lo permite, usar `Egeo` como nombre del repositorio. Si se conserva el nombre `LibreChat`, los comandos de este manual siguen siendo válidos: solo cambia la URL de `origin`.

### Paso 2. Clonar vuestro repositorio

Reemplazar `TU_ORG` por la organización o usuario real de GitHub. Si el repositorio tiene otro nombre, ajustar el último segmento.

```bash
git clone https://github.com/TU_ORG/Egeo.git
cd Egeo
```

Verificar el estado:

```bash
git status
git branch --show-current
git remote -v
```

En una clonación nueva, `origin` debe apuntar a vuestro repositorio.

### Paso 3. Registrar el repositorio oficial como `upstream`

```bash
git remote add upstream https://github.com/LibreChat-AI/LibreChat.git
git fetch upstream
git remote -v
```

El resultado esperado es similar a:

```text
origin    https://github.com/TU_ORG/Egeo.git (fetch)
origin    https://github.com/TU_ORG/Egeo.git (push)
upstream  https://github.com/LibreChat-AI/LibreChat.git (fetch)
upstream  https://github.com/LibreChat-AI/LibreChat.git (push)
```

`origin` es el repositorio de Egeo. `upstream` es la fuente oficial. En el flujo normal, el equipo publica sus cambios en `origin`, no en el repositorio oficial.

> Si ya existe un remoto llamado `upstream`, no repetir `git remote add`. Revisar con `git remote -v` y corregir su URL, si hiciera falta, con `git remote set-url upstream https://github.com/LibreChat-AI/LibreChat.git`.

### Paso 4. Sincronizar `main` y crear `egeo-prod`

En un fork recién creado, `main` debería corresponder al historial oficial. Sincronizarlo y crear la rama del producto:

```bash
git switch main
git fetch upstream
git merge --ff-only upstream/main
git push origin main

git switch -c egeo-prod
git push -u origin egeo-prod
```

Si `git merge --ff-only` informa que no es posible avanzar rápidamente, **detenerse y revisar el estado de las ramas**; no ejecutar `reset --hard` ni forzar un push como solución automática.

Verificar la configuración final:

```bash
git branch -vv
git remote -v
git status
```

## 5. Configurar los remotos de Git

- `origin`: repositorio propio de Egeo, donde se publican las ramas de trabajo.
- `upstream`: repositorio oficial de LibreChat, desde donde se reciben cambios.

Comprobarlos siempre antes de sincronizar:

```bash
git remote -v
```

Si se trabaja sobre una clonación que no se creó desde vuestro fork, revisar con especial cuidado `origin` antes de hacer `git push`.

## 6. Flujo diario de desarrollo

### Paso 1. Actualizar la rama de trabajo

```bash
git switch egeo-prod
git pull --ff-only origin egeo-prod
git status
```

### Paso 2. Crear una rama por cambio

```bash
git switch -c feature/egeo-branding
```

Usar un nombre relacionado con el cambio; evitar mezclar, por ejemplo, cambios de logos con modificaciones de autenticación o infraestructura.

### Paso 3. Implementar y revisar el cambio

Consultar la estructura real del código antes de modificar archivos, ya que las rutas pueden cambiar entre versiones:

```bash
find client -type f \( -iname '*logo*' -o -iname '*favicon*' -o -iname '*brand*' \)
```

Si está instalado `ripgrep` (`rg`), buscar referencias de marca con:

```bash
rg -n --hidden --glob '!node_modules/**' --glob '!**/dist/**' 'LibreChat|APP_TITLE' client .env.example
```

Revisar los cambios:

```bash
git status
git diff --check
git diff
```

### Paso 4. Crear commits claros

```bash
git add <archivos-modificados>
git commit -m "feat(branding): introduce Egeo identity"
```

Ejemplos de prefijos útiles: `feat`, `fix`, `docs`, `refactor`, `test`, `chore` y `security`.

### Paso 5. Publicar la rama y abrir un Pull Request

```bash
git push -u origin feature/egeo-branding
```

Abrir un Pull Request desde `feature/egeo-branding` hacia `egeo-prod`. El PR debe explicar el problema, la solución, los archivos afectados, la prueba realizada y los riesgos de regresión.

### Paso 6. Integrar y limpiar

Después de aprobar y fusionar el PR:

```bash
git switch egeo-prod
git pull --ff-only origin egeo-prod
git branch -d feature/egeo-branding
```

Borrar la rama remota cuando corresponda desde GitHub o con:

```bash
git push origin --delete feature/egeo-branding
```

No borrar ramas que aún se necesiten para una revisión o despliegue.

## 7. Personalizar la marca Egeo

La personalización de la marca tiene varias capas. Cambiar una sola variable no garantiza que se modifiquen el título de la pestaña, los metadatos, todos los logotipos y cada texto visible.

### 7.1. Crear la configuración de entorno local

```bash
cp .env.example .env
```

Editar `.env` y establecer, como mínimo, el nombre visible de la aplicación y las URLs correctas para el entorno:

```dotenv
APP_TITLE=Egeo
DOMAIN_CLIENT=http://localhost:3080
DOMAIN_SERVER=http://localhost:3080
HELP_AND_FAQ_URL=https://TU_DOMINIO_DE_AYUDA
```

Usar las URLs locales solo durante desarrollo. En producción, reemplazarlas por las URLs HTTPS definitivas. Confirmar en `.env.example` de la revisión elegida que las variables siguen disponibles y conservan el significado esperado.

### 7.2. Logotipos, favicon e iconos

No asumir que todas las rutas son idénticas en cada versión. Localizar los recursos de marca en el código actual:

```bash
find client -type f \( \
  -iname '*logo*' -o \
  -iname '*favicon*' -o \
  -iname '*icon*' -o \
  -iname '*brand*' \
\)
```

Procedimiento recomendado:

1. Identificar el archivo y los componentes que realmente lo utilizan.
2. Crear una copia de seguridad o registrar el archivo en Git antes de sustituirlo.
3. Reemplazar el recurso conservando el nombre y formato cuando resulte compatible, para minimizar los cambios de código.
4. Si se cambia el nombre o formato, actualizar las referencias correspondientes.
5. Comprobar versiones clara y oscura, pantalla de acceso, interfaz principal, favicon, resolución móvil y escalado de SVG/PNG.

Mantener archivos distintos cuando el proyecto use variantes diferentes del logo. No sobreescribir recursos que no sean de la marca sin verificar su función.

### 7.3. Título de pestaña y metadatos

Buscar el título HTML, los metadatos y los lugares donde se muestra el nombre de la aplicación:

```bash
rg -n --hidden --glob '!node_modules/**' --glob '!**/dist/**' \
  '<title>|LibreChat|APP_TITLE|description' client
```

Inspeccionar especialmente el HTML de entrada del cliente, los componentes de marca y cualquier configuración de construcción. En determinadas revisiones, el título HTML puede seguir definido en el código del cliente aunque `APP_TITLE` esté configurado. Si sucede, cambiarlo explícitamente a Egeo y validar la salida compilada.

### 7.4. Textos de interfaz y bienvenida

Buscar referencias visibles a la marca original y clasificarlas antes de cambiarlas. Actualizar solo los textos que corresponden a la identidad del producto, por ejemplo:

- título, bienvenida y mensajes de acceso;
- enlaces de ayuda, privacidad y condiciones;
- información de la aplicación y textos de pie;
- nombre del producto en notificaciones propias, si corresponde.

**No** ejecutar un reemplazo global indiscriminado de `LibreChat`: el nombre puede aparecer en documentación de origen, licencias, dependencias, nombres de configuración, rutas, pruebas o datos internos que no deben cambiarse.

### 7.5. Personalización visual

Antes de modificar estilos de manera extensa, revisar las opciones de configuración disponibles en `librechat.yaml` para la versión elegida. Si se requiere una identidad visual que exceda las opciones soportadas, implementar el cambio en componentes/estilos específicos y mantenerlo en una rama propia. No basar personalizaciones críticas únicamente en selectores CSS frágiles.

## 8. Configuración de la aplicación

### `.env`

Contiene configuración del entorno y secretos. Debe ser privado y no debe subirse al repositorio. Comprobar que Git lo ignore:

```bash
git check-ignore -v .env
```

Si no aparece como ignorado, revisar `.gitignore` antes de seguir. No publicar claves reales en commits, capturas, logs ni incidencias.

### `librechat.yaml`

Es el archivo de configuración avanzada de LibreChat: endpoints personalizados, modelos, interfaz y otras opciones. Partir del ejemplo oficial de la misma revisión:

```bash
cp librechat.example.yaml librechat.yaml
```

Este archivo puede contener valores específicos del despliegue. No asumir que editarlo alcanza: verificar que el servicio API lo reciba en la ruta `/app/librechat.yaml` y reiniciar la aplicación después de cambiarlo. Si el `docker-compose.yml` de esa revisión no lo monta, agregar un volumen en el archivo de override local, no editar la configuración base sin necesidad.

Ejemplo de override para montar la configuración en el servicio API:

```yaml
services:
  api:
    volumes:
      - ./librechat.yaml:/app/librechat.yaml:ro
```

Integrar este fragmento con el Compose de la versión fijada; no reemplazar las demás secciones `volumes` existentes. Si la versión utiliza otro nombre de servicio o ruta, respetar el archivo oficial.

## 9. Conectar Ainsophic Intelligence

La integración es directa cuando el endpoint de inferencia expone una API compatible con OpenAI Chat Completions. LibreChat debe poder alcanzar la URL desde el contenedor del servicio API, no solamente desde el navegador del desarrollador.

### 9.1. Definir variables privadas

Añadir a `.env` los valores del entorno correspondiente, usando una clave de API válida y la URL base real:

```dotenv
AINSOPHIC_API_KEY=REEMPLAZAR_POR_UN_SECRETO_REAL
AINSOPHIC_BASE_URL=https://TU_GATEWAY_O_ENDPOINT/v1
```

No usar esos valores de ejemplo literalmente. No incluir la clave dentro de `librechat.yaml` ni del frontend; se referencia mediante variable de entorno.

### 9.2. Registrar un endpoint personalizado

En `librechat.yaml`, dentro de `endpoints.custom`, configurar un endpoint similar a este:

```yaml
endpoints:
  custom:
    - name: "Ainsophic Intelligence"
      apiKey: "${AINSOPHIC_API_KEY}"
      baseURL: "${AINSOPHIC_BASE_URL}"
      models:
        default:
          - "ID_REAL_DEL_MODELO"
        fetch: false
      titleConvo: true
      titleModel: "ID_REAL_DEL_MODELO"
      modelDisplayLabel: "Egeo · Ainsophic"
```

Reemplazar `ID_REAL_DEL_MODELO` por el identificador exacto que entiende el servicio de inferencia. Mantener `fetch: false` cuando se quieran declarar modelos manualmente o cuando el endpoint no implemente un listado compatible; habilitar la consulta dinámica solo si la API soporta el comportamiento esperado.

Requisitos para este ejemplo:

- La URL base debe ser la raíz correcta de la API, normalmente terminada en `/v1`, no necesariamente la ruta completa de completado.
- La API debe aceptar la autenticación configurada y la ruta compatible que LibreChat construya.
- Si el proveedor usa una API distinta de OpenAI Chat Completions, comprobar la documentación del proveedor y el tipo de endpoint compatible antes de usar esta plantilla.
- En Docker, verificar conectividad, DNS, certificados TLS y reglas de red desde el contenedor `api`.

Después de guardar la configuración, reiniciar Egeo y comprobar que el endpoint y el modelo aparecen en el selector. Ejecutar una conversación de prueba y revisar los logs del backend si falla.

### 9.3. Papel de Kong y Cloudflare

Si el modelo se publica a través de Kong o de un gateway de inferencia, configurar en `AINSOPHIC_BASE_URL` la URL que Egeo puede alcanzar con TLS y autenticación. Kong puede aplicar políticas de acceso, límites y observabilidad de API; Cloudflare AI Gateway puede incorporarse si encaja con el recorrido real de las solicitudes. Evitar encadenar proxies innecesarios y documentar cuál componente valida credenciales, registra métricas y aplica límites.

La configuración de un endpoint no crea por sí sola planes comerciales, cuotas por usuario ni facturación. La integración con Polar.sh u otro sistema de pagos requiere diseñar y probar la sincronización de suscripciones, derechos de acceso y límites del backend.

## 10. Ejecutar Egeo en desarrollo

Hay dos modos distintos: desarrollo del código fuente o prueba de la stack en contenedores. Elegir el que corresponda.

### Opción A. Desarrollo de frontend y backend desde el código

Este modo permite observar los cambios del código fuente durante el desarrollo.

1. Instalar la versión de Node.js indicada por la documentación de la revisión de LibreChat fijada.
2. Configurar MongoDB y las dependencias que requiera la configuración de Egeo. Para funciones de búsqueda, RAG u otras capacidades puede haber servicios auxiliares adicionales.
3. Copiar y completar `.env.example`:

   ```bash
   cp .env.example .env
   ```

4. Ajustar, como mínimo, `MONGO_URI`, `DOMAIN_CLIENT`, `DOMAIN_SERVER` y los secretos requeridos para el entorno.
5. Instalar/reconstruir dependencias según los scripts de la revisión:

   ```bash
   npm run reinstall
   ```

6. Iniciar el backend en una terminal:

   ```bash
   npm run backend:dev
   ```

7. Iniciar el frontend en otra terminal:

   ```bash
   npm run frontend:dev
   ```

8. Abrir `http://localhost:3080` si la configuración local conserva el puerto por defecto. Si la revisión usa otra configuración, seguir el mensaje de inicio del servidor.

Los scripts y dependencias pueden cambiar entre versiones. Consultar siempre los scripts vigentes en `package.json` y la [guía oficial de instalación local](https://www.librechat.ai/docs/local).

### Opción B. Probar la stack oficial de Docker Compose

Desde la raíz del repositorio:

```bash
cp .env.example .env
```

Completar `.env` y después iniciar los servicios:

```bash
docker compose up -d
docker compose ps
docker compose logs -f api
```

En una configuración que conserve el puerto por defecto, abrir `http://localhost:3080`.

**Atención:** según la revisión, el Compose base puede usar una imagen de desarrollo ya construida. En ese caso, editar archivos en el repositorio no cambia automáticamente el contenido de esa imagen. Para verificar tus modificaciones de Egeo, construir una imagen desde el código propio, como se explica en la sección siguiente.

Para detener los servicios sin borrar volúmenes:

```bash
docker compose down
```

No usar `docker compose down -v` salvo que se tenga claro que se desean borrar también los volúmenes de datos.

## 11. Construir y probar la imagen Docker de Egeo

El objetivo es probar una imagen que contenga los cambios versionados de Egeo, no una imagen pública que se actualiza de forma independiente.

### 11.1. Crear un override local de Compose

Revisar primero el `docker-compose.yml` y el ejemplo de override de la revisión actual. En las versiones que usan el servicio `api` y el `Dockerfile` en la raíz, crear `docker-compose.override.yml` con una configuración equivalente a esta:

```yaml
services:
  api:
    image: egeo:local
    build:
      context: .
      dockerfile: Dockerfile
```

Este ejemplo añade la construcción local al servicio API y etiqueta la imagen como `egeo:local`. Conservar en el Compose base los servicios, redes, volúmenes y variables de entorno que requiere LibreChat. Si el servicio tiene otro nombre o el proyecto cambia su estructura de Docker, adaptar el override a esa revisión.

### 11.2. Construir e iniciar

```bash
docker compose config
docker compose up -d --build
docker compose ps
docker compose logs -f api
```

`docker compose config` ayuda a detectar errores de sintaxis y a inspeccionar la configuración combinada. Revisar su salida antes de compartirla: puede expandir valores sensibles según el entorno.

### 11.3. Verificaciones mínimas

- La aplicación carga con la marca Egeo.
- El título y favicon son correctos.
- La API del modelo configurado responde.
- El acceso, las sesiones y el envío de mensajes funcionan.
- Los archivos y conversaciones persisten tras reiniciar el contenedor.
- No hay errores inesperados en los logs.

Una imagen válida para desarrollo no es automáticamente una versión apta para producción. Para producción, fijar un tag inmutable, validar seguridad, configurar HTTPS y ejecutar las pruebas correspondientes.

## 12. Actualizar Egeo desde LibreChat

Realizar la sincronización en una ventana de mantenimiento del desarrollo, no directamente en el servidor de producción.

### Paso 1. Asegurar el estado actual

```bash
git status
git switch main
git status
```

La rama `main` debe estar limpia y sin commits de personalización. Si hay cambios pendientes, guardarlos o resolverlos antes de actualizar.

### Paso 2. Descargar los cambios oficiales

```bash
git fetch upstream
git log --oneline --decorate -10 upstream/main
```

Revisar las notas de versión, los cambios de configuración y las instrucciones de actualización oficiales para identificar migraciones, variables nuevas y breaking changes.

### Paso 3. Avanzar `main` únicamente si es posible hacerlo en fast-forward

```bash
git merge --ff-only upstream/main
git push origin main
```

Si se produce un error, detenerse y diagnosticarlo. No forzar `main` ni sobrescribir el historial con un `reset --hard` sin entender qué diverge.

### Paso 4. Integrar el upstream en Egeo

```bash
git switch egeo-prod
git pull --ff-only origin egeo-prod
git merge --no-ff main
```

Resolver los conflictos, ejecutar pruebas y revisar el diff completo. La política recomendada es que el merge de `main` a `egeo-prod` sea un evento visible en el historial.

### Paso 5. Probar antes de publicar

```bash
git diff --check
npm run lint
npm run test:client
npm run test:api
```

Estos comandos dependen de las herramientas y los scripts de la versión elegida. Si una revisión cambia los scripts o requiere servicios auxiliares, seguir su documentación. No declarar la actualización aprobada si las pruebas fallan sin explicación o sin un plan de corrección.

### Paso 6. Publicar la rama y desplegar en staging

```bash
git push origin egeo-prod
```

Abrir un Pull Request o registrar la aprobación de la actualización, construir una nueva imagen identificable por tag o SHA, desplegar en staging y ejecutar pruebas funcionales. Promover exactamente esa versión a producción después de aprobarla.

## 13. Resolver conflictos de actualización

Los conflictos aparecen cuando LibreChat y Egeo modifican las mismas líneas o archivos.

Procedimiento recomendado:

1. Identificar qué cambió upstream y por qué.
2. Leer el archivo completo y su contexto; no elegir automáticamente la versión de Egeo ni la de upstream.
3. Conservar la corrección funcional upstream y volver a aplicar la identidad de Egeo de la forma menos invasiva posible.
4. Ejecutar las pruebas del área afectada.
5. Documentar decisiones no obvias en el PR.

Comandos útiles:

```bash
git status
git diff --name-only --diff-filter=U
git diff --check
```

Para un conflicto concreto, editar manualmente las marcas `<<<<<<<`, `=======` y `>>>>>>>`, y después:

```bash
git add <archivo-resuelto>
git commit
```

No resolver en bloque usando `--ours` o `--theirs` para toda la actualización: se puede descartar una corrección de seguridad o romper una personalización sin notarlo.

Si la actualización es demasiado grande, trabajar en una rama de integración separada desde `egeo-prod`, documentar los riesgos y fusionarla solo cuando termine la batería de pruebas.

## 14. Seguridad y datos

### Secretos

- Mantener `.env`, claves de proveedores, tokens, claves privadas y credenciales fuera de Git.
- Generar secretos únicos y persistentes para cada entorno, siguiendo los nombres y longitudes indicados en el `.env.example` de la versión instalada.
- No compartir claves entre desarrollo, staging y producción.
- Rotar cualquier secreto que haya sido incluido accidentalmente en un commit; eliminarlo del último archivo no basta para borrarlo del historial.

### Registro y autenticación

- No exponer directamente el puerto de la aplicación a Internet si el diseño requiere acceso mediante Cloudflare, HAProxy, Traefik o Kong.
- Configurar HTTPS y los encabezados de proxy de acuerdo con la topología real; no confiar ciegamente en encabezados enviados por clientes.
- Restringir la inscripción pública hasta tener definidos verificación de correo, protección contra abuso, recuperación de cuenta y políticas de acceso. Para pruebas puede habilitarse; en producción, revisar `ALLOW_REGISTRATION` y otras variables equivalentes.
- La integración con Authentik/OIDC, el cierre de registros y la gestión de sesiones deben validarse contra la versión concreta de Egeo; no dar por hecho que la mera presencia del proveedor de identidad configura automáticamente todos los controles.
- Limitar el acceso al panel administrativo y comprobar la protección de endpoints sensibles.

### Persistencia y copias de seguridad

LibreChat utiliza **MongoDB** para su estado de aplicación. Redis puede formar parte de la arquitectura para coordinación, caché o streams según la configuración; no reemplaza a MongoDB. No suponer que una base PostgreSQL existente puede sustituir la base nativa de LibreChat sin una adaptación específica.

Definir y probar una estrategia de backup que cubra, según las funciones habilitadas:

- MongoDB y su procedimiento de restauración;
- archivos de usuarios, subidas y datos auxiliares persistidos;
- configuración operativa cifrada y secretos almacenados con controles adecuados;
- imágenes/versiones desplegadas y configuración de infraestructura.

Cloudflare R2 puede servir como destino de copias de seguridad de archivos o dumps, pero **no constituye por sí solo** una estrategia de backup. Probar una restauración periódicamente y definir retención, cifrado y permisos mínimos.

### Observabilidad

Centralizar logs y métricas evitando registrar claves, tokens o contenido sensible innecesario. Langfuse puede utilizarse para trazas de IA cuando se configura correctamente; revisar las políticas de consentimiento, retención y datos personales antes de habilitar trazas en producción.

## 15. Pruebas antes de publicar

### Lista funcional mínima

- [ ] Login, logout y persistencia de sesión.
- [ ] Registro de usuario controlado según la política del entorno.
- [ ] Conversación nueva y continuación de una existente.
- [ ] Streaming de respuestas y manejo de errores del proveedor.
- [ ] Selección y disponibilidad de los modelos configurados.
- [ ] Título de conversación y recarga de una conversación existente.
- [ ] Marca Egeo en acceso, interfaz, favicon y título HTML.
- [ ] Subida/descarga de archivos si la función está habilitada.
- [ ] Comportamiento en escritorio y móvil; modo claro y oscuro.
- [ ] Errores en consola del navegador y logs del backend revisados.
- [ ] Reinicio de contenedores sin pérdida inesperada de datos.
- [ ] Backup y restauración probados en un ambiente no productivo.

### Lista técnica mínima

```bash
git diff --check
docker compose config
```

Ejecutar los scripts de lint, pruebas unitarias e integración que existan en el `package.json` de la versión fijada. Las pruebas end-to-end pueden necesitar servicios auxiliares y variables adicionales; seguir las instrucciones del proyecto.

## 16. Estrategia de publicación y despliegue

El pipeline recomendado es:

```text
Pull Request
    -> revisión de código
    -> pruebas automáticas
    -> merge en egeo-prod
    -> construcción de imagen
    -> escaneo y validaciones
    -> despliegue en staging
    -> pruebas de aceptación
    -> promoción de la misma imagen a producción
    -> monitoreo y plan de rollback
```

### Versionado

Etiquetar imágenes y releases con una versión de Egeo y registrar el SHA del commit upstream integrado. Ejemplos ilustrativos:

- Release: `egeo-v0.1.0`
- Imagen: `ghcr.io/TU_ORG/egeo:0.1.0`
- Metadato de build: SHA de `egeo-prod` y referencia de LibreChat integrada.

No usar `latest` como único identificador de producción. Guardar la imagen anterior y documentar el procedimiento de rollback.

### Reverse proxies e infraestructura

Antes de publicar, definir claramente el flujo de entrada —por ejemplo, Cloudflare, edge/HAProxy, Kong y/o Traefik— evitando dos proxies que apliquen políticas distintas sin una necesidad documentada. La configuración concreta depende de dónde termine TLS, qué componente autentique, dónde se apliquen los límites y qué red pueda alcanzar al contenedor de Egeo.

No publicar MongoDB, Redis ni paneles de administración hacia Internet. Limitar las redes y puertos a los componentes que los necesitan.

## 17. Convenciones de trabajo

- Un PR debe resolver un problema definido y tener alcance revisable.
- Toda personalización persistente debe estar versionada; ningún cambio manual de producción puede ser la única copia de una funcionalidad.
- Separar personalización visual, configuración de modelos, autenticación, facturación y observabilidad en cambios independientes cuando sea posible.
- Documentar variables nuevas en `.env.example` **sin valores secretos reales**.
- Evitar modificar dependencias o archivos generados si el proyecto ofrece una forma oficial de configuración.
- Con cada actualización, registrar la versión upstream, los conflictos resueltos, las pruebas y los cambios operativos.
- Mantener un entorno de staging equivalente a producción en sus componentes críticos, pero con credenciales y datos de prueba.

## 18. Problemas frecuentes

### `git merge --ff-only upstream/main` falla

La rama local `main` no puede avanzar en línea recta hasta upstream. Ejecutar `git status`, inspeccionar `git log --oneline --graph --decorate --all -30` y revisar los commits divergentes. No forzar el historial para ocultar la divergencia.

### Los logos cambiaron en el código, pero no en la web

Probablemente se está ejecutando una imagen precompilada o el cliente no se volvió a construir. Confirmar qué imagen usa el contenedor y reconstruir desde el código versionado:

```bash
docker compose ps
docker compose images
docker compose up -d --build
```

Después, comprobar si el navegador conserva archivos en caché.

### `APP_TITLE=Egeo` no cambia toda la marca

`APP_TITLE` controla los lugares que usan esa variable; no necesariamente todos los metadatos estáticos ni cada componente. Buscar referencias con `rg`, revisar el HTML de entrada y reconstruir el frontend.

### El endpoint personalizado no aparece

Comprobar que `librechat.yaml` está montado en el servicio API, que su YAML es válido, que se reinició la aplicación y que los nombres de variables existen en el entorno del contenedor. Revisar logs del backend y validar el identificador exacto del modelo.

### El endpoint aparece, pero la inferencia falla

Verificar desde el contenedor API la URL, DNS, certificado TLS, autenticación, ruta base, compatibilidad de Chat Completions, nombre del modelo y parámetros admitidos. Un endpoint que solo implementa otro formato de API puede requerir una configuración distinta o un adaptador.

### El servicio no conserva conversaciones al reiniciar

Revisar los logs de MongoDB, el `MONGO_URI`, el nombre del volumen y que ningún comando haya eliminado volúmenes (`docker compose down -v`). No operar sobre volúmenes de producción sin backup comprobado.

### No arranca el build después de una actualización

Revisar los requisitos de Node, los scripts, lockfile, variables nuevas y notas de actualización de esa revisión. Repetir la instalación de dependencias de acuerdo con el procedimiento oficial; no borrar lockfiles ni actualizar todas las dependencias indiscriminadamente.

## 19. Fuentes oficiales

Consultar estas fuentes junto con la versión exacta del repositorio utilizada:

- [Repositorio oficial de LibreChat](https://github.com/LibreChat-AI/LibreChat)
- [Guía oficial de instalación local](https://www.librechat.ai/docs/local)
- [Instalación con Docker](https://www.librechat.ai/docs/install/docker)
- [Configuración de variables de entorno](https://www.librechat.ai/docs/configuration/dotenv)
- [Configuración `librechat.yaml`](https://www.librechat.ai/docs/configuration/librechat_yaml)
- [Endpoints de IA](https://www.librechat.ai/docs/configuration/librechat_yaml/ai_endpoints)
- [Estructura de endpoint personalizado](https://www.librechat.ai/docs/configuration/librechat_yaml/object_structure/custom_endpoint)
- [Configuración de interfaz](https://www.librechat.ai/docs/configuration/librechat_yaml/object_structure/interface)
- [Licencia del repositorio](https://github.com/LibreChat-AI/LibreChat/blob/main/LICENSE)

### Aviso de licencia y atribución

Antes de distribuir Egeo, revisar la licencia vigente del código upstream y las licencias de todas las dependencias y componentes que se empaqueten. Conservar los avisos y archivos de licencia exigidos y no presentar Egeo como el producto oficial de LibreChat. Este apartado es una pauta operativa, no asesoramiento jurídico.

---

**Mantenimiento del documento:** actualizar esta guía cuando cambie el flujo de ramas, la versión upstream fijada, la forma de construir Docker, las variables de entorno o el proceso de publicación de Egeo.
