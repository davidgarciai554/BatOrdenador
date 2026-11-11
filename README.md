# BatOrdenador

Una consola de casa: recordatorios, entrenos, salud, proyectos y un chat con tu propio agente
(Hermes). Todo se queda en tu servidor.

Esto es lo que necesitas para instalarla. No incluye el código fuente.

## Qué necesitas

- Una máquina con **Docker** y **Docker Compose**, encendida siempre que quieras usarla.
- Para usarla fuera de casa, y para las notificaciones: una dirección con **HTTPS** (un proxy
  inverso o un túnel que ya tengas). BatOrdenador no trae el suyo.
- Para el chat: **Hermes** ya instalado, con su panel accesible. Sin Hermes todo lo demás funciona.

Se usa desde el navegador, y se puede instalar como aplicación en el móvil y en el ordenador
(«Añadir a pantalla de inicio» o «Instalar»). No hay, por ahora, aplicación nativa para Android ni
para Windows.

## Instalación

1. **Descarga** `docker-compose.yml` y `.env.example` de este repositorio en una carpeta nueva de tu
   servidor, y entra en ella.

2. **Prepara la carpeta de datos.** El contenedor no corre como administrador, así que la carpeta
   tiene que ser suya:

   ```bash
   mkdir data
   chmod 700 data
   sudo chown 1000:1000 data
   ```

   En ese orden: después del `chown` la carpeta ya no es tuya. Si te saltas esto, el contenedor arranca y no puede abrir su base de datos.

3. **Crea tu configuración:**

   ```bash
   cp .env.example .env
   chmod 600 .env
   ```

   Abre `.env` y rellena, como mínimo:
   - `ALLOWED_ORIGINS`: la dirección exacta con la que vas a abrir la app.
   - `BATORDENADOR_ADMIN_USER` y `BATORDENADOR_ADMIN_PASSWORD`: tu primera cuenta.

4. **Arranca:**

   ```bash
   docker compose up -d
   docker compose logs
   ```

   En el registro tiene que aparecer `Creado el administrador`. Si la contraseña es demasiado
   floja, ahí verás por qué se ha rechazado: corrígela en `.env` y repite
   `docker compose up -d --force-recreate`.

5. **Entra** en la dirección que pusiste en `ALLOWED_ORIGINS` con tu usuario y tu contraseña.

6. **Cambia la contraseña** desde la app y **borra** de `.env` las dos líneas
   `BATORDENADOR_ADMIN_*`. Ya no hacen nada, pero no hay motivo para dejar una contraseña escrita.

## Si algo no va

| Qué ves | Qué pasa | Qué hacer |
|---|---|---|
| La página carga pero al entrar da error 403 | `ALLOWED_ORIGINS` no coincide con la dirección del navegador | Copia la dirección tal cual, con `http://` o `https://` y sin barra final |
| Entras por `https://` y la sesión no se mantiene | Falta `FORCE_SECURE_COOKIES=1` | Añádelo a `.env` |
| Entras por `http://` y no puedes iniciar sesión | Sobra `FORCE_SECURE_COOKIES=1` | Coméntalo |
| En el registro: `unable to open database file` o `EACCES` | La carpeta `data` no es del contenedor | Repite el paso 2 |
| En el registro: `no tiene ningún usuario` | Faltan las variables de la primera cuenta | Rellénalas y recrea el contenedor |
| No llegan notificaciones | Hace falta HTTPS | Ábrela por `https://` y vuelve a activarlas en el aparato |
| El micrófono no aparece en el chat | No hay servidor de voz configurado | Es opcional: `WHISPER_HOST` en `.env` |

Para ver el registro: `docker compose logs --tail=100`.

## Más personas

Desde la app, en **Usuarios**, das de alta a quien quieras y eliges qué módulos ve cada persona.
Cada cuenta necesita un perfil de Hermes propio; se crean desde esa misma pantalla.

## Conectar Hermes

