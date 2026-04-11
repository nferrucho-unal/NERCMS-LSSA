# team-2e — Sistema 1: Data Acquisition and Edge Computing

Modelo arquitectónico del **Sistema 1** del SoS NERCMS — captura de datos
desde sensores heterogéneos (hidrológicos, sísmicos, atmosféricos, cámaras),
procesamiento en nodos de borde regionales, y consolidación en un nodo
central para auditoría, topología y reportes.

## Archivos

- `model.arch` — instancia del `metamodel.tx` compartido. Describe el
  Sistema 1 como un `subsystem` con todos sus componentes (nodo de borde
  y nodo central) y las interfaces hacia Sistema 2 (Early Warning) y
  Sistema 5 (C4I).

## Responsable

- **Team leader:** Julián Bustos (`@dbustos106`)
- **Miembros:** César Pineda, Daniel Silva, José Alarcón, María Jara, Johan Medina

## Atributo de calidad principal

**Disponibilidad / operación en degraded connectivity.** El Sistema 1 debe
continuar capturando y procesando datos localmente incluso cuando se pierde
conectividad con el resto del SoS. Como consecuencia, el nodo central **no
está en la ruta crítica de alertas**: las alertas a S2 y S5 se publican
directamente desde el nodo de borde en tiempo real, y el nodo central sirve
únicamente funciones no críticas (agregación, auditoría, reportes, frontend
de operadores).

## Estructura del modelo

El Sistema 1 se organiza en **dos tipos de nodos complementarios**:

### Nodo de borde — captura y procesamiento en tiempo real (13 componentes)

- 1 edge node regional (tier `edge`) — runtime de contenedores
- 4 sensores físicos:
  - `hydro_sensor` → label del SVG: *Sensores hidrológicos*
  - `seismic_sensor` → label del SVG: *Sensores sísmicos*
  - `atmo_sensor` → label del SVG: *Sensores atmosféricos*
  - `camera` → label del SVG: *Cámaras*
- 3 microservicios del pipeline:
  - `data_ingestion_ms` — recibe y buferiza flujos crudos de los sensores
  - `data_standarization_ms` — normaliza formatos al modelo común del SoS
  - `publisher` — publica alertas y streams a S2 y S5 (ruta rápida)
- 5 componentes de almacenamiento local:
  - `db_raw_data_storage` + `bucket_raw_files_storage` — datos crudos
  - `processed_data_db` + `bucket_processing_file_storage` — datos procesados
  - `file_storage_metadata` — índice compartido entre raw y processed

### Nodo central — agregación, auditoría y consultas históricas (4 componentes)

- `processing_unit_ms` (logic / microservice) — Unidad Principal de
  procesamiento. Recibe datos del pipeline del edge vía `data_standarization_ms`,
  genera reportes consolidados, mantiene estado de topología, y expone la
  información hacia el frontend y hacia los sistemas externos del SoS.
- `topology_status_db` (data / database) — estado actual de los nodos de
  borde activos, cuándo reportaron por última vez, qué sensores tienen
  conectados.
- `report_db` (data / database) — histórico de alertas, trazabilidad de
  eventos, estadísticas agregadas, material para análisis post-evento.
- `operator_frontend` (presentation / web_ui) — interfaz web para que
  operadores humanos consulten topología y reportes (capa T1 del tiered model).

### Conectores internos del subsystem `data_acquisition_edge` (20)

- **Ingestión IoT (4):** 3 sensores telemétricos y 1 cámara hacia
  `data_ingestion_ms` — MQTT para telemetría, WebSockets para video.
- **Pipeline asincrónico (2):** `data_ingestion_ms → data_standarization_ms →
  publisher` vía message broker AMQP.
- **Persistencia en el edge (10):** escrituras de ingestión y estandarización
  sobre sus almacenamientos propios; dos **lecturas cruzadas** de
  `data_standarization_ms` sobre el storage raw de ingestión; y dos
  **lecturas del publisher** sobre el storage procesado
  (`publisher → processed_data_db` y `publisher → bucket_processing_file_storage`).
  El publisher recibe notificación por broker y lee los datos reales
  directamente del storage procesado (patrón Claim Check), evitando el
  overhead de pasar blobs grandes en base64 a través del broker.
