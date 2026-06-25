# Entregable I: Propuesta de Mejora Arquitectónica del Sistema Legacy JWQL

**Autores:** Margiory Alvarado Chavez, Aaron Coorahua Peña. UTEC, Lima, Perú
**Curso:** Arquitectura de Software, Ciclo 2026-I
**Repositorio del sistema:** https://github.com/spacetelescope/jwql
**Repositorio de la propuesta:** https://github.com/AaronCoorahua/jwql-arch-evolution

> Versión markdown del paper en `presentation/main.tex` (formato IEEE). Ambos documentos deben mantenerse consistentes.

---

## Resumen

JWQL es la aplicación web con que el STScI monitorea la salud de los instrumentos del telescopio James Webb. Su subsistema de procesamiento es un cuello de botella: la calibración de imágenes, la tarea más costosa, se ejecuta sin paralelismo real y comparte infraestructura con el *dashboard*, de modo que un lote pesado ralentiza el procesamiento y degrada la lectura a la vez. El dato de entrada es además una rampa de cuatro dimensiones que vuelve la calibración intensiva en memoria, por lo que el escalado lo limita la RAM por nodo, no la CPU. Se propone una modernización evolutiva (extraer el servicio de calibración y separar el camino de escritura del de lectura) y un plan de validación que mide, a recursos fijos, la mejora en latencia y confiabilidad del *dashboard* y el techo de *throughput* según el tamaño de imagen.

---

## 1. Introducción

JWQL es una aplicación web y *framework* de automatización que los equipos de instrumentos del STScI usan para vigilar los cinco instrumentos del telescopio James Webb. Obtiene datos crudos del archivo MAST, los calibra con el *pipeline* científico de JWST y ejecuta monitores (corriente de oscuridad, píxeles defectuosos, rayos cósmicos, ruido de lectura y telemetría) cuyos resultados publica en un *dashboard*. Está en producción desde 2017 y su código es público. Este trabajo analiza su arquitectura, define el problema y propone una mejora validable; el diseño técnico y las métricas finales corresponden al siguiente entregable.

---

## 2. Sistema Actual y Problema

JWQL integra un sistema de archivos de red para los productos calibrados y sin calibrar, una base relacional con los metadatos observacionales, una biblioteca de monitoreo y la aplicación web de inspección. Son unos 160 archivos Python y 52 mil líneas de código sobre Django, con una cola de tareas asíncrona en Celery/Redis, PostgreSQL como base de datos y Bokeh para la visualización.

El estilo es un monolito en capas con procesamiento asíncrono parcial: un disparador lanza un monitor, este detecta archivos nuevos, los calibra, guarda resultados y estado en la base, y la web consulta esos datos para armar el *dashboard*.

### 2.1 Problema

El subsistema de procesamiento concentra el problema, por tres causas que se agravan entre sí:

- **(P1) Sin paralelismo real.** La cola Celery corre un proceso por worker (`worker_concurrency=1`) y reinicia el worker tras cada tarea (`max_tasks_per_child=1`), y la orquestación bloquea hasta terminar cada calibración. La tarea más costosa se ejecuta, en la práctica, en serie.
- **(P2) Acotado por la memoria.** El `uncal` es una rampa 4D (columnas × filas × grupos × integraciones) de hasta ~2 GB; el ajuste de rampa la carga completa y mantiene varias copias en RAM. Los comentarios de configuración de JWQL fijan `worker_concurrency=1` porque un proceso de *pipeline* *"puede consumir toda la memoria disponible"*, y el código trata el error `Cannot allocate memory` como un fallo conocido. El escalado lo limita la RAM por nodo; la huella exacta se mide en el POC sobre una exposición pública de MAST.
- **(P3) Lectura y escritura acopladas.** Los monitores comparten infraestructura y modelos con la capa web; un lote pesado degrada directamente la latencia del *dashboard*.

