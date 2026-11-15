---
tags: [otca, opentelemetry, resumen]
---

# OpenTelemetry Certified Associate (OTCA)

Resumen del curso de preparación de KodeKloud, redactado en español y con notas sobre los matices que suelen caer en el examen.

1. Conceptos de observabilidad
2. Conceptos base de OpenTelemetry
3. Anatomía del span y propagación de contexto
4. Instrumentación
5. Data model de métricas
6. Registrar mediciones
7. Logs
8. Fundamentos del Collector
9. Componentes del Collector
10. Transformar telemetría (OTTL, sampling, schemas)
11. OpenTelemetry en Kubernetes
12. Depuración, operación y escalado
13. SLIs y SLOs
14. Chuleta final

Notas propias a partir de los apuntes públicos del curso (notes.kodekloud.com). Diagramas hechos para este resumen.

# 1. Conceptos de observabilidad

## Aplicaciones distribuidas

Una petición de usuario (por ejemplo, un checkout) recorre muchos servicios: frontend, caché, catálogo en Kubernetes, precios alimentados por Kafka, pagos de un tercero, envíos. Cuando algo falla o va lento, el problema puede estar en cualquier salto. Sin telemetría que conecte esos saltos, depurar es adivinar.

## Las tres señales principales

| Señal | Qué es | Para qué sirve |
|---|---|---|
| Métricas | Valores numéricos agregados en el tiempo | Tendencias, alertas, detectar anomalías (latencia, errores, throughput, CPU, saturación) |
| Logs | Eventos con timestamp y contexto | Detalle de un fallo concreto, stack traces |
| Trazas | Recorrido de una petición entre servicios con tiempos | Encontrar qué salto es lento o falla |

Analogía del curso: el médico. Los síntomas son las métricas, el historial son los logs y las pruebas de punta a punta son las trazas. Además existen señales de contexto: **baggage** (clave-valor que viaja con la petición), **profiling** (dónde gasta CPU/memoria el código) y eventos de negocio.

La clave práctica es **correlacionar**: que las tres señales compartan identificadores (`trace_id`, `service.name`) para saltar de una a otra.

## Monitorización vs observabilidad

|  | Monitorización | Observabilidad |
|---|---|---|
| Pregunta | ¿Qué está pasando? | ¿Por qué está pasando? |
| Señales | Métricas, umbrales, alertas | Métricas, logs, trazas, profiles, baggage |
| Ideal para | Fallos conocidos, SLAs/SLIs | Fallos nuevos o emergentes, causa raíz |
| Flujo | Alerta → runbook → arreglo | Detectar → explorar → hipótesis → arreglo |

No compiten: la monitorización avisa del síntoma y la observabilidad permite investigarlo. Juntas reducen el MTTR.

## Antes de OpenTelemetry

Dos modelos de recogida: **push** (la app o un agente envía los datos) y **pull** (un sistema externo hace scrape, como Prometheus con `/metrics`). El problema era la fragmentación: cada vendor con su agente y su SDK, cambiar de backend obligaba a reinstrumentar, y era difícil mandar los mismos datos a varios destinos. OpenTelemetry nace para estandarizar eso.

# 2. Conceptos base de OpenTelemetry

## Qué es OTel

Un framework open source de la CNCF, neutral respecto a vendors y multilenguaje, para **generar, recoger, procesar y exportar** telemetría. Abreviatura correcta: OTel. **No almacena ni visualiza** datos: eso lo hace el backend que elijas (Prometheus, Jaeger, Zipkin, un APM comercial…).

Características: vendor-neutral, open source, independiente del lenguaje, portable, flexible y compatible con muchos backends.

## Objetivo, misión, visión y valores

- **Objetivo:** instrumentación fácil, consistente y universal.
- **Misión:** telemetría de calidad y portable para una observabilidad efectiva.
- **Visión:** fácil de adoptar, universal, vendor-neutral y **componible** (usas solo las piezas que necesitas).
- **Valores de ingeniería:** compatibilidad, estabilidad (APIs retrocompatibles), resiliencia (la instrumentación no debe tumbar la app) y rendimiento (overhead mínimo).

## La especificación

Define el comportamiento que deben cumplir todas las implementaciones:

- APIs y SDKs de cada señal.
- Semantic conventions (nombres estándar de atributos).
- OTLP (protocolo de transporte).
- Propagación de contexto y baggage.
- Guías de instrumentación, configuración (variables de entorno) y versionado.

Niveles de madurez: **Draft** (en diseño) → **Experimental** (para probar, puede cambiar) → **Stable** (retrocompatible, apto para producción) → **Deprecated** (se eliminará).

## Semantic conventions

Nombres estándar para que el mismo concepto se llame igual en todos los servicios, lenguajes y señales. Facilitan correlacionar, consultar y cambiar de backend.

- Formato: minúsculas separadas por puntos, con prefijo de dominio: `http.*`, `db.*`, `k8s.*`, `cloud.*`, `messaging.*`, `rpc.*`, `exception.*`.
- Cubren resource attributes, atributos de spans, eventos, logs y nombres/unidades de métricas.
- **Resource** para lo estable del servicio (`service.name`, `service.version`, `k8s.pod.uid`); **span/evento** para lo de cada operación (`http.request.method`, `db.statement`).
- Ojo: `db.system` es el tipo de base de datos (postgresql), no su nombre.
- Antes de inventar un atributo propio, busca si ya existe uno en las convenciones.

> [!note]
> El curso usa nombres antiguos como `http.method` y `http.status_code`. En las convenciones estables actuales son `http.request.method` y `http.response.status_code`. Conoce los dos.

## Componentes principales

| Capa | Piezas |
|---|---|
| Estándares | Especificación, OTLP, semantic conventions |
| Código | API, SDK, instrumentation libraries, agentes de autoinstrumentación |
| Procesado | Collector (receivers → processors → exporters) |
| Despliegue | Operator de Kubernetes, Helm charts, distribuciones |

## Arquitectura de punta a punta

![[otca-diagrama-01.svg]]

*Flujo típico. El Collector es opcional: el SDK puede exportar directo al backend.*

1. El código (o una instrumentation library) llama a la **API**.
2. El **SDK** implementa la API: sampling, procesado, batching, detección del resource.
3. El **exporter** (dentro del SDK) envía por OTLP al Collector o directo al backend.
4. El **Collector** recibe, transforma y exporta a uno o varios backends.
5. El backend almacena y visualiza.

Cada señal tiene sus piezas: trazas (`TracerProvider` → `Tracer` → `Span`, con `SpanProcessor`, sampler y exporter), métricas (`MeterProvider` → `Meter` → instrumentos, con views/agregaciones y reader/exporter) y logs (`LoggerProvider` → `Logger` → `LogRecord`, con log processors y exporters). Los **propagators** serializan el contexto entre procesos (W3C Trace Context).

## API vs SDK

| API (el "qué") | SDK (el "cómo") |
|---|---|
| Interfaces estables para crear spans, métricas, logs y propagar contexto | Implementación: sampling, procesado, batching, agregación, exportación |
| Si no hay SDK, funciona en modo **no-op**: no falla, simplemente descarta | Se configura al arrancar la aplicación y se puede cambiar sin tocar la instrumentación |