- **Uplink edge → central (1):** `data_standarization_ms → processing_unit_ms`
  sobre Http con estilo RequestResponse.
- **Internos del nodo central (3):** `operator_frontend → processing_unit_ms`,
  `processing_unit_ms → topology_status_db`, `processing_unit_ms → report_db`.

### Subsystem `sos_frontier` (2 externos + 4 conectores SoS-level)

- **Componentes externos:**
  - `s2_early_warning` → label del SVG: *Sistema 2*
  - `s5_c4i` → label del SVG: *Sistema 5*
- **Conectores SoS-level:**
  - `publisher → s2_early_warning` (ruta rápida, alertas) — `event_notification` Http
  - `publisher → s5_c4i` (ruta rápida, streams operacionales) — `data_stream` Http
  - `processing_unit_ms → s2_early_warning` (ruta lenta, auditoría) — `data_stream` Http
  - `processing_unit_ms → s5_c4i` (ruta lenta, reportes históricos) — `data_stream` Http

Todos los conectores Http del modelo usan adicionalmente el atributo
`style=RequestResponse` introducido en la versión reciente del metamodel
compartido, explicitando que son comunicaciones petición-respuesta (en
contraste con los canales AMQP del pipeline interno, que son asincrónicos
tipo Pub/Sub o MessageQueue).

## Decisiones de nomenclatura frente al SVG

La mayoría de los identificadores del modelo coinciden literalmente con los
labels del SVG final (`Vista de componentes y conectores-new-final.drawio.svg`).
En los casos donde los labels del SVG no son compatibles con la sintaxis de
identificadores de textX (espacios, paréntesis, tildes), se aplicaron
conversiones a `snake_case` minúsculo preservando el orden y contenido del
label original:

| Label del SVG | Identificador en el modelo |
|---|---|
| `DB Raw data storage` | `db_raw_data_storage` |
| `Bucket (Raw Files Storage)` | `bucket_raw_files_storage` |
| `Processed data (DB)` | `processed_data_db` |
| `Bucket processing (File Storage)` | `bucket_processing_file_storage` |
| `File Storage Metada` | `file_storage_metadata` (corregido el typo `Metada`) |
| `Processing_unit_ms` | `processing_unit_ms` (minúscula inicial por consistencia snake_case) |

## Puntos abiertos para revisión

- **RTSP/LoRaWAN no están en el enum de protocolos del metamodel.** Las
  cámaras se modelan con `WebSockets` como aproximación y los sensores con
  `MQTT`. Si el equipo considera importante representar RTSP/LoRaWAN, hay
  que abrir un PR al `metamodel.tx` compartido.
- **Atributos de calidad (availability, performance, security)** no están
  en el metamodel actual. Se añadirán en un PR posterior, alineado con la
  extensión que se está haciendo en el Lab 3.
- **Client web browser del operador:** no se modeló como componente
  explícito para mantener el alcance del modelo dentro del Sistema 1. Si
  el equipo prefiere modelarlo como `external_agency` del subsystem
  `sos_frontier`, es un cambio menor en un PR futuro.
- **Typo en el label del SVG (`File Storage Metada`):** el modelo formal
  usa `file_storage_metadata` (con la `ta` final). Se recomienda corregir
  el SVG en el próximo refinamiento para mantener consistencia visual.

## Estado

- Modelo alineado con la versión final del diagrama del equipo
  (`Vista de componentes y conectores-new-final.drawio.svg`), usando la
  versión actualizada del `metamodel.tx` compartido (protocolo `Http` en
  lugar de `REST`, con estilo `RequestResponse` para comunicaciones
  sincrónicas). Incluye el patrón Claim Check del `publisher` sobre el
  storage procesado.
