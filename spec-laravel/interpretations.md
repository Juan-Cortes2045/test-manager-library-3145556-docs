# Registro de interpretaciones — Manager Library (Laravel + React)

El spec original (`../spec-python/`, idéntica a `../spec-.net/` salvo en tecnología) **no se edita**. Este registro recoge los vacíos
e inconsistencias encontrados al traducirlo, la evidencia de cada uno y la interpretación que sigue la implementación. Ninguna añade
tablas, columnas, endpoints ni funciones: el modelo de datos y el alcance quedan como están especificados.

Estados: **aplicada** (la implementación la sigue), **propuesta** (necesita confirmación del dueño del spec).
Las referencias de línea apuntan a `../spec-python/`; las mismas líneas existen en `../spec-.net/`.

## Hallazgos con interpretación

### I-01 · ADR-002 menciona un testigo de versión que el modelo de datos no tiene

- **Hallazgo:** ADR-002 dice «el libro lleva un testigo de versión», pero la tabla `book` no tiene columna de versión.
- **Evidencia:** `adr/adr-002-concurrencia-optimista.md:6` frente a `data-model.md:63-75` (columnas de `book`).
- **Interpretación:** gana el modelo de datos (artículo X). La concurrencia optimista se implementa con un `UPDATE` condicional
  `UPDATE book SET available_copies = available_copies - 1 WHERE id = ? AND available_copies > 0`; cero filas afectadas significa
  que ya no queda ejemplar. `ck_book_available_range` (`data-model.md` §4) sigue siendo la última barrera. No se añade ninguna columna.
- **Estado:** aplicada.

### I-02 · ADR-002 ofrece dos salidas ante un conflicto

- **Hallazgo:** ante un conflicto de versión, el servidor «responde 409 `BOOK_NOT_AVAILABLE` (o reintenta una vez si el ejemplar
  sigue disponible)».
- **Evidencia:** `adr/adr-002-concurrencia-optimista.md:7`.
- **Interpretación:** responder **409 `BOOK_NOT_AVAILABLE` sin reintento**. Con el `UPDATE` condicional de I-01, reintentar no aporta
  nada: si el `UPDATE` afectó cero filas, no hay ejemplar disponible. No se guarda nada (CA-06.3).
- **Estado:** aplicada.

### I-03 · El 401 cubre tres casos, pero solo existe un código de error

- **Hallazgo:** el 401 se usa para «sin token, token inválido o vencido, credenciales inválidas», pero el único código 401 listado es
  `INVALID_CREDENTIALS` (origen CA-01.2, es decir, el inicio de sesión).
- **Evidencia:** `api-contract.md:36` y `api-contract.md:46`.
- **Interpretación:** `POST /auth/login` con credenciales erróneas → `INVALID_CREDENTIALS`. Token ausente, mal formado, inválido o
  vencido en cualquier otro endpoint → 401 con el código `UNAUTHENTICATED` y el mismo cuerpo de error único. El mensaje nunca dice qué
  parte falló.
- **Estado:** propuesta (introduce un valor de `code` que no está en el contrato; el código de estado y la forma del cuerpo no cambian).

### I-04 · `GET /books/{id}/cover` se usa, pero no está en la tabla

- **Hallazgo:** el contrato dice que la portada se pinta con `GET /books/{id}/cover` (binario o 404), pero la tabla de endpoints solo
  lista `PUT` y `DELETE` para `/books/{id}/cover`, y no da rol para `GET`.
- **Evidencia:** `api-contract.md:121` frente a `api-contract.md:101-109`.
- **Interpretación:** tratarlo como parte del contrato: `GET /books/{id}/cover` → 200 con la imagen (`image/jpeg` o `image/png`) o 404
  (sin portada, libro inexistente o dado de baja). Rol: **autenticado** (igual que `GET /books/{id}`), por confirmar.
- **Estado:** propuesta (rol por confirmar).

### I-05 · «422 con el conteo actual» no dice dónde va el conteo