- **Las librerías dependen solo de la API.** Nunca deben exponer tipos del SDK ni configurar exporters o samplers: eso lo decide quien despliega la aplicación.
- Roles: quien escribe instrumentación usa la API; quien opera la app configura el SDK; quien escribe plugins implementa exporters, processors o samplers.
- Principios de diseño de los clientes: fácil de usar, uniforme entre lenguajes, flexible, funcional por defecto y extensible.
- Estructura modular de cada cliente: API, SDK, semantic conventions y contrib (integraciones de la comunidad).
- OTel es un *cross-cutting concern*: atraviesa toda la aplicación, lo que tensiona el principio de separación de responsabilidades. Separar API y SDK lo mitiga.
- Las **distribuciones** son empaquetados de componentes upstream con valores por defecto de un vendor, no forks.

## Estado de madurez por señal

| Señal | API | SDK | OTLP |
|---|---|---|---|
| Trazas | Stable | Stable | Stable |
| Métricas | Stable | Stable (algunas partes evolucionando) | Stable |
| Logs | Stable (Bridge API) | Stable | Stable |
| Baggage | Stable | Stable | n/a |
| Profiles | En desarrollo / experimental |  |  |

## OTLP

Antes cada herramienta tenía su formato (Prometheus en texto, Jaeger en Thrift y luego gRPC, Zipkin en Thrift/JSON, OpenCensus en protobuf). OTLP los unifica. Es tres cosas a la vez:

- Un **data model**: mensajes protobuf para trazas, métricas y logs.
- Un **protocolo de red**: cómo se estructuran y envían.
- Una **especificación de transporte**: gRPC o HTTP, con compresión, reintentos y gestión de errores.

Todas las señales usan la misma jerarquía: **Resource** (quién emite) → **InstrumentationScope** (qué librería, nombre y versión) → datos de la señal. El Resource se fija al crear el provider y es inmutable. Protobuf da mensajes pequeños, parseo rápido y evolución del esquema (los campos desconocidos se ignoran).

|  | gRPC + protobuf | HTTP + protobuf | HTTP + JSON |
|---|---|---|---|
| Puerto por defecto | **4317** | **4318** | **4318** |
| Transporte | Requiere HTTP/2 | HTTP/1.1 o HTTP/2 | HTTP/1.1 o HTTP/2 |
| Rutas | Un endpoint, servicios gRPC | `/v1/traces`, `/v1/metrics`, `/v1/logs` | Las mismas |
| Content-Type | Lo gestiona gRPC | `application/x-protobuf` | `application/json` |
| Rendimiento | El mejor | Algo peor | El peor |
| Cuándo | Producción, SDK → Collector | Si gRPC está bloqueado (proxies, firewalls) | Depuración, payloads legibles |

> [!note]
> Con HTTP configuras la URL base y el SDK añade `/v1/traces` etc. Con gRPC basta `host:4317`.

# 3. Anatomía del span y propagación de contexto

## Trazas y spans

Una **traza** es el recorrido completo de una petición. Un **span** es una unidad de trabajo dentro de ella: atender una petición HTTP, una query, una llamada a otro servicio, parsear un JSON, consumir un mensaje. Los spans forman un árbol: el **root span** no tiene padre, cada llamada aguas abajo crea un hijo, y los hermanos comparten padre. En la vista *waterfall*, la longitud de cada barra es la duración y el anidamiento muestra la dependencia.

![[otca-diagrama-02.svg]]

*Ejemplo de waterfall: el root abarca todo; el checkout acumula casi toda la latencia y el email falla.*

## Qué contiene un span

- **Nombre**, **kind**, **inicio y fin** (duración = fin − inicio).
- **SpanContext** y **parent_span_id** (vacío en el root).
- **Atributos**, **eventos**, **links** y **status**.
- El **Resource** del servicio que lo emite y, opcionalmente, un `schema_url`.

## Nombre del span

Corto, genérico y de **baja cardinalidad**: describe la operación (`processPayment`, `getCart`, `GET /pedido/{id}`), no la petición concreta. Los datos concretos (id de pedido, usuario) van a atributos. Patrón recomendado: verbo + objeto.

## SpanContext: lo que viaja entre servicios

| Campo | Tamaño | Notas |
|---|---|---|
| `trace_id` | 16 bytes (32 hex) | Igual en todos los spans de la traza; si cambia, es otra traza |
| `span_id` | 8 bytes (16 hex) | Único dentro de su traza |
| `trace_flags` | 1 byte | `01` = sampled, `00` = no |
| `trace_state` | Lista `clave=valor` | Datos de vendors; límite de 512 caracteres; la clave `ot=` está reservada para OTel |

## Span kind

| Kind | Dirección | Uso |
|---|---|---|
| `CLIENT` | Saliente, síncrono | Llamada HTTP/RPC, query a BD |
| `SERVER` | Entrante, síncrono | Servidor que atiende una petición |
| `PRODUCER` | Saliente, asíncrono | Publicar en Kafka o una cola |
| `CONSUMER` | Entrante, asíncrono | Procesar un mensaje de la cola |
| `INTERNAL` | Local | Trabajo dentro del proceso. **Es el valor por defecto** |

Cada span tiene un solo rol. Cada llamada saliente lleva su propio span CLIENT (tres queries = tres spans). Con contexto propagado, el SERVER del receptor queda como hijo del CLIENT del emisor.

## Span links

Relacionan spans que no tienen una relación padre-hijo directa: trabajo asíncrono, paralelo o en lote. Ejemplos: workers de un map-reduce enlazados al job que los lanzó, un agregador que procesa mensajes de varias trazas, consumidores independientes (email, inventario) del mismo pedido. Un link apunta a un SpanContext y puede llevar atributos.

| Modelo | Cuándo |
|---|---|
| Padre-hijo | Cadenas síncronas: A llama a B |
| Links | Trabajo asíncrono, paralelo o independiente con un disparador común |

## Status

- `UNSET`: por defecto. Éxito implícito; es lo normal.
- `ERROR`: fallo detectado (excepción, timeout, 5xx…).
- `OK`: éxito marcado **explícitamente** por quien desarrolla la app, por ejemplo para que un error controlado no cuente como fallo. Las instrumentation libraries no lo ponen.

> [!note]
> Matiz de la spec que el curso simplifica: en un span `SERVER`, un 4xx **no** marca ERROR (el fallo es del cliente); un 5xx sí. En un `CLIENT`, los dos lo marcan.

Las excepciones se registran como un **evento** llamado `exception` con `exception.type`, `exception.message` y `exception.stacktrace` (`record_exception()`).

## Eventos vs atributos

|  | Evento | Atributo |
|---|---|---|
| Tiempo | Un único timestamp, sin duración | Ninguno: aplica a todo el span |
| Para qué | Algo que pasó en un momento concreto (excepción, reintento, cache miss) | Contexto descriptivo (método HTTP, `order_id`, tipo de pago) |
| Ejemplo | `span.add_event("retry", {...})` | `span.set_attribute("payment.method", "card")` |

Si necesitas expresar un **intervalo**, no uses un evento: crea un span hijo. Los eventos solo existen dentro de un span.

**Reglas de atributos:** claves de tipo string no nulas; valores de tipo string, booleano, entero, float o arrays de estos; nunca `null` (si no hay valor, no pongas el atributo). Conviene ponerlos pronto, para que el sampler pueda usarlos.

## Resource

Describe el origen de la telemetría (servicio, host, contenedor, pod, cloud). Se configura una vez al iniciar el SDK y se aplica a todos los spans, métricas y logs.