A esto se suman problemas de mantenibilidad: módulos con demasiadas responsabilidades (varios sobre 1 500 líneas, alguno sobre 3 000) y una persistencia que mezcla dos ORM sobre dos bases. En síntesis: cuando llega un lote, el procesamiento es lento *y* el *dashboard* se degrada, y no se puede escalar porque no hay paralelismo, la memoria pone techo y el monolito acopla ambos caminos.

### 2.2 Tamaño de los productos del *pipeline*

| Producto / unidad | Tamaño aprox. |
|---|---|
| *Frame* de lectura (2048×2048, uint16) | ~8 MB |
| Exposición `uncal` típica (4D) | 40-90 MB |
| `uncal` de series temporales (segmentada) | hasta ~2 GB |
| Producto calibrado `rate` (2D, float32) | ~17 MB |

---

## 3. Requerimientos

El trabajo se centra en el procesamiento y la visualización de monitores. Restricciones: la evolución debe ser incremental porque el sistema está en producción; el escalado debe ser horizontal por el alto consumo de memoria del *pipeline*; y la prueba de concepto debe correrse localmente.

**Funcionales (a preservar)**

| ID | Requerimiento |
|---|---|
| RF-1 | Detectar archivos nuevos y disparar el monitoreo. |
| RF-2 | Calibrar cada archivo con el *pipeline* de JWST. |
| RF-3 | Calcular las métricas de monitoreo y guardarlas. |
| RF-4 | Registrar el estado de cada corrida. |
| RF-5 | Publicar los resultados en el *dashboard*. |
| RF-6 | Evitar ejecuciones duplicadas o concurrentes. |

**No funcionales (a mejorar)**

| Atributo | Objetivo |
|---|---|
| Escalabilidad | Procesar varios archivos en paralelo sumando workers. |
| Rendimiento | Reducir el tiempo total de una corrida. |
| Mantenibilidad | Reducir el tamaño y el acoplamiento de los módulos. |
| Desplegabilidad | Escalar el procesamiento sin afectar la web. |
| Resiliencia | Recuperarse si falla un worker sin perder la corrida. |

---

## 4. Usuario Modelo

El usuario principal es el **científico de instrumento**: necesita detectar tendencias anómalas cuanto antes y que el *dashboard* refleje los datos nuevos sin demora; hoy, con un lote grande, el procesamiento tarda horas y la web se vuelve lenta. El usuario secundario es el **ingeniero que mantiene JWQL**: necesita modificar o agregar monitores y escalar el procesamiento sin afectar otros componentes.

---

## 5. Estado del Arte

Modernizar un sistema legacy en producción conviene hacerlo de forma evolutiva. El patrón **Strangler Fig** de Fowler construye el nuevo sistema alrededor del existente y lo reemplaza poco a poco, sin reescritura total; Newman aporta tácticas como **Branch by Abstraction** (aislar un componente tras una interfaz) y **Parallel Run** (correr ambas implementaciones y comparar). Para una tarea pesada detrás de una cola, Richardson propone **Competing Consumers**: varios workers compiten por la misma cola y el rendimiento crece con su número. El patrón **CQRS** separa el camino de escritura del de lectura para escalarlos por separado, lo que importa cuando hay contención de recursos entre ambos.

Li, Ma y Lu aplicaron esta combinación en el sistema Green Button: extrajeron con Strangler Fig y CQRS el servicio que era cuello de botella y midieron el impacto a recursos fijos. Su resultado es la clave metodológica que se adopta: el efecto más importante no fue sobre el servicio migrado, sino sobre el resto del sistema, ya que los módulos no migrados mejoraron su tiempo de respuesta al liberarse los recursos que consumía el componente pesado. La arquitectura original de JWQL está documentada por Bourque et al.

---

## 6. Propuesta

El foco son P1, P2 y P3 (el cuello de botella de procesamiento). La propuesta es una **modernización evolutiva en un solo corte**: extraer el servicio de calibración detrás de una interfaz, separar el camino de escritura del de lectura y mantener el camino actual operativo durante la transición. No es una migración completa a microservicios ni una reescritura.