- **Hallazgo:** CA-06.4 exige un 422 «con el conteo actual», pero no define en qué campo.
- **Evidencia:** `spec.md:63`; cuerpo de error en `api-contract.md:19-28`; código `MEMBER_LIMIT_EXCEEDED` en `api-contract.md:56`.
- **Interpretación:** 422 `MEMBER_LIMIT_EXCEEDED` con
  `details: [{ "field": "member_id", "issue": "tiene N libros sin devolver; máximo 3" }]`, donde `N` es el número actual de líneas sin
  devolver. El `message` también indica el conteo, en español.
- **Estado:** aplicada.

### I-06 · `days_late` y `from`/`to` en `GET /loans` no están del todo definidos

- **Hallazgo:** CA-09.1 pide «días de retraso» sin unidad ni redondeo. `GET /loans` acepta `from`/`to` sin formato; solo el reporte
  los define como «fecha, ambos inclusive».
- **Evidencia:** `spec.md:80`; `api-contract.md:136`; `api-contract.md:160`; `api-contract.md:167`.
- **Interpretación:**
  - `days_late` = número de periodos de 24 horas **iniciados** desde `due_at` (`ceil((now - due_at) / 24 h)`), así toda línea que
    aparezca como vencida tiene `days_late >= 1`. Se calcula, nunca se guarda (artículo VII).
  - `from` y `to` en `GET /loans` siguen la convención del reporte: fechas `YYYY-MM-DD`, ambas inclusive, en UTC, es decir,
    `loaned_at >= from 00:00:00Z AND loaned_at < (to + 1 día) 00:00:00Z`. Ambas opcionales. `to < from` → 422 `INVALID_DATE_RANGE`.
- **Estado:** aplicada.

### I-07 · `active_loans` no dice si cuenta préstamos o libros

- **Hallazgo:** el detalle del socio devuelve `active_loans`, descrito como «conteo sin devolver».
- **Evidencia:** `api-contract.md:128`; RN-2 en `spec.md:95`; `spec.md:63` (CA-06.4 cuenta libros).
- **Interpretación:** `active_loans` = número de **líneas de préstamo sin devolver (libros)** del socio, el mismo número que CA-06.4
  compara con el límite de 3. El nombre del campo se mantiene como en el contrato.
- **Estado:** aplicada.

### I-08 · `infra` debe levantar la API y el frontend, que viven en otros repositorios

- **Hallazgo:** T-20 exige que `docker compose up` levante el motor vacío, la API y el frontend, pero son repositorios distintos y nada
  dice quién ejecuta las migraciones.
- **Evidencia:** `tasks.md:26`; artículo V en `constitution.md:37-41`; ADR-001.
- **Interpretación:** los seis repositorios se clonan uno al lado del otro; `infra/docker-compose.yml` usa contextos de build relativos
  (`../test-manager-library-3145556-api`, `../test-manager-library-3145556-app`). El entrypoint del contenedor de la API ejecuta
  `php artisan migrate --force` y después `php artisan library:bootstrap-admin`. `infra` no contiene DDL ni `CREATE TABLE`.
- **Estado:** aplicada (documentada en el README de `infra`).

## Inconsistencias menores registradas sin resolver

No bloquean la entrega y quedan para el dueño del spec.

| # | Inconsistencia | Evidencia |
|---|---|---|
| M-01 | RNF-1 mide la búsqueda con 5 000 libros usando el sembrador de `tool`, pero T-21 solo pide al sembrador 40 libros o más | `spec.md:105` frente a `tasks.md:27` |
| M-02 | CA-10.4 habla de renombrar una categoría, pero las categorías son una semilla fija «sin mantenimiento» y ningún endpoint las cambia | `spec.md:87` frente a `data-model.md:30` |
| M-03 | «Gana» se usa con dos alcances: `data-model.md` gana sobre los demás documentos, mientras que el motor gana sobre `data-model.md` | `data-model.md:7` frente a `constitution.md:64-66` |
| M-04 | El contenido del sitio público estático (`page`) no está definido más allá de «presentación», «sin API», «móvil y escritorio» | `tasks.md:29`; `arquitectura-panoramica.md:26` |
| M-05 | El artículo XI pide código y rutas en inglés, pero el propio spec tiene nombres de archivo en español (`arquitectura-panoramica.md`, los cuatro ADR). En `spec-laravel/` los nombres de archivo van en inglés y el contenido en español; los archivos originales no se renombran | `constitution.md:70`; listado de `../spec-python/` y `../spec-python/adr/` |