- Atributos por defecto: `service.name` (si no lo pones, **`unknown_service`**), `telemetry.sdk.name`, `telemetry.sdk.language` y `telemetry.sdk.version`.
- Los **resource detectors** añaden SO, host, proceso, contenedor, Kubernetes, cloud y región.
- Para añadir los tuyos: `Resource.create({...})` en código, o las variables `OTEL_RESOURCE_ATTRIBUTES=deployment.environment=prod,team=pagos` y `OTEL_SERVICE_NAME`.

## Baggage

Pares clave-valor pequeños que viajan con la petición entre servicios, en la cabecera W3C `baggage`. Se diferencia de los atributos en que **se propaga** aguas abajo, mientras que los atributos se quedan en su span. **No se añade solo a la telemetría**: para verlo en un span hay que leerlo y copiarlo como atributo (o usar un `BaggageSpanProcessor`).

- Usos: `correlation_id`, `cart_id`, región, tier, routing.
- Seguridad: viaja en texto plano por cabeceras y se puede manipular o filtrar fuera de tu red. **Nunca** metas PII, credenciales ni tokens.
- Mantenlo pequeño (pesa en cada petición) y con nombres estandarizados.

## Propagación de contexto

Para que los spans de distintos servicios formen una sola traza, el SpanContext (y el baggage) tiene que viajar con la petición. OTel lo hace con **propagators**:

- **Carrier**: el medio donde viaja (cabeceras HTTP, metadata de gRPC, atributos de un mensaje).
- **inject**: el emisor escribe el contexto actual en el carrier. **extract**: el receptor lo lee y lo restaura como contexto activo.
- **TextMapPropagator**: la interfaz que hace ese inject/extract, con funciones setter/getter.
- **Propagator global**: configuración de la app que decide qué formatos se usan (`OTEL_PROPAGATORS`; por defecto `tracecontext,baggage`).

| Propagator | Cabeceras | Notas |
|---|---|---|
| W3C Trace Context | `traceparent`, `tracestate` | Por defecto en OTel |
| W3C Baggage | `baggage` | Por defecto junto al anterior |
| B3 | `X-B3-TraceId`, `X-B3-SpanId`, `X-B3-Sampled` | Zipkin y sistemas antiguos |
| Composite | Varios | Combina formatos: entornos mixtos o migraciones |
| Custom | Lo que quieras | Implementando `TextMapPropagator` |

Formato de `traceparent`: `00-<trace_id 32 hex>-<parent span_id 16 hex>-<flags>`, por ejemplo `00-4bf92f35…4736-00f067aa0ba902b7-01`.

# 4. Instrumentación

## Los tres enfoques

| Enfoque | Cambios de código | Cobertura | Lógica de negocio | Cuándo |
|---|---|---|---|---|
| Zero-code (auto) | Ninguno; agente o wrapper | Amplia: HTTP, BD, mensajería | Baja | Visibilidad rápida, apps que no puedes tocar |
| Library-based | Mínimos (paquetes de instrumentación) | Librerías concretas | Media | Una dependencia necesita más visibilidad |
| Manual (code-based) | Sí | Lo que tú decidas | Alta | Flujos críticos, atributos de negocio, async propio |

Estrategia recomendada: empieza con zero-code para tener cobertura amplia, revisa las trazas buscando huecos (falta de spans de negocio, jerarquías raras, flujos asíncronos rotos) y añade instrumentación manual o de librería donde haga falta. Las tres se combinan sin problema.

> [!note]
> En los quizzes, "library-based" a veces significa "importar las librerías de OTel en tu código" (o sea, code-based). Mira las otras opciones para saber a cuál se refieren.

## Instrumentación manual: el flujo

1. Configurar un `TracerProvider` (SDK) con su Resource, processors y exporters.
2. Obtener un `Tracer`: `trace.get_tracer(__name__)`. El nombre es el *instrumentation scope*.
3. Crear spans con `start_as_current_span` (Python), `spanBuilder().startSpan()` (Java) o `startSpan()` (JS).
4. El sampler decide si se registra.
5. El `SpanProcessor` recibe el span al terminar, aplica los `SpanLimits` y lo pasa al exporter.
6. El exporter lo envía al Collector o al backend.

Métodos clave: `set_attribute`, `add_event`, `record_exception`, `set_status`. Cierra los spans siempre (mejor con `with`). Sin `TracerProvider` configurado, todo es no-op y no da errores. Configura un único provider por aplicación.

```python
provider = TracerProvider(resource=Resource.create({"service.name": "tienda"}))
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)
with tracer.start_as_current_span("processPayment") as span:
    span.set_attribute("payment.method", "card")
```

## Span processors y exporters

- **SimpleSpanProcessor**: exporta cada span en cuanto termina. Para depurar.
- **BatchSpanProcessor**: acumula y exporta por lotes. **El de producción.**
- Exporters: Console (depuración), OTLP (Collector o backends compatibles), Zipkin y otros.
- Métodos del exporter: `export(spans)`, `shutdown()` (llámalo al apagar la app para no perder datos) y `force_flush()` (vacía el buffer al momento).

## Sampling en el SDK (head sampling)

La decisión se toma al crear el span y viaja en el flag `sampled` de `traceparent`. Así toda la traza respeta la misma decisión y no quedan trazas a medias.

| Sampler | Comportamiento |
|---|---|
| `AlwaysOn` | Lo muestrea todo (desarrollo) |
| `AlwaysOff` | Nada; mantiene la instrumentación pero sin datos |
| `TraceIdRatioBased(0.2)` | Un porcentaje fijo de trazas, calculado a partir del trace_id |
| `ParentBased(root=…)` | Respeta la decisión del padre; si no hay padre, usa el sampler `root` |

Combinación típica en producción: `ParentBased(root=TraceIdRatioBased(0.1))`, que en variables de entorno es `OTEL_TRACES_SAMPLER=parentbased_traceidratio` con `OTEL_TRACES_SAMPLER_ARG=0.1`. Solo `ParentBased` acepta otro sampler dentro. Si necesitas decidir con la traza completa (errores, latencia), eso es tail sampling en el Collector.

## Instrumentation libraries

Paquetes `opentelemetry-instrumentation-<librería>` que enganchan librerías de terceros (requests, Flask, Django, FastAPI, SQLAlchemy, Kafka, Celery, boto3…). Por ejemplo, la de `requests` envuelve la llamada: abre un span CLIENT, ejecuta la petición original y cierra el span con atributos semánticos.

| Hacen | No hacen |
|---|---|
| Enganchar la librería y crear spans o métricas | Configurar el SDK, los processors o los exporters |
| Rellenar atributos según las semantic conventions | Depender de una implementación concreta del SDK |
| Usar solo la API | Exportar ni almacenar |

Reparto de responsabilidades: la librería decide **qué** se recoge; la aplicación decide **cómo** se procesa y se exporta.

## Zero-code: mecanismos

- **Monkey patching** (Python, JS): en runtime se reemplazan o envuelven funciones de las librerías.
- **Manipulación de bytecode** (Java, .NET): el agente reescribe las clases al cargarlas.
- **eBPF**: observa eventos del kernel (red, syscalls). Independiente del lenguaje y nada intrusivo.

Se configura con variables de entorno (lo más común), propiedades de sistema (Java/.NET) o argumentos de arranque.

### Java