El **benchmark prueba** el efecto de Competing Consumers (escalado) y CQRS (separación lectura/escritura); Strangler Fig y Branch by Abstraction son el *método* de extracción incremental; el Sidecar provee la observabilidad.

| Objetivo | Patrón |
|---|---|
| Paralelizar la calibración | Competing Consumers |
| Separar lectura de escritura | CQRS + réplicas de lectura |
| Aislar el *pipeline* | Branch by Abstraction |
| Migrar sin detener el sistema | Strangler Fig + Parallel Run |
| Observabilidad sin tocar el core | Sidecar |

### 6.1 Hipótesis

A recursos totales fijos: (i) separar escritura de lectura reduce la latencia y la tasa de error del *dashboard* mientras corre un lote (efecto de mayor impacto que solo añadir workers), y (ii) el *throughput* de procesamiento se satura por memoria antes que por CPU, con un techo que depende del tamaño de imagen.

### 6.2 Experimento

El cuello de botella es la calibración (P1, P2). Se mide su rendimiento antes y después de extraerla, a igualdad de recursos (CPU y RAM fijos en contenedores Docker), con el diseño de Li, Ma y Lu: se eligen dos componentes bajo prueba (AUT) y se compara cada uno antes y después de la mejora.

- **AUT-1, procesamiento (escritura):** la calibración `calwebb_detector1` que dispara el dark monitor vía `shared_tasks.run_pipeline`. Es el componente que se extrae.
- **AUT-2, dashboard (lectura):** la vista `dark_monitor(inst)` de `monitor_views.py`, que arma las pestañas Bokeh leyendo de la base `monitors` (endpoint `GET /<inst>/dark_monitor/`). Es el componente no migrado que comparte recursos con el procesamiento.

La carga es de 200 hilos por 5 repeticiones por AUT, como en el caso Green Button, y se repite con dos tamaños de exposición (`rate` pequeño y `uncal` grande) para observar el techo de memoria. Se ejecuta el stack real de JWQL (web, Celery, Redis y PostgreSQL) en contenedores; el *pipeline* se sustituye por una carga sintética calibrada contra una corrida real sobre datos públicos de MAST/CRDS, para fijar el perfil de CPU y memoria. CPU y RAM se recolectan a nivel de nodo (en producción, vía un *sidecar* por servicio).

### 6.3 Métricas

Por cada AUT, antes y después de extraer la calibración (carga: 200 hilos × 5):

| Métrica | Criterio de mejora |
|---|---|
| Tiempo de prueba (s) | menor o igual |
| Respuesta media (ms) | menor en la vista `dark_monitor` (lectura) |
| Error (%) | tiende a 0 |
| Throughput (req/s) | mayor |
| Tamaño medio de respuesta (B) | igual (control de carga) |
| Pico de RAM por tarea (GB) | caracteriza el techo de escalado |
| `Cannot allocate memory` (conteo) | tiende a 0 |

### 6.4 Resultado esperado

Como en Green Button, se espera que el componente no migrado, la vista `dark_monitor`, sea el que más mejore: su respuesta media y su tasa de error bajo carga deben caer al dejar de competir por los recursos de la calibración. En el procesamiento (`calwebb_detector1`) se espera mayor *throughput* al sumar workers y la desaparición de los errores de memoria. Las cifras se obtienen en el siguiente entregable.

---

## Referencias

1. M. Bourque et al., *The James Webb Space Telescope Quicklook Application (JWQL)*, ASP Conf. Ser., vol. 527, p. 539, 2020. Zenodo, doi: 10.5281/zenodo.3698708.
2. M. Fowler, *StranglerFigApplication*, martinfowler.com, 2004.
3. S. Newman, *Monolith to Microservices*, O'Reilly Media, 2019.
4. C. Richardson, *Microservices Patterns*, Manning Publications, 2018.
5. M. Fowler, *CQRS*, martinfowler.com.
6. C.-Y. Li, S.-P. Ma, T.-W. Lu, *Microservice Migration Using Strangler Fig Pattern: A Case Study on the Green Button System*, 2020 Int. Computer Symposium (ICS), pp. 519-524, IEEE.
