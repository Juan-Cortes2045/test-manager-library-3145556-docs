# Tareas — Manager Library (Laravel + React)

Traducción de [`../spec-python/tasks.md`](../spec-python/tasks.md) al stack del reto. Los identificadores, las dependencias y la
evidencia esperada son los originales; solo cambia la redacción técnica. Las interpretaciones citadas como `I-xx` están en
[`interpretations.md`](interpretations.md).

**Regla (artículo XII):** una tarea está **hecha** solo cuando su evidencia existe y se puede ejecutar. Cada tarea hecha enlaza a esa
evidencia (commit, prueba o comando). Nada se marca como hecho porque «ya está programado».

Estados: **pendiente** (no iniciada) · **en curso** (iniciada, la evidencia aún no está completa) · **hecha** (evidencia enlazada).

Última actualización: 2026-10-13, 16:35 (UTC-5). Hechas: T-01, T-02, T-03 y T-04. En curso: T-20 (base y API levantan con
`docker compose up`; falta el frontend). `app`, `page` y `tool` contienen solo su `README.md` inicial.

## Núcleo (T-01 a T-17)

| # | Tarea | Traducción a Laravel | Depende de | Evidencia exigida | Estado | Enlace a la evidencia |
|---|---|---|---|---|---|---|
| T-01 | Esqueleto del proyecto y `GET /health` | Aplicación Laravel en `api`, ruta `GET /api/v1/health` | — | `GET /api/v1/health` responde `{"status":"ok"}`; el contenedor arranca con un comando | hecha | `api` commits `de2cc6b` (ruta y error único) y `78583ba` (Dockerfile). 2026-10-13 16:17: desde `infra`, `docker compose up --build` → `curl localhost:8000/api/v1/health` devuelve `{"status":"ok"}`; `tests/Feature/HealthTest.php` y `ErrorBodyTest.php` pasan (3 pruebas) |
| T-02 | Migración inicial: seis tablas, FK, índices únicos y las cinco categorías sembradas | Una migración de Laravel con SQL explícito (`DB::unprepared`) en el esquema `library`; migraciones por defecto de Laravel eliminadas; tabla `migrations` en `public`; sin `DEFAULT` ni marcas de tiempo | T-01 | Las consultas de `data-model.md` §10 devuelven lo documentado; ninguna columna con `DEFAULT` | hecha | `api` commit `fa3c0d2`. `database/checks/data-model-section-10.sql` (copia literal de §10): 6 tablas en `library`, 30 columnas como en §3, 0 con `DEFAULT`; 6 PK, 5 FK con su `ON DELETE` (§5), `ck_book_available_range`, 5 índices únicos y 4 de acceso (§4); 5 categorías con los UUID de §9. Invariantes probadas con `INSERT`/`DELETE` reales (rango de ejemplares, ISBN único entre activos y reutilizable tras baja, línea repetida, documento y usuario únicos, FK `RESTRICT`). `ck_book_total_positive` y `ck_user_role` no existen, como marca §4 (T-18) |
| T-03 | Dominio: `Isbn`, `Quantity`, `DateRange` | Clases `final readonly` en `app/Domain` | T-01 | Pruebas unitarias: ISBN de 13 dígitos, cantidad > 0, fin ≥ inicio | hecha | `api/app/Domain/{Isbn,Quantity,DateRange}.php` (PHP puro, 0 imports de framework, artículo I). `php vendor/bin/phpunit tests/Unit/Domain`: 14 pruebas pasan (ISBN de 12, 14, con letras, guiones, espacios o vacío → `InvalidIsbn`; 0 y negativos → `InvalidQuantity`; fin < inicio → `InvalidDateRange`; fin = inicio aceptado). 2026-10-13 16:23 |
| T-04 | Dominio: `Book` (`withdraw`, `restock`, baja lógica) | `app/Domain/Book.php` | T-03 | Pruebas: nunca < 0, nunca > total | hecha | `api/app/Domain/Book.php` (PHP puro). `php vendor/bin/phpunit tests/Unit/Domain/BookTest.php`: 8 pruebas pasan (disponibles = total al crear; `withdraw` con 0 → `BookNotAvailable`; `restock` con disponibles = total → `InvalidAvailableCopies`; reconstruir con disponibles < 0 o > total, título vacío o `cover_key` vacío se rechaza; baja lógica conserva el primer instante). Suite completa: 25 pruebas. 2026-10-13 16:28. La regla «sin préstamos abiertos para dar de baja» es del caso de uso (T-08) |
| T-05 | Dominio: `Loan` (`addItem`, `returnItem`, estado calculado, 14 días, máx. 3) | `app/Domain/Loan.php`, `LoanItem.php`; puerto `Clock` | T-04 | Pruebas con un `Clock` falso: `open`, `overdue`, `closed` | pendiente | — |
| T-06 | Dominio: `Member` y `User` (normalización del nombre, rol válido) | `app/Domain/Member.php`, `User.php` | T-03 | `"Ana "` queda `ana`; el rol `admin` por API se rechaza | pendiente | — |
| T-07 | Puertos y casos de uso de autenticación (`LoginService`, `CreateLibrarianService`) con fakes | `app/Application/Services`, puertos en `app/Application/Ports` | T-06 | HU-01 y HU-02 pasan contra fakes | pendiente | — |
| T-08 | Puertos y casos de uso de libros (crear, buscar, editar, baja, portada) con fakes | Misma estructura | T-04 | HU-03 y HU-04 pasan contra fakes | pendiente | — |
| T-09 | Caso de uso prestar (`LendBooksService`) con fakes | Misma estructura | T-05, T-08 | HU-06 completa: límites, vencidos, sin ejemplares (I-05) | pendiente | — |
| T-10 | Casos de uso devolver y vencidos | Misma estructura | T-09 | HU-07 y HU-09; devolver dos veces → `ITEM_ALREADY_RETURNED` (I-06 para `days_late`) | pendiente | — |
| T-11 | Caso de uso del reporte de más prestados y su puerto de lectura | Puerto `ReportReader` | T-09 | HU-10; se conserva el título viejo tras renombrar (ADR-004) | pendiente | — |
| T-12 | Adaptador de persistencia: repositorios de libro, socio y usuario | `app/Adapters/Persistence` | T-02, T-08 | Pruebas de integración contra PostgreSQL 16 real | pendiente | — |
| T-13 | Adaptador de persistencia: repositorio de préstamos + control de concurrencia | `UPDATE` condicional sobre `available_copies` (I-01, I-02) | T-12, T-09 | Dos préstamos del último ejemplar: uno 201, el otro 409 (RNF-2) | pendiente | — |
| T-14 | Adaptador de seguridad: hash de claves y JWT | Argon2id con `password_hash`; `firebase/php-jwt` (D-09, D-11) | T-07 | La clave nunca aparece en logs ni respuestas (RNF-3) | pendiente | — |
| T-15 | Adaptador de portadas (disco) | Disco de filesystem de Laravel detrás de `CoverStorage` | T-08 | PNG de 2 MB ok; 3 MB → 422; sin portada → `cover_key` nulo | pendiente | — |
| T-16 | Adaptador HTTP: rutas de `api-contract.md`, cuerpo de error único, autorización por rol | `routes/api.php` bajo `/api/v1`; manejo de excepciones en `bootstrap/app.php`; middleware de JWT y rol (I-03, I-04) | T-10, T-11, T-14, T-15 | La prueba de contrato recorre todas las rutas y todos los códigos de `api-contract.md` §2 | pendiente | — |
| T-17 | Composición y arranque (admin inicial desde el entorno) | `LibraryServiceProvider`; comando `library:bootstrap-admin` (D-10) | T-16 | El sistema arranca con la base vacía y se puede iniciar sesión como admin | pendiente | — |