`java -javaagent:opentelemetry-javaagent.jar -jar app.jar`. Se configura con `-Dotel.*` o con `OTEL_*` (mejor en contenedores). El agente se puede extender: processors propios, enmascarar PII, desactivar instrumentaciones o instrumentar librerías internas.

| Variable | Para qué |
|---|---|
| `OTEL_SERVICE_NAME` | Nombre del servicio |
| `OTEL_RESOURCE_ATTRIBUTES` | Atributos extra del Resource |
| `OTEL_TRACES_EXPORTER` | `otlp`, `zipkin`, `console`… |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | A dónde enviar |
| `OTEL_PROPAGATORS` | `tracecontext,baggage`… |
| `OTEL_TRACES_SAMPLER` / `_ARG` | Sampler y su parámetro |
| `OTEL_SDK_DISABLED` | Apaga el SDK (útil en tests) |

### Python

1. `pip install opentelemetry-distro opentelemetry-exporter-otlp`
2. `opentelemetry-bootstrap -a install`: detecta las librerías instaladas e instala sus instrumentaciones.
3. `opentelemetry-instrument python app.py`: un `sitecustomize` configura el SDK desde las variables de entorno y llama a `.instrument()` en cada instrumentación instalada.

Opciones específicas de Python: `OTEL_PYTHON_DISABLED_INSTRUMENTATIONS` (para excluir instrumentaciones; no existe una lista de "solo estas") y `OTEL_PYTHON_LOG_CORRELATION=true`. Para controlar qué se instrumenta, instala solo los paquetes que quieras.

# 5. Data model de métricas

## Para qué sirven las métricas

Responden preguntas como si la app está sana ahora, si crece el tráfico, si algo se degrada o si hace falta escalar. Son baratas y rápidas para detectar y alertar, pero rara vez explican la causa: las métricas dicen *qué*, y las trazas y los logs, *por qué*. Una métrica tiene **nombre** (`container.cpu.usage`), **valores** con timestamp, **unidad** y **dimensiones** (atributos).

## Las cuatro propiedades estructurales

| Propiedad | Significado | Regla práctica |
|---|---|---|
| Monotonicidad | Si el valor solo puede crecer (monótona) o también bajar | Las tasas (`rate()`) solo tienen sentido sobre sumas monótonas; una bajada indica un reset |
| Temporality | **Cumulative** (total desde el inicio) o **delta** (lo ocurrido en el intervalo) | Usa la que espere el backend; Prometheus trabaja con cumulative |
| Agregación | Cómo se resumen las mediciones: Sum, LastValue (gauge), Histogram | Histogramas para latencias; sumas para conteos |
| Dimensiones | Atributos (labels en Prometheus) de cada medición | Vigila la cardinalidad: nada de ids sin límite |

**Temporality con un ejemplo de cuenta bancaria:** ingresas 1000, cobras 500 y gastas 200. Los deltas son +1000, +500 y −200; el acumulado es 1000, 1500 y 1300. La temporality cambia *cómo* se calcula una tasa, no *si* se puede calcular. Con cumulative hay que detectar los resets (si el valor actual es menor que el anterior, ha habido un reinicio).

## Histogramas y percentiles

![[otca-diagrama-03.svg]]

*Un histograma guarda conteos por bucket, no los valores. El p95 solo se sabe que está entre 0,5 y 1 s.*

- Guarda conteos por bucket, la suma y el número total (y opcionalmente min/max). Así puedes ver la forma de la distribución y calcular percentiles (p50, p90, p95, p99) sin almacenar cada medición.
- Buckets **explícitos** (límites fijos, los defines tú) o **exponenciales** (se adaptan solos a rangos amplios, con más resolución).
- El percentil que sale de un histograma de buckets explícitos es una **estimación**: interpola suponiendo un reparto uniforme dentro del bucket. Lo exacto es el porcentaje de valores por debajo de cada límite. Por eso conviene poner un límite justo en el umbral del SLO.

## Dimensiones y cardinalidad

Las dimensiones permiten **filtrar** (`service.name="cart"`), **agrupar** (una serie por servicio) y **agregar** (`avg … by server.address`). Cada combinación única de atributos es una serie temporal nueva.

| Cardinalidad | Ejemplos | Uso |
|---|---|---|
| Baja | Status HTTP, región, tier | Monitorización general y alertas; barata |
| Media | Nombre de servicio, tipo de instancia, modelo | Segmentar y ver tendencias |
| Alta | `user_id`, `request_id`, matrícula | Investigaciones puntuales; con mucho cuidado en métricas (mejor en trazas o logs) |

## Del instrumento al backend: tres modelos

| Modelo | Dónde | Qué contiene |
|---|---|---|
| **Event model** | Tu código | Las mediciones crudas de los instrumentos |
| **OTLP stream model** | En tránsito (SDK → Collector → backend) | Datos ya agregados: Sum, Gauge, Histogram, ExponentialHistogram, Summary (solo compatibilidad), con temporality, monotonicidad y timestamps de inicio y fin |
| **Timeseries model** | El backend | Series por combinación de nombre y atributos, listas para consultar |

Transformaciones que permite el modelo, en el SDK (con Views), en el Collector o en el backend:

- **Reagregación espacial**: quitar atributos y fusionar series, por ejemplo por región en lugar de por instancia. Sirve para bajar la cardinalidad.
- **Reagregación temporal**: juntar puntos en intervalos más largos, por ejemplo seis de 10 s en uno de 1 min (downsampling).
- Conversión **delta ↔ cumulative**, normalización de unidades (ns → s) y enriquecimiento con atributos.

El Collector suele ser el mejor sitio para aplicar estas políticas de forma centralizada. También traduce desde y hacia Prometheus (remote write) y StatsD.

## Instrumentos

Cada instrumento tiene nombre, tipo, unidad, descripción y, opcionalmente, pistas como los límites de bucket.

| Instrumento | Monótono | Agregación por defecto | Ejemplos |
|---|---|---|---|
| Counter / ObservableCounter | Sí | Sum | Peticiones, bytes enviados |
| UpDownCounter / ObservableUpDownCounter | No | Sum | Conexiones activas, tamaño de cola |
| Histogram | n/a | Histogram | Latencias, tamaños |
| Gauge / ObservableGauge | n/a | LastValue | CPU, memoria, temperatura |

**Síncronos**: los llamas tú cuando ocurre algo (`add()`, `record()`) y conocen el contexto activo, así que pueden generar exemplars. **Asíncronos** (Observable): registras un callback y el SDK lo ejecuta en cada recolección. Se usan para leer valores que ya existen, por ejemplo con polling al sistema operativo. En un ObservableCounter se reporta el **total**, no el incremento. "Asíncrono" aquí no tiene nada que ver con `asyncio`.

# 6. Registrar mediciones (Metrics API y SDK)

![[otca-diagrama-04.svg]]

- **MeterProvider**: el punto de entrada. Lleva el Resource, las Views, los readers y los exporters.
- **Meter**: un instrumentation scope (`name` obligatorio; `version` y `schema_url` opcionales). Crea los instrumentos.
- **View**: cambia cómo se agrega un instrumento sin tocar el código. Puede renombrarlo, cambiar la agregación o los buckets, quedarse solo con ciertos atributos (bajando cardinalidad) o descartarlo.
- **MetricReader**: decide cuándo se recoge y hace la agregación y la temporality. El exporter solo serializa y envía. Puedes tener varios readers, cada uno con su exporter.

