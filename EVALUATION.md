# Evaluación de Sistema Legacy: JWQL (JWST Quicklook Application)

**Curso:** Arquitectura de Software  
**Ciclo:** 2026-I  
**Fecha de evaluación:** 18 de junio de 2026  

---

## 1. Descripción del Sistema

**JWQL** (JWST Quicklook Application) es una aplicación web y framework de automatización desarrollado por el Space Telescope Science Institute (STScI) para el equipo de instrumentos del Telescopio Espacial James Webb (JWST). Su propósito es monitorear y registrar la salud, estabilidad y rendimiento de los instrumentos del telescopio en tiempo operacional.

El sistema gestiona cuatro responsabilidades principales:

1. Almacenamiento de productos de datos calibrados y no calibrados en un sistema de archivos de red centralizado (caché de datos MAST).
2. Base de datos relacional con metadatos observacionales accesibles vía API (MAST Database API).
3. Framework de software para rutinas automatizadas de monitoreo de instrumentos.
4. Aplicación web para inspección visual de datos JWST y resultados de monitoreo.

Repositorio oficial: [github.com/spacetelescope/jwql](https://github.com/spacetelescope/jwql)

---

## 2. Métricas de Escala del Codebase

| Métrica | Valor |
|---|---|
| Archivos Python | 162 |
| Líneas de código (LOC) | ~52,357 |
| Instrumento-monitores | 5 instrumentos (FGS, MIRI, NIRCam, NIRISS, NIRSpec) |
| Monitores comunes | 5 (bad pixel, bias, cosmic ray, dark, readnoise) |
| Historial de commits | Activo desde 2017 |

---

## 3. Arquitectura Actual (As-Is)

### 3.1 Estilo Arquitectónico

El sistema sigue una **arquitectura monolítica en capas** construida sobre el framework Django, con un intento parcial de procesamiento asíncrono mediante Celery + Redis. Los componentes principales son:

```
┌─────────────────────────────────────────────────────┐
│                 Django Web Application               │
│         (jwql/website/apps/jwql/)                   │
│  views.py · data_containers.py · bokeh_dashboard.py │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│              Capa de Procesamiento                   │
│         (jwql/instrument_monitors/)                  │
│    dark · bias · bad_pixel · cosmic_ray · readnoise  │
│          + monitores por instrumento                 │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│           Celery Workers + Redis Broker              │
│           (jwql/shared_tasks/)                       │
│         run_pipeline.py · shared_tasks.py            │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│         Capa de Datos                               │
│   PostgreSQL (via psycopg2 / SQLAlchemy)            │
│   MAST API · CRDS · EDB (Engineering Database)     │
└─────────────────────────────────────────────────────┘
```

### 3.2 Stack Tecnológico

| Capa | Tecnología |
|---|---|
| Web framework | Django 5.x |
| Servidor WSGI | Gunicorn |
| Base de datos | PostgreSQL (psycopg2, SQLAlchemy) |
| Task queue | Celery 5.x |
| Message broker | Redis |
| Visualización | Bokeh |
| Procesamiento científico | NumPy, SciPy, Astropy, JWST pipeline |
| Autenticación externa | MAST, CRDS |

---

## 4. Identificación de Problemas Arquitectónicos

### 4.1 God Files (Alta deuda técnica)

El análisis del codebase revela archivos con responsabilidades excesivas:

| Archivo | Líneas | Problema |
|---|---|---|
| `jwql/utils/preview_image.py` | ~3,600 | Generación de imágenes, transformaciones, thumbnails — todo en un módulo |
| `jwql/instrument_monitors/common_monitors/edb_telemetry_monitor.py` | ~2,700 | Lógica de telemetría, consultas EDB, persistencia mezcladas |
| `jwql/website/apps/jwql/data_containers.py` | ~2,703 | Orquesta acceso a datos, transformaciones y preparación de contexto para vistas |
| `jwql/website/apps/jwql/views.py` | ~1,557 | Contiene lógica de negocio que debería estar en servicios separados |
| `jwql/instrument_monitors/common_monitors/dark_monitor.py` | ~1,900 | Monitor individual con lógica de pipeline completa embebida |

### 4.2 Acoplamiento Fuerte entre Monitores y Web Layer

Los monitores de instrumentos comparten estado e infraestructura directamente con la capa web (mismos modelos Django, mismo proceso), lo que impide escalar cada componente independientemente.

### 4.3 Celery como Solución Parcial

La introducción de Celery en `shared_tasks/` es un intento de desacoplar el procesamiento pesado, pero los tasks siguen siendo monolíticos — un solo worker ejecuta el pipeline completo de cada instrumento sin posibilidad de paralelismo granular.

### 4.4 Ausencia de Contratos de API Internos

No existe una capa de servicios con interfaces definidas. Las dependencias entre módulos son directas (imports), lo que hace difícil reemplazar componentes sin afectar el resto del sistema.

---

## 5. Justificación como Caso de Estudio

El sistema `jwql` constituye un candidato sólido para el proyecto de arquitectura por las siguientes razones:

### 5.1 Es un Sistema Legacy Real en Producción

Desarrollado y mantenido desde 2017 por el STScI (institución que opera el JWST para NASA), con un historial de evolución orgánica que generó deuda técnica visible y medible.

### 5.2 Los Problemas son Concretos y Cuantificables

Los anti-patrones identificados (God Files, acoplamiento fuerte, workers monolíticos) tienen métricas directas: LOC por módulo, acoplamiento aferente/eferente, tiempo de respuesta de procesamiento, throughput de tareas Celery.

### 5.3 Existe un Camino de Mejora Arquitectónica Claro

La combinación Django + Celery + Redis ya establece una base sobre la cual aplicar patrones bien documentados en la literatura:

- **Strangler Fig Pattern**: descomposición incremental del monolito
- **Event-Driven Architecture**: uso de eventos para desacoplar monitores
- **CQRS**: separar lecturas (dashboard) de escrituras (pipeline de monitoreo)
- **Microservicios por instrumento**: cada instrumento como unidad desplegable independiente

### 5.4 Benchmarking Factible

El dominio permite definir métricas concretas para comparar arquitectura actual vs. propuesta:

- Throughput de procesamiento (imágenes/minuto)
- Latencia de respuesta del dashboard bajo carga
- Tiempo de ejecución de monitores por instrumento
- Escalado horizontal de workers (Celery)
- Tiempo de recuperación ante fallos (resiliencia)

### 5.5 Documentación del Estado del Arte Disponible

Existe literatura académica directamente aplicable: migración de monolitos a microservicios, Strangler Fig en sistemas científicos, arquitecturas event-driven para pipelines de datos — facilitando la construcción del estado del arte del Entregable I.

---

## 6. Alcance Propuesto para el Proyecto

Dado que el sistema completo requiere infraestructura NASA (datos MAST, CRDS, red interna del STScI), el proyecto se enfocará en un **subsistema acotado y reproducible**:

**Subsistema objetivo**: Pipeline de monitoreo de instrumentos (dark monitor / bad pixel monitor) + sistema de tareas Celery + capa de visualización de resultados.

Este subsistema es:
- Ejecutable localmente con datos de muestra
- Lo suficientemente representativo de los problemas arquitectónicos del sistema completo
- Acotado para caber en un POC de 3 semanas con métricas demostrables

---

## 7. Referencias

- Bourque, M. et al. (2020). *JWQL: The James Webb Space Telescope Quicklook Application*. Zenodo. https://doi.org/10.5281/zenodo.3698708
- Newman, S. (2019). *Monolith to Microservices*. O'Reilly Media.
- Fowler, M. (2004). *Strangler Fig Application*. martinfowler.com.
- Richardson, C. (2018). *Microservices Patterns*. Manning Publications.
- Taibi, D., & Lenarduzzi, V. (2021). *Microservice Migration Using Strangler Fig Pattern: A Case Study on the Green Button System*. ResearchGate.
