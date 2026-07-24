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

## Por qué no todo está en verde

De las 19 aserciones de la suite, **10 fallan a propósito** y 9 pasan. No es una suite rota: cada fallo documenta un hallazgo real sobre la PokeAPI o una mala práctica de testing común, verificado contra la API en vivo (no simulado). Una suite 100% verde sobre una API que apenas se prueba con happy paths no dice nada; esta suite está diseñada para que cada ❌ sea información accionable.

| Test | Por qué falla | Categoría |
|---|---|---|
| `CP-004-F` | Compara `name` contra `"Pikachu"` (mayúscula) a propósito — demo de aserción case-sensitive | Fallo forzado (demo) |
| `CP-005` | Asume que `/pokemon` trae `description`; ese campo vive en `/pokemon-species`, un recurso distinto | Supuesto incorrecto |
| `CP-007` | Los errores 404 responden `Content-Type: text/plain`, no `application/json` | Mala práctica de la API |
| `CP-009` | El link `next` generado por el servidor ignora el `limit=0` solicitado y usa el default (20) | Bug real de la API |
| `CP-010` | `limit=0` no se valida ni se respeta: la API igual devuelve 20 resultados | Bug real de la API |
| `CP-014` | `height` se asume en metros; la API lo expresa en decímetros (confusión de unidades muy común) | Supuesto incorrecto |
| `CP-016` | SLA de 10ms, imposible de cumplir sobre HTTPS a un tercero — ilustra el riesgo de un SLA no negociado con una línea base real | Mala práctica de testing |
| `CP-017` | La API no expone headers de rate-limit (`X-RateLimit-Remaining`) para que el consumidor gestione su cuota | Mala práctica de la API |
| `CP-019` | Conteo hardcodeado (100) de Pokémon tipo eléctrico; el valor real crece con cada generación (hoy 114) | Mala práctica de testing |
| `CP-020` | Se asume orden alfabético; la API devuelve el listado ordenado por ID de Pokédex | Supuesto incorrecto |

## Cobertura de pruebas

### 1. `GET - Pokemon por Id / Nombre` — Happy path + contrato de datos

| Test | Qué valida |
|---|---|
| `CP-003` | Código de estado 200 OK |
| `CP-004` | Tipos de datos correctos (`id` numérico, `name` string, `abilities` array) |
| `CP-005` | El JSON contiene las llaves obligatorias del contrato (`id`, `name`, `sprites`, `types`) **+ ❌ hallazgo:** también asume `description`, un campo que en realidad vive en `/pokemon-species`, no en `/pokemon` — error común al integrar ambos recursos |
| `CP-004-F` | **Fallo intencional** — compara `name` contra `"Pikachu"` (mayúscula) a propósito. Se deja en la colección como demo de que Newman reporta correctamente los fallos y de la importancia de las aserciones case-sensitive. |

### 2. `GET - Not Found` — Manejo de errores

| Test | Qué valida |
|---|---|
| `CP-006` | Código de estado 404 cuando el recurso no existe, y el cuerpo contiene un mensaje de error controlado (`"Not Found"`), no un stacktrace ni un 500 |
| `CP-007` | ❌ **hallazgo:** un error debería declarar `Content-Type: application/json` para que el consumidor lo parsee de forma uniforme; la PokeAPI responde `text/plain` en sus 404 |

### 3. `GET - Paginacion (limit / offset) - Caso limite (limit=0)` — Contrato de listados

| Test | Qué valida |
|---|---|
| `CP-008` | Código de estado 200 OK |
| `CP-009` | La respuesta trae el contrato de paginación (`count`, `next`, `previous`, `results`), `previous` es `null` en la primera página. ❌ **hallazgo:** el `next` generado por el servidor debería respetar `limit=0`, pero la API lo ignora y arma el link con `limit=20` (el default) |
| `CP-010` | ❌ **hallazgo:** `limit=0` debería devolver un array vacío (o un 400), pero la API no valida el valor e igual retorna 20 resultados |
| `CP-011` | Cada elemento del listado trae `name` (string no vacío) y `url` con el formato esperado |

**Por qué importa:** la paginación es de los contratos que más silenciosamente se rompen. Este escenario usa `limit=0` a propósito — un valor de borde que casi nadie prueba — y expone que la PokeAPI no valida el parámetro: en vez de rechazarlo o devolver una página vacía, aplica el default (20) sin avisar. Sin una prueba de este tipo, un cliente que arme `limit` dinámicamente (p. ej. a partir de un filtro que devuelve 0 resultados) recibiría datos que no pidió.

### 4. `GET - Pokemon por ID numerico` — Consistencia entre identificadores

| Test | Qué valida |
|---|---|
| `CP-012` | Código de estado 200 OK |
| `CP-013` | Buscar por ID `25` devuelve el mismo recurso que buscar por nombre `"pikachu"` (mismos `id`/`name`) |
| `CP-014` | Tipos de datos numéricos propios del recurso (`height`, `weight`, `base_experience`). ❌ **hallazgo:** asume `height` en metros (0.4), pero la PokeAPI lo expresa en decímetros (4) — confusión de unidades muy común al integrar con esta API |

**Por qué importa:** muchas APIs exponen el mismo recurso por múltiples claves (ID, slug, nombre). Si ambos caminos no son consistentes, cualquier integración que cachee por una clave y consulte por otra mostrará datos contradictorios.

### 5. `GET - Rendimiento (Response Time)` — Prueba de performance

| Test | Qué valida |
|---|---|
| `CP-015` | Código de estado 200 OK |
| `CP-016` | ❌ **hallazgo (mala práctica de testing):** el tiempo de respuesta debe ser menor a **10ms**, un SLA imposible sobre HTTPS a un tercero. Se deja así a propósito para ilustrar qué pasa cuando se fija un presupuesto de performance sin medir antes una línea base real |
| `CP-017` | El header `Content-Type` declara `application/json`. ❌ **hallazgo:** también espera un header `X-RateLimit-Remaining`; la PokeAPI no expone ningún header de rate-limiting para que el consumidor gestione su cuota |

**Por qué importa:** una API "correcta" pero lenta igual rompe la experiencia del usuario. Fijar un presupuesto de rendimiento (SLA) convierte la performance en un criterio de aprobación/rechazo automático — pero si ese presupuesto no se basa en una medición real, el resultado es una prueba que falla siempre y que el equipo termina ignorando (la antesala del "alert fatigue").

### 6. `GET - Recurso relacionado (Type)` — Integridad referencial

| Test | Qué valida |
|---|---|
| `CP-018` | Código de estado 200 OK |
| `CP-019` | El contrato de `damage_relations` trae sus 6 llaves y `pokemon` es un array no vacío. ❌ **hallazgo (mala práctica de testing):** también espera exactamente 100 Pokémon del tipo, un conteo hardcodeado sobre un dataset que crece con cada generación (hoy son 114) |
| `CP-020` | Pikachu existe dentro del listado de Pokémon del tipo `electric`. ❌ **hallazgo:** también asume que el listado viene ordenado alfabéticamente; la PokeAPI en realidad lo ordena por ID de Pokédex |

**Por qué importa:** valida que las relaciones entre recursos sean coherentes (un tipo realmente contiene a los Pokémon que dice contener), no solo que cada recurso individual responda bien de forma aislada. También muestra dos errores clásicos al escribir aserciones sobre datos de terceros: hardcodear conteos que cambian con el tiempo, y asumir un orden que la API nunca prometió.

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
│   └── POKEMONS.postman_collection.json   # colección Postman (6 requests, 19 aserciones: 9 pasan, 10 fallan a propósito)
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