| Modo | Quién inicia | Ejemplo | `ForceFlush` |
|---|---|---|---|
| Push | El reader, periódicamente (`PeriodicExportingMetricReader`) | OTLP al Collector | Recoge y exporta en el momento |
| Pull | Un scraper externo | Exporter de Prometheus con `/metrics` | **No tiene efecto** |

**Shutdown**: se llama una sola vez, propaga el cierre a readers y exporters e informa si ha ido bien, ha fallado o ha agotado el tiempo. Después, el provider devuelve meters no-op.

## Exemplars

Un exemplar es una medición cruda de muestra guardada junto a un data point agregado. Lleva el valor, el timestamp, los atributos filtrados (los que se descartaron al agregar) y el `trace_id`/`span_id` del span activo. Sirven para saltar de un pico en una gráfica a la traza concreta que lo causó (en Grafana aparecen como puntos sobre la gráfica).

- Solo salen de instrumentos **síncronos**, porque necesitan un span activo en el momento de medir. Valen tanto para counters como para histogramas.
- Filtro: `OTEL_METRICS_EXEMPLAR_FILTER` = `trace_based` (por defecto, solo mediciones dentro de spans muestreados), `always_on` o `always_off`.
- En Prometheus requieren `--enable-feature=exemplar-storage` y el formato OpenMetrics.

# 7. Logs

## Tipos de log

Hay logs **estructurados** (esquema fijo, normalmente JSON, legibles por máquina), **no estructurados** (texto libre, stack traces) y **semiestructurados** (clave=valor). Lo recomendable es usar estructurados, con nivel, origen y contexto (`trace_id`, `span_id`).

## Cómo encaja OTel con tu librería de logging

OTel no sustituye a Log4j, Logback, `logging`, Serilog, Winston o Zap: se engancha a ellos.

1. La app escribe logs con su librería de siempre.
2. Un **appender o bridge** de OTel captura cada log y consulta el contexto.
3. Si hay un span activo, le añade el `trace_id` y el `span_id`.
4. El log pasa por el `LoggerProvider`, el `LogRecordProcessor` y el `LogRecordExporter`, y sale por OTLP.

La alternativa sin appender es escribir a fichero o stdout y que el Collector lo recoja con el receiver `filelog` o `syslog`. En ese caso, para correlacionar con las trazas, el `trace_id` tiene que estar en el propio texto del log.

## Componentes

| Componente | Función |
|---|---|
| Log appender / bridge | Conecta la librería de logging existente con el SDK; la app no cambia su código |
| LoggerProvider | Fábrica de loggers, uno por aplicación; lleva el Resource y los exporters |
| Logger | Emite LogRecords para un instrumentation scope |
| LogRecordProcessor | Simple (síncrono, para depurar), Batching (producción) o Custom (filtrar, enriquecer, enrutar) |
| LogRecordExporter | `Export(batch)` y `Shutdown()`; batching, reintentos y backpressure |

## Logs API (Bridge API)

Pensada sobre todo para quien escribe appenders, no para escribir logs a mano desde la app. `LoggerProvider.getLogger(name)` (el nombre es obligatorio) → `logger.emit(LogRecord)`. Con `isEnabled` evitas construir logs caros cuando el nivel está desactivado. Las APIs son thread-safe.

## Logs SDK

Funciona como el de trazas: el provider lleva el Resource, y si cambias processors o exporters en el provider, los loggers que ya existen usan la nueva configuración. Al apagar la app: `ForceFlush()` y después `Shutdown()`. El exporter debe ser thread-safe, respetar los timeouts y devolver un resultado claro para que el processor decida si reintenta.

## Data model de un LogRecord

| Campo | Significado |
|---|---|
| `timestamp` | Cuándo ocurrió en origen |
| `observed_timestamp` | Cuándo lo recogió OTel (Collector o appender) |
| `severity_text` / `severity_number` | Nivel en texto y en número |
| `body` | El mensaje o la carga |
| `trace_id`, `span_id`, `trace_flags` | Correlación con la traza (opcionales) |
| `attributes`, `resource`, scope | Contexto del log, quién lo emite y qué librería lo generó |

| TRACE | DEBUG | INFO | WARN | ERROR | FATAL |
|---|---|---|---|---|---|
| 1–4 | 5–8 | 9–12 | 13–16 | 17–20 | 21–24 |

Cada nivel tiene 4 valores para poder mapear sistemas con grados intermedios. Si el destino no tiene un nivel equivalente, se usa el más cercano. Los timestamps son enteros de 64 bits en nanosegundos desde epoch. Se recomiendan logs estructurados de tipo **evento**; la spec actual añade un campo `event_name` para ellos.

## Buenas prácticas

- Logs estructurados con un esquema consistente; incluye siempre nivel, servicio y host.
- Añade el `trace_id`/`span_id` para poder correlacionar.
- Usa nombres de atributos consistentes (`order.id`, `user.id`) en lugar de concatenarlos en el mensaje.
- No registres PII ni credenciales; filtra o redacta en origen o en el Collector.
- Centraliza en el Collector el parseo, el enriquecimiento y la exportación.

En Python: añade `LoggingHandler(logger_provider=provider)` al logger raíz, o con zero-code usa `OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true`. El paquete `opentelemetry-instrumentation-logging` solo inyecta los ids en el formato del texto; no exporta.

# 8. Fundamentos del Collector

El Collector es un proceso independiente, neutral respecto a vendors, que **recibe, procesa y exporta** telemetría. Ventajas: desacopla las apps de los backends, centraliza la configuración (un YAML), escala horizontalmente y permite aplicar gobierno de datos (filtrar, redactar PII o enriquecer antes de que los datos salgan de tu red).

![[otca-diagrama-05.svg]]

## Distribuciones

|  | Core | Contrib |
|---|---|---|
| Contenido | Mínimo y estable: `otlp`, `batch`, `memory_limiter`, `debug`, extensiones básicas | Cientos de integraciones de la comunidad y de vendors |
| Ritmo de cambios | Lento, conservador | Rápido, más experimental |
| Cuándo | Producción crítica o entornos auditados | Probar integraciones, prototipos |

**OCB (OpenTelemetry Collector Builder)**: con un manifiesto YAML que lista los componentes que quieres, generas un binario o una imagen a medida. Así la imagen es más pequeña, hay menos superficie de ataque y las revisiones de seguridad son más fáciles. Las distribuciones de vendors añaden soporte y SLAs. Las **feature gates** sirven para activar comportamientos experimentales de forma controlada.

## Anatomía y config.yaml

| Sección | Función | Ejemplos |
|---|---|---|
| `receivers` | Ingestar datos y traducirlos al modelo interno | `otlp`, `prometheus`, `filelog` |
| `processors` | Transformar, filtrar, muestrear, agrupar (opcional, el orden importa) | `batch`, `attributes`, `tail_sampling` |
| `exporters` | Enviar a backends | `otlp`, `prometheusremotewrite`, `debug` |
| `connectors` | Exporter de un pipeline y receiver de otro | `spanmetrics`, `count` |
| `extensions` | Funciones fuera de los pipelines | `health_check`, `zpages`, `pprof` |
| `service` | Activa las extensiones y conecta los pipelines (**obligatorio**) | `service.pipelines.traces` |