## Más allá del núcleo (T-18 a T-23)

| # | Tarea | Traducción a Laravel / React | Depende de | Evidencia exigida | Estado | Enlace a la evidencia |
|---|---|---|---|---|---|---|
| T-18 | Bajar a la base las invariantes solo dominio: `ck_book_total_positive`, `ck_user_role` | Nueva migración de Laravel con `DB::statement` | T-12 | Un `INSERT` por `psql` que las viole falla; el dueño del spec actualiza `data-model.md` §4 | pendiente | — |
| T-19 | Pruebas de arquitectura (artículos I, II y III) | `arch()` de Pest o prueba de PHPUnit que revise los `use` (D-12) | T-17 | La prueba pasa, y falla si se introduce una importación prohibida | pendiente | — |
| T-20 | `infra`: `docker compose up` levanta el motor vacío, la API y el frontend | Contextos de build relativos; migraciones al arrancar la API (I-08) | T-17 | Un comando; la base arranca sin tablas hasta que corren las migraciones | en curso | `db` verificado vacío (0 tablas, UTC) el 2026-10-13 a las 15:31. 16:17: `docker compose up --build` levanta `db` y `api`; la API aplica la migración al arrancar. Falta el servicio `app` (T-22) y el admin inicial al arrancar (T-17) |
| T-21 | `tool`: sembrador de demostración **por la API** (≥ 40 libros, 10 socios, 15 préstamos, 3 vencidos) | Script de Node o PHP que llama a `/api/v1` | T-17 | Tras ejecutarlo, `/loans/overdue` devuelve 3 líneas y el reporte tiene filas | pendiente | — |
| T-22 | `app`: login, catálogo, socios, prestar, devolver, vencidos, reporte | React + Vite | T-16 | Recorrido manual de punta a punta | pendiente | — |
| T-23 | `page`: sitio público estático de presentación | HTML/CSS estático (contenido no definido, M-04) | — | Se abre sin la API; correcto en móvil y escritorio | pendiente | — |

## Prioridad de entrega para el 2026-10-13 a las 17:00 (UTC-5)

1. `docs/spec-laravel/` (esta carpeta): traducción e interpretaciones.
2. Compose mínimo en `infra` (parte de T-20) para que la API tenga un PostgreSQL 16 vacío.
3. T-01, T-02; después T-03 a T-07, T-14, T-17 (iniciar sesión como admin con la base vacía).
4. T-08 a T-13 si el tiempo alcanza; después T-23, la tarea independiente más barata.
5. Último paso antes de las 16:45: actualizar las columnas **Estado** y **Enlace a la evidencia** de este archivo con el estado real.

Las tareas a las que no se llegue antes del plazo quedan **pendientes** en este archivo.
