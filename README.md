# Pruebas QA — PokeAPI

Suite de pruebas automatizadas de API, ejecutada con **Newman** (CLI de Postman) sobre la [PokeAPI](https://pokeapi.co/), sin depender de la interfaz gráfica de Postman. Funciona igual en Windows y Linux porque toda la ejecución vive en Node.js.

## Por qué importa esta suite

Probar una API pública de solo lectura como la PokeAPI parece trivial, pero sirve como caso de estudio realista de los pilares de QA de APIs: cada escenario aquí representa una categoría de riesgo que existe en cualquier API productiva, no solo en esta.

| Riesgo que se busca detectar | Cómo lo cubre la suite |
|---|---|
| La API rompe su contrato (cambia un campo, un tipo de dato) | Validación de esquema y tipos en cada respuesta |
| Errores no controlados devuelven 500 en vez de un error manejado | Prueba dedicada al camino 404 |
| La paginación deja de respetar los parámetros del cliente | Escenario de `?limit=` / `?offset=` |
| Dos formas de acceder al mismo recurso devuelven datos distintos | Comparación búsqueda por ID vs. por nombre |
| La API se degrada en tiempo de respuesta sin que nadie lo note | Presupuesto de rendimiento (`< 500ms`) |
| Los datos relacionados pierden integridad referencial | Validación de que un tipo (`electric`) realmente contiene a sus Pokémon |

En una presentación, esto demuestra que la suite no es una lista de "requests que devuelven 200", sino un conjunto de pruebas con intención: cada una protege contra una forma específica de que la API falle en producción.

## Cobertura de pruebas

### 1. `GET - Pokemon por Id / Nombre` — Happy path + contrato de datos

| Test | Qué valida |
|---|---|
| `CP-003` | Código de estado 200 OK |
| `CP-004` | Tipos de datos correctos (`id` numérico, `name` string, `abilities` array) |
| `CP-005` | El JSON contiene las llaves obligatorias del contrato (`id`, `name`, `sprites`, `types`) |
| `CP-004-F` | **Fallo intencional** — compara `name` contra `"Pikachu"` (mayúscula) a propósito. Se deja en la colección como demo de que Newman reporta correctamente los fallos y de la importancia de las aserciones case-sensitive; queda como el único ❌ esperado en cada corrida. Elimínalo si quieres un reporte 100% en verde. |

### 2. `GET - Not Found` — Manejo de errores

| Test | Qué valida |
|---|---|
| `CP-006` | Código de estado 404 cuando el recurso no existe |
| `CP-007` | El cuerpo de la respuesta contiene un mensaje de error controlado (`"Not Found"`), no un stacktrace ni un 500 |

### 3. `GET - Paginacion (limit / offset)` — Contrato de listados

| Test | Qué valida |
|---|---|
| `CP-008` | Código de estado 200 OK |
| `CP-009` | La respuesta trae el contrato de paginación (`count`, `next`, `previous`, `results`) y `previous` es `null` en la primera página |
| `CP-010` | La cantidad de resultados devueltos coincide exactamente con el `limit` solicitado |
| `CP-011` | Cada elemento del listado trae `name` (string no vacío) y `url` con el formato esperado |

**Por qué importa:** la paginación es de los contratos que más silenciosamente se rompen (p. ej. un cambio de backend que ignora `limit` y siempre devuelve 20). Sin esta prueba, un frontend que pagina por lotes empezaría a mostrar datos duplicados o incompletos sin que nadie lo detecte hasta que un usuario se queje.

### 4. `GET - Pokemon por ID numerico` — Consistencia entre identificadores

| Test | Qué valida |
|---|---|
| `CP-012` | Código de estado 200 OK |
| `CP-013` | Buscar por ID `25` devuelve el mismo recurso que buscar por nombre `"pikachu"` (mismos `id`/`name`) |
| `CP-014` | Tipos de datos numéricos propios del recurso (`height`, `weight`, `base_experience`) |

**Por qué importa:** muchas APIs exponen el mismo recurso por múltiples claves (ID, slug, nombre). Si ambos caminos no son consistentes, cualquier integración que cachee por una clave y consulte por otra mostrará datos contradictorios.

### 5. `GET - Rendimiento (Response Time)` — Prueba de performance

| Test | Qué valida |
|---|---|
| `CP-015` | Código de estado 200 OK |
| `CP-016` | El tiempo de respuesta es menor a **500ms** (`pm.response.responseTime`) |
| `CP-017` | El header `Content-Type` declara `application/json` |

**Por qué importa:** una API "correcta" pero lenta igual rompe la experiencia del usuario. Fijar un presupuesto de rendimiento (SLA) convierte la performance en un criterio de aprobación/rechazo automático, no en algo que se descubre manualmente después.

### 6. `GET - Recurso relacionado (Type)` — Integridad referencial

| Test | Qué valida |
|---|---|
| `CP-018` | Código de estado 200 OK |
| `CP-019` | El contrato de `damage_relations` trae sus 6 llaves y `pokemon` es un array no vacío |
| `CP-020` | Pikachu existe dentro del listado de Pokémon del tipo `electric` |

**Por qué importa:** valida que las relaciones entre recursos sean coherentes (un tipo realmente contiene a los Pokémon que dice contener), no solo que cada recurso individual responda bien de forma aislada.

## Requisitos previos

- [Node.js](https://nodejs.org/) v16 o superior (requerido por Newman) y `npm` — funciona igual en Windows y Debian/Linux.
- Conexión a internet (las pruebas llaman a la PokeAPI real, no hay mocks).

## Instalación

Desde la raíz del proyecto:

```bash
npm install
```

Esto instala `newman` y `newman-reporter-htmlextra` como dependencias de desarrollo (declaradas en `package.json`, no requiere instalación global).

## Ejecución local

**Correr todas las pruebas en consola:**

```bash
npm run test
```

**Correr las pruebas y generar un reporte visual en HTML:**

```bash
npm run test:report
```

El reporte se guarda en `reports/report.html`. Ábrelo con doble clic, o desde terminal:

```bash
# macOS / Linux
open reports/report.html      # macOS
xdg-open reports/report.html  # Linux

# Windows
start reports/report.html
```

## Estructura del proyecto

```
pruebas-qa-pokeAPI/
├── collections/
│   └── POKEMONS.postman_collection.json   # colección Postman (6 requests, 18 aserciones)
├── reports/                                # reportes HTML generados (no versionado)
├── package.json                            # scripts npm run test / test:report
└── README.md
```

## Cómo se integraron los nuevos casos a la colección

Los 4 escenarios nuevos son objetos JSON dentro del array `"item"` de la colección, con la misma forma que los 2 requests originales (`name`, `event.script.exec` con las pruebas en JavaScript/Chai, y `request.url`). Para agregar un caso adicional a mano:

1. Abre `collections/POKEMONS.postman_collection.json` y ubica el cierre del último elemento del array `"item"` (la línea `}` justo antes del `]` final).
2. Cambia esa `}` por `},` para separarlo del nuevo bloque.
3. Pega el nuevo objeto `{ "name": ..., "event": [...], "request": {...}, "response": [] }` completo.
4. Verifica que el archivo siga siendo JSON válido (por ejemplo con `python3 -m json.tool collections/POKEMONS.postman_collection.json > /dev/null`, o pegándolo en cualquier validador de JSON) antes de correr `npm run test`.

El punto crítico es no dejar comas sobrantes ni faltantes entre elementos del array — es el error más común al editar colecciones de Postman a mano.