- La configuración mínima válida es un receiver y un exporter conectados en un pipeline. **Lo que no aparece en `service` no se activa.**
- Puedes tener varias instancias del mismo tipo con `tipo/nombre` (`otlp/ingest2`) y varios pipelines por señal (`traces`, `traces/2`).
- **Fan-out** es un pipeline con varios exporters. **Fan-in** son varios receivers que alimentan un mismo pipeline.
- La configuración se puede parametrizar con `${env:VAR}` y `${file:ruta}`, y repartir en varios ficheros.
- Se valida antes de arrancar con `otelcol validate --config=config.yaml`.
- La telemetría interna del Collector se configura en `service.telemetry`: logs y métricas, con `metrics.readers` en modo pull (Prometheus) o periodic (push OTLP).

```yaml
receivers:
  otlp:
    protocols: { grpc: {}, http: {} }
processors:
  memory_limiter: { check_interval: 1s, limit_percentage: 80 }
  batch: {}
exporters:
  otlp: { endpoint: backend:4317 }
  debug: { verbosity: basic }
extensions:
  health_check: {}
service:
  extensions: [health_check]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp, debug]
```

# 9. Componentes del Collector

## El Collector como agente

| Despliegue | Uso |
|---|---|
| Agente en VM | Uno por host (por ejemplo, un servicio systemd); añade los metadatos del host |
| DaemonSet | Uno por nodo de Kubernetes; recoge de todos los pods del nodo, kubelet y host |
| Sidecar | Uno por pod; la app envía a `localhost:4317`. Máximo aislamiento, sin saltos de red |

Arquitectura en capas: los agentes locales enriquecen con metadatos del host o del pod, agrupan y reintentan. Después pueden ir Collectors por equipo. Al final, un gateway central se encarga de TLS, routing entre equipos, redacción de PII y la exportación final. Enriquece lo más cerca posible del origen y centraliza las políticas sensibles.

## Receivers importantes

| Receiver | Para qué | Notas |
|---|---|---|
| `otlp` | Entrada principal de apps instrumentadas | 4317 gRPC, 4318 HTTP; las tres señales |
| `prometheus` | Hacer scrape de endpoints Prometheus y convertir a OTLP | Config y service discovery de Prometheus |
| `hostmetrics` | CPU, memoria, disco y red del host | Como agente o DaemonSet |
| `kubeletstats` | Métricas de pods y contenedores desde el kubelet | DaemonSet (cada uno, su kubelet) |
| `k8s_cluster` | Métricas del clúster y estado de los nodos | Deployment de **1 réplica** (evita duplicados) |
| `k8sobjects` | Eventos y objetos de Kubernetes como logs | Deployment de 1 réplica |
| `filelog` | Leer ficheros de log; soporta rotación y guarda la posición | Logs de contenedores en el nodo |
| `journald`, `syslog` | Logs del sistema y de dispositivos de red | syslog en el 514 |
| `jaeger`, `zipkin`, `opencensus` | Migrar desde formatos antiguos | Zipkin en el 9411 |

## Processors

| Processor | Para qué |
|---|---|
| `memory_limiter` | Evitar OOM rechazando datos al pasar el límite. **Siempre en producción, el primero** |
| `batch` | Agrupar antes de exportar (al final de la cadena) |
| `resourcedetection` | Detectar metadatos de host, cloud y contenedor |
| `resource` | Añadir o cambiar atributos del Resource |
| `k8sattributes` | Añadir pod, namespace, deployment, labels… |
| `attributes` | insert/update/upsert/delete/hash de atributos (por ejemplo, PII) |
| `transform` | Reglas OTTL condicionales |
| `filter` | Descartar telemetría con condiciones OTTL |
| `probabilistic_sampler` | Muestreo por porcentaje, determinista (por hash del trace_id) |
| `tail_sampling` | Decidir con la traza completa (errores, latencia) |
| `deltatocumulative` / `cumulativetodelta` | Cambiar la temporality de las métricas |
| `span` | Renombrar spans o fijar el status |

Orden recomendado en el curso: `memory_limiter` → normalizar y enriquecer (`attributes`, `transform`, `resourcedetection`, `k8sattributes`) → `batch`.

## Exporters y resiliencia

El exporter `otlp` (gRPC) y el `otlphttp` son la opción por defecto: portables y estándar. Opciones clave:

| Opción | Para qué |
|---|---|
| `endpoint`, `tls` (`insecure` solo para pruebas) | Destino y cifrado o mTLS |
| `headers` o una extensión de autenticación | Autenticación (los secretos, en variables de entorno o en un gestor de secretos) |
| `timeout` | Tiempo máximo de cada intento |
| `retry_on_failure` | Reintentos con backoff exponencial; `max_elapsed_time` limita la duración total |
| `sending_queue` | Buffer mientras el backend no responde; con la extensión `file_storage` se guarda en disco y sobrevive a reinicios |
| `compression: gzip` | Ahorrar ancho de banda |

Exporters de contrib: `prometheus` (expone un endpoint para scrape), `prometheusremotewrite` (push a Cortex, Thanos o Mimir), `zipkin`, `kafka` (streaming duradero), `file` (depuración) y los de cada vendor. `debug` imprime los datos con una verbosity de basic, normal o detailed.

### Load balancing exporter

Reparte la telemetría entre varios Collectors de destino, pero de forma "pegajosa": todo lo que tiene la misma clave de routing va siempre al mismo destino. Es imprescindible para el **tail sampling a escala**, porque todos los spans de una traza tienen que acabar en el mismo Collector. Crea un sub-exporter OTLP por destino, cada uno con su propia cola y sus reintentos.

- Claves de routing: `traceID` (por defecto en trazas), `service` (por defecto en métricas), `resource`, `metric` y `streamID` (solo métricas), `attributes`.
- Resolvers: `static` (lista fija), `dns` y `k8s` (servicio headless). Los destinos deberían tener una identidad estable, por ejemplo con un StatefulSet.

### Patrones

- **Fan-out**: los mismos datos a varios destinos; duplica pronto para que todos reciban lo mismo.
- **Routing por atributo**: por entorno, por servicio, etc.
- **Forwarding entre pipelines** o entre Collectors.
- Usa siempre `batch` junto con cola y reintentos, valida los cambios con un pipeline canario y el exporter `debug`, y vigila los errores en las métricas internas.

## Connectors

Hacen de exporter en un pipeline y de receiver en otro. Sirven para generar telemetría nueva o enrutar sin tocar las apps.

| Connector | Qué hace |
|---|---|
| `spanmetrics` | Métricas RED (rate, errors, duration) a partir de spans, con histogramas y exemplars |
| `servicegraph` | Métricas de dependencias entre servicios (mapa de topología) |
| `routing` | Enruta a pipelines con condiciones OTTL (`default_pipelines`, `match_once`) |
| `count` | Cuenta spans, eventos o logs que cumplen una condición |
| `sum` | Suma un atributo numérico (por ejemplo, el total de los pedidos) |
| `forward` | Une o divide pipelines sin transformar |
| `exceptions` | Extrae las excepciones de las trazas como logs o métricas |

El connector se declara en `connectors:` y se usa como exporter de un pipeline (por ejemplo `traces`) y como receiver de otro (por ejemplo `metrics`).

# 10. Transformar telemetría en el pipeline

## OTTL (OpenTelemetry Transformation Language)

Un lenguaje de sentencias que el Collector evalúa sobre cada elemento de telemetría. Funciona con trazas, métricas y logs. Lo usan el `transform` processor, el `filter` processor, el connector `routing` y algunas políticas de `tail_sampling`.

