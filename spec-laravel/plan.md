# Plan técnico — Manager Library (Laravel + React)

Traducción de [`../spec-python/plan.md`](../spec-python/plan.md) al stack del reto. Las reglas de negocio, las historias de usuario,
los criterios de aceptación, el contrato de la API y el modelo de datos **no** cambian: siguen viviendo en `../spec-python/` (idéntica
a `../spec-.net/` salvo en tecnología). Aquí solo se traducen las decisiones de tecnología. Donde el spec original es ambiguo o
inconsistente, la interpretación aplicada queda en [`interpretations.md`](interpretations.md). El estado de las tareas vive en
[`tasks.md`](tasks.md).

Precedencia en caso de duda: `constitution.md` → `spec.md` → `data-model.md` / `api-contract.md` → este plan.
Si este plan y el spec original discrepan, gana el spec original y este plan está mal.

Idioma (artículo XI): código, columnas, rutas y nombres de archivo en inglés; documentación y mensajes al usuario en español.

## 1. Decisiones técnicas

| # | Decisión | Original (Python) | Traducción a Laravel / React | Por qué |
|---|---|---|---|---|
| D-01 | Lenguaje y runtime | Python 3.12 | PHP 8.3 (imagen `php:8.3-cli`). Versión de Laravel: la que instale `composer create-project laravel/laravel` sobre PHP 8.3; la versión exacta queda en `composer.lock`. Frontend: React con Vite sobre Node 22 | Stack del reto (`../README.md`) |
| D-02 | Framework HTTP | FastAPI | Enrutamiento de Laravel. Todas las rutas en `routes/api.php` bajo el prefijo `api/v1` (`apiPrefix: 'api/v1'` en `bootstrap/app.php`). **No** se usa `php artisan install:api` porque instala Sanctum | Ruta base del contrato `/api/v1` |
| D-03 | Persistencia | SQLAlchemy 2 + Alembic sobre PostgreSQL 16 | Migraciones de Laravel + query builder / Eloquent sobre PostgreSQL 16, esquema `library`. La migración inicial usa SQL explícito con `DB::unprepared(...)` (tablas, `CHECK`, índices parciales y semilla), porque Laravel no tiene API fluida para `CHECK` ni índices parciales y así el DDL coincide línea a línea con `data-model.md` | Artículo V, ADR-001 |
| D-04 | Concurrencia | Testigo de versión por libro + `CHECK` de rango (ADR-002) | `UPDATE book SET available_copies = available_copies - 1 WHERE id = ? AND available_copies > 0` condicional dentro de `DB::transaction`; 0 filas afectadas → 409 `BOOK_NOT_AVAILABLE`, sin reintento. `ck_book_available_range` sigue siendo la última barrera. No se añade columna de versión (ver interpretaciones I-01, I-02) | RNF-2 sin cambiar el modelo de datos |
| D-05 | Fechas | Instantes UTC (`timestamptz`); puerto `Clock` | `config('app.timezone') = 'UTC'`; columnas `timestampTz`; `Clock` es una interfaz en `app/Application/Ports`; el adaptador real devuelve `CarbonImmutable::now('UTC')` y las pruebas usan un reloj fijo | Probar vencimientos sin esperar 14 días |
| D-06 | Reporte | Una consulta agregada detrás de un puerto de lectura (ADR-004) | Puerto `ReportReader`; el adaptador ejecuta una sola consulta `GROUP BY book_id, book_title, category_name` sobre `loan_item` unida a `loan`. Nunca se une al catálogo vivo | Artículo VIII, ADR-004 |
| D-07 | Objetos de valor | `Isbn`, `Quantity`, `DocumentNumber`, `DateRange` dentro de la fila de su dueño | Clases PHP `final readonly` en `app/Domain`; se guardan como columnas simples del dueño, sin tabla | No tienen identidad |
| D-08 | Portadas | Almacenamiento externo (disco en local, S3 en producción); la base guarda `cover_key` | Puerto `CoverStorage`; el adaptador local usa un disco de filesystem de Laravel con raíz en `COVERS_DIR`. La base guarda solo la clave opaca; `null` si no hay portada, nunca `''` | CA-03.4 |
| D-09 | Contraseñas | Argon2id detrás de `PasswordHasher` | `password_hash($plain, PASSWORD_ARGON2ID)` / `password_verify` detrás del puerto `PasswordHasher`. **Verificado el 2026-10-13 a las 15:24:** `php:8.3-cli` (PHP 8.3.35) devuelve `true` para `defined("PASSWORD_ARGON2ID")` | Artículo IX |
| D-10 | Admin inicial | La aplicación crea el admin inicial desde `ADMIN_USERNAME` / `ADMIN_PASSWORD` | Comando Artisan idempotente `library:bootstrap-admin`, que ejecuta el entrypoint del contenedor de la API después de `php artisan migrate --force`. No es un seeder: el hash solo puede producirlo el puerto `PasswordHasher` | `data-model.md` §9 |
| D-11 | Autenticación | JWT HS256, 8 h, `JWT_SECRET` | `firebase/php-jwt`, HS256, `exp = now + 28800`, solo los claims `sub` y `role`. Un middleware propio valida el token `Bearer` y el rol. **No se usa Sanctum**: emite tokens opacos, no JWT | `api-contract.md` §4 |
| D-12 | Pruebas | pytest, fakes para los puertos, base real solo en integración | PHPUnit (viene con Laravel) con fakes en memoria para los puertos; PostgreSQL solo en pruebas de integración y de concurrencia. Pruebas de arquitectura (T-19): `arch()` de Pest o una prueba de PHPUnit que revise los `use`; se decide al tomar T-19 | Pirámide de pruebas |