BatOrdenador no trae un agente: se conecta al [Hermes](https://github.com/nousresearch/hermes-agent)
que ya tengas. Hacen falta dos caminos, uno en cada sentido.

### 1. De BatOrdenador a Hermes

BatOrdenador entra en el **panel** de Hermes con una cuenta suya. En `.env`:

```
HERMES_DASHBOARD_URL=http://hermes:9119
HERMES_DASHBOARD_USER=tu-usuario-del-panel
HERMES_DASHBOARD_PASSWORD=tu-contraseña-del-panel
```

La dirección tiene que ser alcanzable **desde dentro del contenedor** de BatOrdenador. `localhost`
no sirve: dentro del contenedor, `localhost` es el propio contenedor.

### 2. De Hermes a BatOrdenador

Hermes usa las herramientas de BatOrdenador llamando a su dirección `/mcp`. En `.env`:

```
HERMES_MCP_URL=http://batordenador:4321/mcp
```

Esa dirección tiene que ser alcanzable **desde Hermes**. Si Hermes corre en otro contenedor de la
misma máquina, lo más sencillo es meter los dos en una red de Docker y usar el nombre del
contenedor, como en el ejemplo:

```bash
docker network create casa
docker network connect casa batordenador
docker network connect casa hermes        # el nombre de tu contenedor de Hermes
```

Para que la red sobreviva a recrear los contenedores, decláralo en los dos `docker-compose.yml`:

```yaml
services:
  batordenador:
    networks: [casa]
networks:
  casa:
    external: true
```

### 3. Enlazar tu cuenta

1. Recrea el contenedor: `docker compose up -d --force-recreate`.
2. En la app, entra en **Usuarios**, abre tu ficha y pulsa **Reintentar conexión**.

Con eso BatOrdenador escribe solo, en tu perfil de Hermes:

- la **conexión** con sus herramientas, con una clave propia de tu cuenta que renueva antes de que
  caduque;
- una **habilidad** llamada `batordenador`, que le explica a Hermes qué es BatOrdenador y qué
  módulos tienes.

No hay que editar ningún fichero de Hermes a mano. Para comprobarlo, pregúntale en el chat: «¿qué
puedes hacer aquí?».

Cada persona que des de alta necesita su **propio perfil** de Hermes; se crean desde su ficha, y
BatOrdenador repite estos pasos solo al asignárselo. Así el Hermes de cada una solo ve lo suyo.

### 4. La personalidad de Alfred (opcional)

En la ficha de cada persona hay un control **Personalidad de Alfred**. Al aplicarla, ese Hermes se
presenta como Alfred, el mayordomo de la casa. Se añade al final de su `SOUL.md` sin tocar lo que
ya tuviera, y se quita desde el mismo sitio.

### Si no conecta

| Qué ves | Qué pasa | Qué hacer |
|---|---|---|
| «No se pudo contactar con el dashboard de Hermes» | BatOrdenador no llega al panel | Revisa `HERMES_DASHBOARD_URL` y que los dos contenedores comparten red |
| «El dashboard de Hermes rechazó las credenciales» | Usuario o contraseña del panel | Corrígelos en `.env` y recrea el contenedor |
| «Falta HERMES_MCP_URL en el .env» | No has dicho por dónde llega Hermes | Rellénala y recrea el contenedor |
| Hermes dice que BatOrdenador no responde | Hermes no llega a `HERMES_MCP_URL` | Comprueba la red del paso 2 desde el contenedor de Hermes |
| Hermes no menciona un módulo | Esa persona no lo tiene concedido | Dáselo en **Usuarios**; la habilidad se actualiza sola |

La conexión entre BatOrdenador y el panel de Hermes va sin cifrar. Que no salga de tu máquina o de
tu red.

## Copias de seguridad

Todo está en la carpeta `data`. Cópiala **entera**, con el contenedor parado:

```bash
docker compose down
sudo cp -a data /ruta/a/tu/copia
docker compose up -d
```

No copies solo `batordenador.db`: los cambios recientes viven en los ficheros que tiene al lado, y
una copia de ese único fichero puede salir vacía sin dar ningún error.

En `data` están también `vapid.json` (las claves de las notificaciones; si se pierden, cada aparato
tiene que volver a activarlas) y las contraseñas de todas las cuentas, cifradas. Guarda la copia
con el mismo cuidado que el original.

Para restaurar: para el contenedor, sustituye `data` por la copia y repite el `chown` del paso 2.

## Actualizar

```bash
docker compose pull
docker compose up -d
```

La base de datos se actualiza sola al arrancar. Haz una copia antes.

## Lo que conviene saber

- **Servicios** y la sincronización de **Salud** con relojes y básculas están en pruebas: pueden
  cambiar o fallar.
- No hay soporte ni garantía.
- Está pensada para una casa: un servidor, unas pocas personas que se conocen.

## Licencia

Puedes descargar BatOrdenador y usarlo gratis en tu casa. **No se puede modificar, descompilar ni
redistribuir.** El texto completo está en [LICENSE](LICENSE), y las licencias de los componentes de
terceros que incluye, en [NOTICE.md](NOTICE.md).
