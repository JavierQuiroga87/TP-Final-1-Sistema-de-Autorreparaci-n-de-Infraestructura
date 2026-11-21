# Documentación Técnica — Área 4: Observabilidad y Comunicación (Integrante 4)

*Responsable:* Javier Alejandro Quiroga  
*Rol:* Integrante 4 — Ingeniero de Observabilidad y Comunicación  
*Rama asignada:* `feature/i4-observabilidad` *(ajustar al nombre real de tu rama)*

## Área 4 — Observabilidad NDJSON y Notificaciones Externas (Integrante 4 - Javier Alejandro Quiroga)

### 1. Responsabilidad y Alcance Técnico

El Área 4 garantiza la **trazabilidad forense no bloqueante** y el **alertado operacional resiliente** del Proyecto Fénix. Su responsabilidad es registrar cada transición de estado del receptor y del worker en formato estructurado NDJSON sin penalizar la latencia de procesamiento (desacoplando la escritura en disco mediante colas en memoria), gestionar la rotación y compresión histórica de logs, y despachar notificaciones hacia canales externos (Telegram / Slack) tolerando fallos de red y límites de tasa (*rate limiting*).

### 2. Módulos Asignados

- `src/fenix/utils/logging_json.py`: Pipeline de observabilidad estructurada y rotación segura:
  - **Registro No Bloqueante en RAM:** Implementación basada en `logging.handlers.QueueHandler` y `QueueListener` para desacoplar la emisión de eventos de las operaciones de E/S en disco, evitando cuellos de botella en el hilo principal del receptor FastAPI y del worker SQS.
  - **Formato NDJSON (Newline-Delimited JSON):** Serialización determinista de cada entrada de log en una línea JSON independiente con marca temporal UTC, nivel, módulo y contexto de correlación (`event_id`, `resource_id`).
  - **Rotación y Compresión Idempotente:** Rotación automática de archivos al alcanzar el umbral de **2 MB** y compresión posterior a formato Gzip (`fenix.log.gz`), asegurando idempotencia para evitar recomprimir archivos ya procesados.
- `src/fenix/services/notifications.py`: Conector externo de alertas operacionales:
  - **Integración Multi-Canal (Telegram Bot API / Slack):** Despacho de mensajes formateados con el resultado de las remediaciones convergentes y alertas críticas.
  - **Resiliencia y Control de Flujo:** Configuración estricta de *timeouts* de conexión/lectura para evitar bloqueos indefinidos y estrategia de *backoff* respetando la cabecera `Retry-After` ante respuestas `HTTP 429 (Too Many Requests)`.

### 3. Comandos de Demostración y Verificación

#### A. Auditoría Estática y Pruebas Unitarias
```bash
# Auditoría estática con Flake8 sobre los módulos asignados
flake8 src/fenix/utils/logging_json.py src/fenix/services/notifications.py

# Ejecución de pruebas asociadas a logging y notificaciones
python -m pytest tests/ -k "log or notif" -v

B. Verificación del Pipeline de Logs y Rotación

# Inspeccionar las últimas entradas estructuradas en formato NDJSON
tail -n 5 logs/fenix.log

# Verificar archivos rotados y comprimidos en Gzip
ls -lh logs/*.gz