## 2. Correspondencia de la arquitectura (hexagonal)

| Original (`src/library/…`) | Laravel (`app/…`) |
|---|---|
| `domain/` | `app/Domain/` — solo PHP puro: sin `Illuminate\*`, sin Eloquent |
| `application/ports/outbound/` | `app/Application/Ports/` — interfaces, sin tipos del framework en las firmas |
| `application/services/` | `app/Application/Services/` — un caso de uso por clase (`LendBooksService`, …) |
| `adapters/inbound/http/` | `app/Adapters/Http/` — controladores, form requests, traducción de errores |
| `adapters/outbound/{persistence,storage,security}/` | `app/Adapters/Persistence/`, `app/Adapters/Storage/`, `app/Adapters/Security/` |
| `bootstrap/composition/port_bindings.py` | `app/Bootstrap/LibraryServiceProvider.php` — el **único** lugar donde un puerto se une a su adaptador |

## 3. Valores por defecto de Laravel que romperían el spec (y qué se hace en su lugar)

| Valor por defecto de Laravel | Regla del spec que rompe | Qué se hace |
|---|---|---|
| Migraciones `users`, `password_reset_tokens`, `sessions`, `cache`, `jobs` | Solo seis tablas (`data-model.md` §2) | Se borran antes de ejecutar la primera migración |
| `$table->timestamps()` y `$timestamps = true` en Eloquent | Sin `created_at`/`updated_at` (`data-model.md` §3) | No se usan; los modelos declaran `public $timestamps = false` |
| `$table->uuid('id')->default(...)` / cualquier `->default()` / `->useCurrent()` | Ninguna columna lleva `DEFAULT` (`data-model.md` §3) | Los ids los genera la aplicación (UUID v4); las migraciones no llevan valores por defecto |
| La tabla `migrations` propia de Laravel | Seis tablas en el esquema `library` | Se mantiene fuera del esquema `library` (p. ej. `public.migrations`). **Por verificar** con las consultas de `data-model.md` §10 |
| Sanctum (`install:api`) | JWT con `sub` y `role` | No se instala; ver D-11 |
| Cuerpo 422 de validación `{message, errors}` | Cuerpo de error único `{code, message, details}` (RNF-4) | El manejo de excepciones en `bootstrap/app.php` convierte toda excepción al cuerpo del contrato |
| JSON del paginador (`data`, `links`, `meta`, …) | `{items, page, size, total}` | Las respuestas se arman a mano desde `LengthAwarePaginator` |
| Modelo `users` / guard de autenticación `User` | La tabla se llama `user`, en el esquema `library` | Tabla `user` propia y middleware JWT; no se usa el guard de Laravel |
| Orden de rutas `/loans/{id}` antes de `/loans/overdue` | `GET /loans/overdue` debe resolverse | `/loans/overdue` se declara primero y `{id}` se restringe con `whereUuid` |

## 4. Configuración por entorno

| Variable original | Variable en Laravel | Obligatoria | Ejemplo |
|---|---|---|---|
| `DATABASE_URL` | `DB_URL` (o `DB_CONNECTION=pgsql` + `DB_HOST`/`DB_PORT`/`DB_DATABASE`/`DB_USERNAME`/`DB_PASSWORD`) | sí | `postgresql://library:<DB_PASSWORD>@db:5432/library` |
| `JWT_SECRET` | `JWT_SECRET` | sí | cadena larga aleatoria |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | igual | sí | definidas en `.env`, nunca en el repositorio |
| `COVERS_DIR` | igual | no | `storage/app/covers` |
| `CORS_ORIGIN` | igual, leída por `config/cors.php` | sí | `http://localhost:5173` |
| — | `APP_KEY` | sí (Laravel) | se genera con `php artisan key:generate`; nunca se versiona |

Solo se versiona `.env.example` (artículo IX). Los valores reales viven en `.env`, ignorado por git; ver `infra/.env.example`.

## 5. Estrategia de pruebas

| Nivel | Qué prueba | Contra qué |
|---|---|---|
| Dominio | Máximo 3, vencimiento a 14 días, ejemplares en rango, ISBN de 13 dígitos | Objetos PHP puros |
| Aplicación | Prestar, devolver, reporte | Fakes de los puertos |
| Integración | Repositorios y consulta del reporte | PostgreSQL 16 real (contenedor) |
| Contrato | Cuerpo de error único y códigos de estado | La API levantada |
| Arquitectura | Artículos I, II y III | Ver D-12 |
| Concurrencia | Dos préstamos del último ejemplar | PostgreSQL 16 real |

## 6. Orden de construcción

El mismo del original: esqueleto y salud → migración inicial con semilla → dominio → casos de uso con fakes → adaptadores → API →
pruebas de contrato y de concurrencia. Prioridad para esta entrega, acotada por el tiempo: el **núcleo** (T-01 a T-17); ver
[`tasks.md`](tasks.md).