- Sintaxis: `funcion(argumentos) where condicion`. Si la condición del `where` es falsa, la sentencia se salta para ese elemento.
- Contextos (sobre qué actúa cada sentencia): `resource`, `scope`, `span`, `spanevent`, `metric`, `datapoint` y `log`.
- Funciones típicas: `set`, `delete_key`, `keep_keys`, `replace_pattern`, `SHA256` (hashear PII), `truncate_all`, `ExtractPatterns`, `Concat`.
- Usos: enmascarar PII, renombrar o normalizar atributos, extraer campos (por ejemplo, el user id de una URL), descartar ruido.

| `error_mode` | Comportamiento |
|---|---|
| `ignore` | Registra el error y sigue. **Lo recomendado en producción** |
| `silent` | Sigue sin registrar nada (oculta problemas) |
| `propagate` | Devuelve el error y detiene el procesado de ese dato |

```yaml
processors:
  transform:
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          - set(attributes["env"], "prod") where resource.attributes["k8s.namespace.name"] == "prod"
          - set(attributes["user.email"], SHA256(attributes["user.email"]))
  filter:
    error_mode: ignore
    traces:
      span:
        - resource.attributes["service.name"] == "healthcheck"
```

> [!note]
> En el `filter` processor, si la condición se cumple, el elemento se **descarta**.

## Estrategias de sampling

![[otca-diagrama-06.svg]]

*Tail sampling a escala: todos los spans de una traza deben llegar al mismo Collector.*

|  | Head sampling | Tail sampling |
|---|---|---|
| Dónde | SDK (sampler) o Collector (`probabilistic_sampler`) | Collector (`tail_sampling`) |
| Cuándo decide | Al principio, sin conocer el resultado de la traza | Con la traza completa, tras esperar `decision_wait` |
| Ventajas | Barato, rápido, escala bien, reduce volumen pronto | Puede quedarse con todas las trazas con error o lentas |
| Inconvenientes | Puede descartar justo las trazas importantes | Memoria (buffer), latencia, y todos los spans de una traza deben ir al mismo Collector |

**`probabilistic_sampler`** hashea el trace_id con una semilla (`hash_seed`) y lo compara con `sampling_percentage`. Es determinista: con la misma semilla en todos los Collectors, todos toman la misma decisión para una traza.

**Políticas de `tail_sampling`**: `status_code` (ERROR), `latency` (`threshold_ms`), `span_count` (`min_spans`), `string_attribute` (por ejemplo, `tenant=vip`), `rate_limiting` (spans por segundo), `probabilistic` (un porcentaje del resto), `ottl_condition` y combinaciones con `and` o `composite`. Si alguna política dice "quedarse", la traza se conserva. Los spans que llegan después de que la traza ya se ha decidido se pierden.

Arquitectura a escala: una primera capa de Collectors sin estado con el exporter `load_balancing` (routing por traceID), y una segunda capa con identidad estable (StatefulSet) que hace el tail sampling. El patrón híbrido habitual es un head sampling bajo para reducir el volumen de base más un tail sampling para quedarse con errores y trazas lentas.

## Telemetry schemas

Las semantic conventions evolucionan (por ejemplo, `http.status_code` pasó a `http.response.status_code`). Un **schema** es un fichero versionado que registra esos cambios, y cada dato lleva un `schema_url` (`https://opentelemetry.io/schemas/1.2.0`) que dice qué versión usa. Así un Collector o un backend puede traducir entre versiones y los dashboards no se rompen.

- Versionado semántico: un cambio de **patch** no cambia nada para el consumidor; un **minor** añade cosas compatibles hacia atrás; un **major** puede romper.
- Compatibilidad: un consumidor `1.3` puede leer datos `1.1`, pero un consumidor `1.1` no puede leer datos `1.3`. Con un major distinto no hay compatibilidad.
- Secciones del fichero: `all` (globales; solo admite `rename_attributes` y **va primero**), `resources`, `spans`, `span_events`, `metrics` (`rename_metrics`, `rename_attributes` con `apply_to_metrics`, `split`) y `logs`.
- Los selectores `apply_to_spans`, `apply_to_events` y `apply_to_metrics` limitan el alcance de un cambio. Si se omiten, se aplica a todo.
- Para migrar se pasa por cada versión intermedia en orden (1.0 → 1.1 → 1.2), sin saltarse ninguna. El `schema_url` indica la versión más alta.
- Mantén las transformaciones mínimas y basadas en renombrar.

# 11. OpenTelemetry en Kubernetes

En Kubernetes la telemetría tiene que llevar el contexto del pod: nombre y UID, namespace, labels y anotaciones, el workload (Deployment, ReplicaSet, StatefulSet) y el nodo. Ese enriquecimiento lo hacen el processor `k8sattributes` (que necesita RBAC para leer la API) o los resource detectors.

## Modos de despliegue del Collector

| Modo | Ideal para | Características |
|---|---|---|
| DaemonSet (agente) | Telemetría del nodo: kubelet, host, logs de los contenedores | Uno por nodo, buffer local, metadatos del nodo; reenvía al gateway o directo al backend |
| Deployment (gateway) | Procesado central: sampling, redacción, agregación | Sin estado, varias réplicas detrás de un Service |
| Sidecar | Aislamiento por aplicación, sin saltos de red | En el mismo pod que la app; configuración propia |
| StatefulSet | Scraping de Prometheus repartido, tail sampling | Identidad y DNS estables, PVC para persistir la cola |

El **TargetAllocator** (funcionalidad del Operator) reparte los targets de scrape de Prometheus entre varios Collectors para que no se scrapeen dos veces.

## Receivers clave en Kubernetes

| Componente | Qué aporta | Dónde |
|---|---|---|
| `k8sattributes` (processor) | Metadatos de pod y nodo | Cualquier Collector, con RBAC |
| `kubeletstats` | CPU y memoria de nodo, pod y contenedor desde el kubelet (puerto 10250) | DaemonSet |
| `filelog` | Logs de `/var/log/containers/*.log` (o `/var/log/pods`) | DaemonSet con hostPath |
| `k8s_cluster`, `k8sobjects` | Métricas del clúster, estado de objetos y eventos | Deployment de 1 réplica |
| `hostmetrics` | Métricas del sistema operativo del nodo | DaemonSet |
| `prometheus` | Scrape de endpoints `/metrics` | Según la topología; cuidado con los duplicados |
| `otlp` | Telemetría de las apps | Gateway o agente |

## OpenTelemetry Operator

Un operador de Kubernetes que gestiona los Collectors y la autoinstrumentación mediante CRDs. Se instala con Helm (lo recomendado) o con manifiestos, y necesita **cert-manager** (o algo equivalente) para los certificados de sus webhooks. Crea los Services y puertos, gestiona los endpoints, hace los rollouts y las actualizaciones, e inyecta la instrumentación.

- **`OpenTelemetryCollector`** (CR): declara un Collector con su `mode` (`deployment`, `daemonset`, `sidecar` o `statefulset`) y su configuración.
- **`Instrumentation`** (CR): la configuración común de la autoinstrumentación. Cada campo se traduce en una variable de entorno: `spec.exporter.endpoint` → `OTEL_EXPORTER_OTLP_ENDPOINT`, `spec.propagators` → `OTEL_PROPAGATORS`, `spec.sampler` → `OTEL_TRACES_SAMPLER` y su `_ARG`.

