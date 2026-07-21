# SilgoDev Outreach

CRM de outreach para agencias de marketing y publicidad. Busca leads en **OpenStreetMap** (Overpass), extrae emails de sus webs, gestiona el pipeline en SQLite y envía correos de presentación como brazo técnico white-label.

Stack: **Laravel 12**, **PHP 8.3**, **SQLite**, **SMTP nativo**, **Docker**.

---

## Arranque

```bash
docker compose up --build
```

Abre el panel en **[http://localhost:8000](http://localhost:8000)**.

En el primer arranque, `app` ejecuta `composer install`, crea `database/database.sqlite`, genera `APP_KEY` si falta y aplica migraciones.

Antes del primer envío real, edita `.env` y pon la contraseña SMTP en `MAIL_PASSWORD`.

---

## Operar el cosechador en un servidor (solo recoger emails)

Objetivo: dejar el cosechador recorriendo toda España de forma autónoma, **sin enviar correos todavía**. El envío está **en pausa por defecto** (`outreach.sending.enabled = false`), independiente de la cosecha.

### 1. Arrancar

```bash
# Primer despliegue (o tras cambios de código/compose)
docker compose up -d --build

# Sembrar las áreas (provincias) una sola vez
docker compose exec app php artisan migrate --force
docker compose exec app php artisan db:seed --class=HarvestAreaSeeder --force
```

Servicios que quedan corriendo: `app` (panel), `scheduler` (lanza `harvest:run` cada `interval` min), `queue` (Overpass/altas), `queue-scraping` + `queue-scraping-2` (scrape de webs). El `scheduler` orquesta todo; no hay que lanzar búsquedas a mano.

### 2. Ver que está vivo

```bash
# Estado por SSH (sale con código ≠0 si el latido está muerto → healthcheck)
docker compose exec app php artisan harvest:status

# Latido crudo
docker compose exec app php artisan tinker --execute='echo App\Services\HarvestHeartbeat::ageSeconds()." s desde el último latido\n";'
```

También en el panel web (`/`, sección **Cosecha**): badge verde si el último latido es &lt; 2 min, barra de avance España, área en proceso y últimas 5 áreas. Se auto-refresca cada 10 s (JSON en `/harvest/status`).

### 3. Pausar / reanudar (sin tocar `.env`)

```bash
docker compose exec app php artisan harvest:pause     # detiene el recorrido
docker compose exec app php artisan harvest:resume    # lo reanuda
```

El flag vive en cache; `harvest:run` lo respeta en cada ejecución.

### 4. Consultar progreso

```bash
docker compose exec app php artisan harvest:areas-status   # tabla de las 52 áreas
docker compose exec app php artisan harvest:status         # resumen + avance + latido
```

Contadores del CRM:

```bash
docker compose exec app php artisan tinker --execute='echo App\Models\Lead::count()." leads / ".App\Models\Lead::whereNotNull("email")->count()." con email\n";'
```

### 5. Resistencia a reinicios

- La cola es **database** (`QUEUE_CONNECTION=database`): si el contenedor se reinicia a mitad del scrape, los jobs persisten y los workers los reanudan solos; el lote termina y cierra el área.
- Dedup por `place_id` + email/dominio + suppressions: reprocesar un área **no duplica** leads.
- Si un área quedó `en_proceso` por una caída en fase Overpass (sin lote), `harvest:run` la recupera: si el lote terminó la cierra, y si lleva más de `stale_area_seconds` (30 min por defecto) sin lote la reintenta. El recorrido nunca se queda bloqueado.

### 6. Activar el envío (más adelante, NO ahora)

Cuando quieras empezar a enviar, en `.env` del servidor pon `OUTREACH_SENDING_ENABLED=true` y reinicia `scheduler`. El envío respeta warm-up, cupo diario, días laborables, suppressions y `List-Unsubscribe`.

### Qué queda por configurar a mano

| Tema | Qué hacer |
|------|-----------|
| **Áreas / prioridades** | Ajusta `HarvestArea` (columna `priority`, menor = antes). Reintentar una: `php artisan harvest:reset-area "Nombre"`. Editar el set inicial: `database/seeders/HarvestAreaSeeder.php`. |
| **Ritmo / recursos** | `config/outreach.php` → `harvest`: `interval`, `overpass_delay`, `requests_per_minute`, `scraping_concurrency` (nº de workers `queue-scraping*`), `worker_*`. |
| **DNS de envío (para después)** | SPF, DKIM y DMARC del dominio remitente en Hostinger; `MAIL_PASSWORD` y `OUTREACH_LEGAL_*` / `OUTREACH_UNSUBSCRIBE_*` en `.env`. Nada de esto hace falta para solo recoger emails. |
| **Directorio de asociación** | Opcional: `sources.association_directory.enabled=true` + URL/robots permitidos. |

---

## Fuentes de leads

Todas las fuentes activas pasan por `agencies:search` → `LeadCaptureService::createBase` (dedup por email/dominio/suppression + verificación MX si hay email en la fuente). El scrape de webs se encola en `ScrapeWebsiteJob` (cola `scraping`), no se hace en línea.

| Clave en `config/outreach.php` → `sources` | Qué captura | Activar |
|--------------------------------------------|-------------|---------|
| **overpass** | Agencias OSM (`filters`) → segmento `agencia` | `enabled => true` (por defecto) |
| **overpass** + flag | Negocios locales (`filters_negocios`) → segmento `negocio` (solo con web) | `php artisan agencies:search --negocios` |
| **association_directory** | Listado HTML de asociación (plantilla DomCrawler) | `enabled => true` + `base_url` / selectores legales |

Para clonar otro directorio: copia el bloque `association_directory`, ajusta URL/selectores y regístralo en `LeadSourceManager`.

**robots.txt:** `AssociationDirectorySource` y `EmailScraper` usan `RobotsChecker` (caché por host). Overpass no scrapea webs; el scrape de emails sí respeta robots.

---

## Flujo completo

```
1. Buscar     agencies:search [--negocios] [--area=Nombre]
              └─ Overpass / HarvestArea → leads base (sin email scrapeado)
              └─ ScrapeWebsiteJob (cola scraping) → EmailScraper + EmailVerifier

2. Verificar  (en captura de email fuente o al completar scrape: valido | invalido | riesgo)

3. Enviar     agencies:send | panel | schedule 09:30 lun–jue
              └─ warm-up / cupo / días / suppressions / List-Unsubscribe

4. Bandeja    outreach:process-inbox (cada 15 min)
              └─ rebote → suppression+rebotado | BAJA → baja | respuesta → respondido
```

Comprobar plantilla sin leads reales: `agencies:test-send tu@email.com` (+ Mailpit o mail-tester).

---

## Colas y workers (Overpass vs scrape)

La búsqueda ya no scrapea webs en el mismo proceso. Flujo:

1. `RunSearchJob` / `harvest:run` consulta Overpass, crea leads base y **despacha** un batch de `ScrapeWebsiteJob`.
2. Workers dedicados procesan la cola **`scraping`** con poca concurrencia (**2** contenedores por defecto).

```bash
# Solo scrape (1 proceso; Docker ya levanta 2)
php artisan queue:work --queue=scraping --sleep=2 --tries=2 --timeout=90 \
  --max-jobs=50 --max-time=1800 --memory=160

# Jobs generales (búsqueda Overpass, envío)
php artisan queue:work --queue=default --sleep=3 --tries=3 --timeout=3600 \
  --max-jobs=30 --max-time=3600 --memory=320
```

`--max-jobs` / `--max-time` / `--memory` reinician el worker periódicamente (anti-fugas PHP). Alineados con `outreach.harvest.worker_max_jobs`, `worker_max_time`, `worker_memory`.

### Docker: límites y concurrencia

| Servicio | CPU | RAM | Notas |
|----------|-----|-----|--------|
| **queue** | 0.5 | 384M | cola `default` |
| **queue-scraping** + **-2** | 0.35 c/u | 192M c/u | 2 workers; no pases de 3 |
| **app** | 1.0 | 512M | panel + artisan |

Para un **3.er** worker de scrape: copia el bloque `queue-scraping-2` como `queue-scraping-3` y sube `scraping_concurrency` en `config/outreach.php` a `3`. Más workers = más riesgo de ban en Overpass/webs.

Throttle scrape: `requests_per_minute` (global) y `requests_per_domain_per_minute`. Overpass: `overpass_delay` (ms) + backoff exponencial en 429/504. UA identificable + `robots.txt`.

Poda semanal: `harvest:prune-logs` (domingos 03:15).

```bash
php artisan harvest:prune-logs
```

Cada `ScrapeWebsiteJob` tiene `tries=2`, backoff 15/60s, timeout 90s; el HTML se libera tras extraer emails.
### Orquestador de áreas (`harvest:run`)

Recorrido autónomo por `HarvestArea` (provincias sembradas):

1. `harvest:run` toma `nextPending()`, Overpass de esa área, batch de `ScrapeWebsiteJob`.
2. Al terminar el lote → `emails_found` + estado `hecho`. Si falla Overpass → `error` y la próxima ejecución coge otra área.
3. `Cache::lock` evita solapamientos; no arranca otra área mientras haya una `en_proceso`.
4. Schedule cada `outreach.harvest.interval` minutos (`withoutOverlapping` + flag `enabled`).

```bash
php artisan harvest:run
php artisan harvest:pause    # pausa sin tocar .env (Cache)
php artisan harvest:resume
php artisan harvest:status   # vivacidad + avance (exit 2 si latido stale y enabled)
php artisan harvest:areas-status
```

Healthcheck (p. ej. cada 5 min): `php artisan harvest:status` — código `2` si no hay latido en `heartbeat_stale_seconds` (por defecto 10 min) y la cosecha está activa.

Panel: sección **Cosecha** en `/` + JSON en `/harvest/status` (auto-refresh cada 10 s). Latido verde si &lt; 2 min.

Config: `config/outreach.php` → `harvest` (`enabled`, `interval`, `pause_between_areas_seconds`, `heartbeat_*`, `scraping_concurrency`, `requests_per_minute`, `overpass_delay`, `worker_max_jobs`, `worker_memory`).

---

## Servicios Docker

Los tres servicios construyen la misma imagen (`Dockerfile` con PHP 8.3-cli) y montan la carpeta del proyecto en `/app`, de modo que los cambios hechos en Cursor se reflejan al instante.

| Servicio | Qué hace |
|----------|----------|
| **app** | Servidor web en el puerto 8000. Instala dependencias, prepara SQLite y ejecuta `php artisan serve`. |
| **scheduler** | Ejecuta `php artisan schedule:work`. Sustituye al cron del sistema: cada minuto comprueba si hay tareas programadas (p. ej. el envío automático de las 09:30). Sin este servicio, el planificador nunca corre. |
| **queue** | `queue:work --queue=default` con reinicio (`--max-jobs` / `--memory`). |
| **queue-scraping** + **queue-scraping-2** | Dos workers de cola `scraping` (límites CPU/RAM). |
| **mailpit** | SMTP local (puerto **1025**) + UI web en **[http://localhost:8025](http://localhost:8025)**. Captura correos en desarrollo sin usar Hostinger. |
| **db-ui** | **Adminer** (sustituto de phpMyAdmin para SQLite) en **[http://localhost:8089](http://localhost:8089)**. |

Para usar Mailpit, en `.env` apunta el mailer a `MAIL_HOST=mailpit` y `MAIL_PORT=1025` (ver bloque comentado en `.env.example`). En producción deja la config de Hostinger.

### Ver tablas SQLite (Adminer)

phpMyAdmin **no** funciona con SQLite. Adminer (`db-ui`) sí:

1. Abre [http://localhost:8089](http://localhost:8089) (en el servidor: `http://IP:8089`).
2. Sistema: **SQLite 3**.
3. Base de datos: `/db/database.sqlite`.
4. Usuario y contraseña: déjalos vacíos → Entrar.

Verás `leads`, `harvest_areas`, `jobs`, `suppressions`, etc. Si el puerto 8089 no está abierto en el firewall del VPS, añádelo (`ufw allow 8089/tcp`).

---

## Uso desde el panel

En `http://localhost:8000` tienes:

- **Estadísticas** — total, con email, nuevos, contactados, contactados hoy, respondidos, clientes.
- **Buscar agencias** — encola Overpass por área (`RunSearchJob`); el scrape de emails va a la cola `scraping`.
- **Enviar correos** — encola el envío (con límite opcional).
- **Alta manual** — crea un lead a mano.
- **Filtros y tabla** — filtra por estado, cambia el estado con el desplegable, paginación de 50.

Las acciones de buscar y enviar se ejecutan en segundo plano vía cola (`queue` + `queue-scraping`).

---

## Uso desde CLI

Dentro del contenedor:

```bash
docker compose exec app php artisan agencies:search
docker compose exec app php artisan agencies:search --dry-run
docker compose exec app php artisan agencies:search --negocios

docker compose exec app php artisan agencies:send
docker compose exec app php artisan agencies:send --dry-run
docker compose exec app php artisan agencies:send --limit=5

docker compose exec app php artisan agencies:test-send tu@email.com
docker compose exec app php artisan outreach:process-inbox
```

| Comando | Descripción |
|---------|-------------|
| `agencies:search` | Overpass → leads base + encola `ScrapeWebsiteJob` (salta duplicados por `place_id`). Si hay `HarvestArea`, toma la siguiente pendiente. |
| `agencies:search --dry-run` | Simula la búsqueda sin guardar nada. |
| `agencies:search --negocios` | Incluye también `filters_negocios` (segmento negocio). |
| `agencies:search --area=Madrid` | Cosecha solo esa `HarvestArea` por nombre. |
| `agencies:search --sync-scrape` | Scrapea en línea (no recomendado; útil en depuración). |
| `agencies:send` | Envía correos a leads con estado `nuevo` y email, respetando warm-up y cupo diario. |
| `agencies:send --dry-run` | Lista los envíos que haría sin enviar nada (ignora restricción de días). |
| `agencies:send --limit=5` | Fuerza un límite de 5 correos en esta ejecución. |
| `agencies:test-send {email}` | Envía **un** correo de prueba (plantilla + `List-Unsubscribe`) a la dirección indicada. Ideal contra Mailpit o tu propio buzón. |
| `outreach:process-inbox` | Lee IMAP (rebotes / BAJA / respuestas). |
| `outreach:suppress {email}` | Añade email (y dominio) a suppressions (`--reason=baja\|rebote\|manual`). |

El envío automático también está programado en `routes/console.php`: **lunes a jueves a las 09:30** (Europe/Madrid), con `withoutOverlapping()`. El inbox se procesa cada 15 minutos.

---

## Correo (SMTP)

### Local vs producción

| Entorno | Destino | Cómo |
|---------|---------|------|
| **Local** | Mailpit | `MAIL_HOST=mailpit`, `MAIL_PORT=1025`, `MAIL_SCHEME=` vacío. UI: [http://localhost:8025](http://localhost:8025) |
| **Producción** | Hostinger | `MAIL_HOST=smtp.hostinger.com`, `MAIL_PORT=465`, `MAIL_SCHEME=smtps` + `MAIL_PASSWORD` |

El bloque de Mailpit está comentado en `.env.example`; no mezcles ambos perfiles a la vez.

### Contraseña (solo Hostinger / real)

En `.env`:

```env
MAIL_PASSWORD=tu-contraseña-real
```

La configuración SMTP de producción apunta a Hostinger (`smtp.hostinger.com:465`, SSL). El remitente es `contacto@onez.es` — **no subas `.env` al repositorio**.

### Cómo probar el envío

1. **Levanta Mailpit** (con el resto del stack):

   ```bash
   docker compose up --build
   ```

   Abre la bandeja de prueba: **[http://localhost:8025](http://localhost:8025)**.

2. **Apunta `.env` a Mailpit** (solo en local):

   ```env
   MAIL_MAILER=smtp
   MAIL_HOST=mailpit
   MAIL_PORT=1025
   MAIL_USERNAME=
   MAIL_PASSWORD=
   MAIL_SCHEME=
   ```

   Reinicia `app`/`queue` si ya estaban en marcha para que recarguen env.

3. **Envía un correo de prueba a ti mismo** (no a un lead):

   ```bash
   docker compose exec app php artisan agencies:test-send tu@email.com
   ```

   El comando valida el render de la plantilla y la cabecera `List-Unsubscribe`, luego envía un `AgencyOutreachMail`.

4. **Revisa en Mailpit** asunto, cuerpo (baja visible), remite y cabeceras.

5. **Deliverability real (SPF/DKIM/DMARC)** — con Hostinger en `.env`, envía a una dirección de [mail-tester.com](https://www.mail-tester.com/):

   ```bash
   docker compose exec app php artisan agencies:test-send test-XXXX@mail-tester.com
   ```

   Abre el informe en mail-tester y corrige DNS hasta una puntuación alta. **No uses leads reales** para estas pruebas.

### Deliverability: SPF, DKIM y DMARC (paso crítico)

Para que los correos lleguen a bandeja de entrada y no a spam, **configura en el DNS del dominio `onez.es`**:

1. **SPF** — autoriza a Hostinger a enviar en nombre de `@onez.es`.
2. **DKIM** — firma criptográfica de cada mensaje (Hostinger suele proporcionar el registro en su panel).
3. **DMARC** — política de qué hacer si SPF/DKIM fallan (empieza con `p=none` para monitorizar, luego `p=quarantine` o `p=reject`).

Sin estos tres registros, aunque la app funcione correctamente, la deliverability será mala. Es el paso más importante antes de enviar en volumen.

Los correos ya incluyen cumplimiento legal: identificación del remitente en el cuerpo, opción de baja visible y cabeceras `List-Unsubscribe` + `List-Unsubscribe-Post`.

---

## Salvaguardas incluidas

| Salvaguarda | Detalle |
|-------------|---------|
| **Warm-up progresivo** | El límite diario sube según el total histórico de contactados (5 → 8 → 12 → 18 → 25), con techo `OUTREACH_MAX_DAILY`. |
| **Cupo diario** | No envía más de lo permitido por warm-up; descuenta los ya enviados hoy. |
| **Solo días laborables** | Envío real solo lun–jue (`send_days: [1,2,3,4]`). Viernes, sábado y domingo no envía (salvo `--dry-run`). |
| **Delay aleatorio** | Entre 25 y 75 segundos entre cada correo del mismo lote. |
| **Sin duplicados OSM** | `agencies:search` salta leads con el mismo `place_id`. |
| **Try/catch por lead** | Si un envío falla, se registra el error y continúa con el siguiente. |
| **Dry-run** | Simula búsqueda o envío sin persistir ni mandar nada. |
| **Lista negra de dominios** | El scraper descarta emails de dominios como `sentry.io`, `wixpress.com`, etc. |
| **Cumplimiento legal** | Remitente identificado, baja en el pie, cabeceras `List-Unsubscribe`. |

---

## Flujo de estados

```
nuevo ──(envío OK)──► contactado ──► respondido ──► cliente
  │
  └──(sin email)──► sin_email

contactado / respondido / cliente ──► descartado | baja
```

| Estado | Significado |
|--------|-------------|
| `nuevo` | Lead con email, pendiente de primer contacto. |
| `sin_email` | Sin email (OSM ni scraper). No se envía. |
| `contactado` | Correo de outreach enviado. |
| `respondido` | La agencia ha respondido. |
| `cliente` | Ha contratado o hay encaje comercial. |
| `descartado` | No interesa seguir. |
| `rebotado` | Rebote SMTP detectado vía IMAP. |
| `baja` | Ha pedido no recibir más mensajes. |

Solo los leads en estado `nuevo` con email entran en `agencies:send`.

---

## Ajustes rápidos

### Área y filtros Overpass

En `.env`:

```env
OUTREACH_OVERPASS_AREA=Madrid
```

En `config/outreach.php` → `overpass.filters`, añade o quita pares `[tag, valor]`:

```php
['office', 'advertising_agency'],
['office', 'marketing'],
// ...
```

### Plantilla de correo

- Mailable: `app/Mail/AgencyOutreachMail.php`
- Vista markdown: `resources/views/emails/agency-outreach.blade.php`

### Warm-up, delays y días de envío

En `config/outreach.php` → `sending`:

```php
'warmup' => [0 => 5, 10 => 8, 30 => 12, 60 => 18, 120 => 25],
'delay_min' => 25,
'delay_max' => 75,
'send_days' => [1, 2, 3, 4],  // 1=lunes … 7=domingo (ISO)
'max_daily' => (int) env('OUTREACH_MAX_DAILY', 25),
```

### Datos legales del remitente

En `.env`:

```env
OUTREACH_LEGAL_NAME="Camilo Silva Gómez"
OUTREACH_LEGAL_ADDRESS="Madrid, España"
OUTREACH_UNSUBSCRIBE_EMAIL=contacto@onez.es
OUTREACH_UNSUBSCRIBE_URL=
```

---

## Estructura relevante

```
app/
  Console/Commands/     agencies:search, agencies:send, agencies:test-send, outreach:*
  Http/Controllers/     DashboardController
  Jobs/                 RunSearchJob, RunSendJob
  Mail/                 AgencyOutreachMail
  Models/               Lead, Suppression
  Services/             OverpassClient, EmailScraper, InboxProcessor, Sources/*
config/outreach.php     Fuentes, Overpass, scraper, envío
database/database.sqlite
resources/views/        layout, dashboard, emails/
routes/
  web.php               Panel
  console.php           Schedule
docker-compose.yml      app, scheduler, queue, queue-scraping(+-2), mailpit, db-ui (Adminer)
Dockerfile
```