Para activarla en un pod o en un namespace se usa una anotación, por ejemplo `instrumentation.opentelemetry.io/inject-java` (o `-python`, `-nodejs`, `-dotnet`, `-go`):

| Valor | Significado |
|---|---|
| `"true"` | Usa la Instrumentation por defecto del mismo namespace |
| `"nombre"` | Usa una Instrumentation concreta del mismo namespace |
| `"ns/nombre"` | Usa una Instrumentation de otro namespace |
| `"false"` | Excluye ese workload |

Con `sidecar.opentelemetry.io/inject: "true"` se inyecta un Collector sidecar. Cuidado con anotar namespaces enteros: puedes inyectar agentes en pods donde no querías.

# 12. Depuración, operación y escalado del Collector

## Herramientas de diagnóstico

| Herramienta | Puerto por defecto | Para qué |
|---|---|---|
| Métricas internas | **8888** | Métricas Prometheus del propio Collector: throughput, rechazos, colas, CPU y memoria |
| `health_check` | **13133** | Latido para las probes de liveness y readiness de Kubernetes y para balanceadores |
| `zpages` | **55679** | Páginas en el navegador: `/debug/servicez`, `pipelinez`, `extensionz`, `featurez`, `tracez` |
| `pprof` | **1777** | Profiling de Go: CPU, heap/allocs, goroutines, mutex, block |

## Logs internos y el exporter debug

- Nivel de los logs del propio Collector: `service.telemetry.logs.level` (DEBUG, INFO por defecto, WARN o ERROR), además de `encoding: json`, `sampling` y `output_paths`. Por defecto los escribe en stderr. Se ven con `journalctl -u otelcol-contrib`, `docker logs` o `kubectl logs`.
- El exporter `debug` imprime en el log los datos que pasan, con una `verbosity` de `basic` (resúmenes y conteos), `normal` (añade resource y scope) o `detailed` (todo el contenido). Admite `sampling_initial` y `sampling_thereafter`. Sirve para comprobar el formato de los datos, la propagación de trace_id y lo que hacen los processors.
- No confundas el nivel de log del Collector con la verbosity del exporter debug: son cosas distintas.

## Métricas internas que hay que conocer

Patrón de nombres: `otelcol_<componente>_<acción>_<señal>` (con `_total` en los contadores).

| Métrica | Qué indica |
|---|---|
| `otelcol_receiver_accepted_spans` | Lo que entra |
| `otelcol_receiver_refused_spans` | Lo que se rechaza al entrar (límites, datos mal formados) |
| `otelcol_processor_refused_spans` | Lo que rechaza un processor, típicamente `memory_limiter` bajo presión |
| `otelcol_exporter_sent_spans` / `send_failed_spans` | Envíos correctos y fallidos (problemas de red o del backend) |
| `otelcol_exporter_queue_size` / `queue_capacity` | Cola de envío; si crece, hay backpressure |
| `otelcol_exporter_enqueue_failed_spans` | Cola llena: se pierden datos |

## Escalado

| Tipo de componente | Cómo escalar |
|---|---|
| Sin estado (recibir OTLP y reenviar) | Horizontalmente, varias réplicas detrás de un balanceador |
| Scrapers (Prometheus) | Repartir los targets entre Collectors con el TargetAllocator (vía `http_sd_config`) para no duplicar métricas |
| Con estado (tail sampling, spanmetrics) | Routing consistente por trace_id (exporter `load_balancing`) hacia réplicas estables |

| Síntoma | Acción |
|---|---|
| Rechazos en un processor o presión de memoria | Más CPU y memoria o más réplicas; ajustar `memory_limiter` |
| La cola del exporter crece (alrededor del 70 %) | Más capacidad de cola, ajustar timeouts y reintentos, o escalar |
| La latencia del exporter es alta | El cuello de botella es el backend o la red: **añadir réplicas del Collector no lo arregla** |

Primero identifica qué señal está sobrecargada. Si tienes pipelines separados por señal, escala solo el que lo necesita.

# 13. SLIs y SLOs con OpenTelemetry

- **SLI**: un indicador medido desde el punto de vista del usuario. Los tipos habituales son disponibilidad, latencia, errores, throughput y saturación.
- **SLO**: el objetivo sobre un SLI en una ventana de tiempo, por ejemplo que el 99 % de las peticiones tarden menos de 200 ms en 30 días. El **SLA** es el compromiso contractual.
- Fórmulas típicas: tasa de éxito = peticiones correctas / total; cumplimiento de latencia = peticiones por debajo del umbral / total; tasa de error = errores / total.
- El proceso: define cómo es una buena experiencia, elige el SLI que la mide, fija el SLO y alerta sobre las tendencias antes de incumplir el SLA.
- El CPU sirve para diagnosticar, pero no es un buen SLI, porque no refleja directamente lo que percibe el usuario.

| Pieza | Responsabilidad |
|---|---|
| App / SDK | Generar telemetría precisa (histogramas de latencia, status de los spans) |
| Collector | Opcional: transformar, preagregar, derivar métricas de spans (`spanmetrics`) o de logs (`count`) |
| Backend | **Evalúa el SLO**: ventanas, error budget, dashboards y alertas. OTel no lo hace |

> [!note]
> Ojo con el sampling: si calculas SLIs a partir de trazas muestreadas, los números pueden salir sesgados. Para SLIs, mejor métricas (que no se muestrean) o spanmetrics generadas antes del sampling.

# Chuleta final para el examen

| Tema | Lo que hay que recordar |
|---|---|
| Puertos | OTLP gRPC 4317 · OTLP HTTP 4318 · métricas internas 8888 · health_check 13133 · zpages 55679 · pprof 1777 · Zipkin 9411 |
| IDs | trace_id 16 bytes (32 hex) · span_id 8 bytes (16 hex) · flags 01 = sampled |
| Span kind | Por defecto INTERNAL. CLIENT/SERVER son síncronos; PRODUCER/CONSUMER, asíncronos |
| Status | UNSET por defecto · ERROR en fallos · OK solo explícito, puesto por quien desarrolla la app |
| Eventos vs atributos | Un evento tiene un timestamp y no tiene duración; un atributo describe todo el span |
| Resource por defecto | `service.name=unknown_service` + `telemetry.sdk.*` |
| Samplers | Solo ParentBased acepta otro sampler dentro. Head = SDK o probabilistic; tail = Collector |
| API sin SDK | No-op, sin errores |
| Librerías | Dependen solo de la API; nunca configuran exporters |
| Instrumentos | Counter (monótono), UpDownCounter, Histogram, Gauge; los Observable usan callback y reportan el total |
| Exemplars | Solo en instrumentos síncronos; filtro por defecto trace_based |
| Severity | TRACE 1–4 · DEBUG 5–8 · INFO 9–12 · WARN 13–16 · ERROR 17–20 · FATAL 21–24 |
| Logs | `timestamp` = cuándo ocurrió; `observed_timestamp` = cuándo lo vio OTel |
| Collector | Lo que no está en `service.pipelines` no se activa · `memory_limiter` primero, `batch` al final |
| Kubernetes | kubeletstats/hostmetrics/filelog en DaemonSet · k8s_cluster/k8sobjects en 1 réplica · tail sampling en StatefulSet |
| Schemas | `all` va primero y solo admite renames · migrar versión a versión |

