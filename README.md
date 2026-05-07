# Sistema de Detección y Pixelado de Rostros de Menores

Sistema distribuido basado en arquitectura orientada a eventos (Event-Driven Architecture) que procesa imágenes para detectar rostros, estimar la edad de cada rostro y pixelar automáticamente los rostros de personas menores de 18 años.

---

## Requisitos previos

- Docker Desktop instalado y en ejecución
- Docker Compose

---

## Cómo ejecutar el sistema

### 1. Clonar el repositorio

```bash
git clone <url-del-repositorio>
cd proyecto_rostros
```

### 2. Levantar todos los servicios

```bash
docker-compose up --build
```

### 3. Crear los buckets en MinIO (solo la primera vez)

Accede a `http://localhost:9001` con las credenciales:
- **Usuario:** admin
- **Contraseña:** password123

Crea los siguientes buckets:
- `images-raw`
- `images-processed`

### 4. Subir una imagen para procesar

```bash
curl -X POST http://localhost:8000/upload -F "file=@tu_imagen.jpg"
```

La respuesta devolverá un `GUID_Solicitud` que identifica el trabajo.

### 5. Consultar el resultado

```bash
curl http://localhost:8001/resultado/{GUID_Solicitud}
```

Cuando el estado sea `FINALIZADA`, la respuesta incluirá la URL de la imagen procesada.

---

## Estructura del proyecto

```
proyecto_rostros/
├── api-1/                  # API de entrada (recibe imágenes)
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
├── detection-service/      # Detección de rostros con Haar Cascade
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
├── orchestrator-2/         # Orquestador: recorta caras y publica en Kafka
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
├── age-service/            # Clasificación de edad con MobileNetV2
│   ├── main.py
│   ├── modelo_menores.keras
│   ├── requirements.txt
│   └── Dockerfile
├── orchestrator-3/         # Orquestador: persiste resultados de edad en BD
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
├── pixelation/             # Pixelado de rostros de menores
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
├── orchestrator-4/         # Orquestador: marca la solicitud como FINALIZADA
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
├── api-2/                  # API de consulta de resultados
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
├── postgres/               # Scripts de inicialización de la BD
│   └── init.sql
├── ia_training/            # Dataset y scripts de entrenamiento del modelo
│   └── train_age.py
└── docker-compose.yml
```

---

## Descripción de cada servicio

### api-1 (puerto 8000)
Punto de entrada del sistema. Recibe imágenes mediante HTTP POST, las sube a MinIO, registra la solicitud en PostgreSQL y publica un evento en Kafka para iniciar el pipeline.

**Endpoint:** `POST /upload`

### detection-service
Consume eventos de Kafka, descarga la imagen de MinIO, detecta rostros usando el clasificador Haar Cascade de OpenCV y publica las bounding boxes de cada rostro detectado.

### orchestrator-2
Recorta cada rostro de la imagen original, sube cada recorte a MinIO como fichero independiente, registra cada cara en la base de datos y publica el payload enriquecido para el servicio de detección de edad.

### age-service
Descarga cada recorte de rostro de MinIO, lo procesa con el modelo MobileNetV2 entrenado (91.8% de accuracy en validación) y clasifica cada cara como menor (< 18 años) o adulto.

### orchestrator-3
Persiste los resultados de clasificación de edad (`Mayor_18` y `Escore`) en la tabla `Imagenes` de PostgreSQL y publica el evento para el servicio de pixelado.

### pixelation
Descarga la imagen original de MinIO, aplica pixelado (reducción a 16x16 y escalado) sobre los rostros clasificados como menores, y sube la imagen procesada al bucket `images-processed`.

### orchestrator-4
Recibe el evento de pixelado completado, actualiza el estado de la solicitud a `FINALIZADA` en PostgreSQL y registra la URL de la imagen procesada.

### api-2 (puerto 8001)
API de consulta. Permite obtener el estado y resultado de una solicitud por su GUID, así como la URL del recorte de una cara específica.

**Endpoints:**
- `GET /resultado/{guid}` — Estado y URL de la imagen procesada
- `GET /resultado/{guid}/cara/{id_cara}` — URL del recorte de una cara

---

## Topics y flujo de eventos

### Topics de Kafka

| Topic | Tipo | Publicado por | Consumido por |
|-------|------|---------------|---------------|
| `cmd.face_detection` | Comando | api-1 | detection-service |
| `evt.face_detection.completed` | Evento | detection-service | orchestrator-2 |
| `cmd.age_detection` | Comando | orchestrator-2 | age-service |
| `evt.age_detection.completed` | Evento | age-service | orchestrator-3 |
| `cmd.pixelation` | Comando | orchestrator-3 | pixelation |
| `evt.pixelation.completed` | Evento | pixelation | orchestrator-4 |
| `images.failed` | Dead Letter Queue | Todos | — |

### Flujo completo

```
Cliente
  → POST /upload (api-1)
  → cmd.face_detection
  → detection-service (Haar Cascade)
  → evt.face_detection.completed
  → orchestrator-2 (recorte de caras + BD)
  → cmd.age_detection
  → age-service (MobileNetV2)
  → evt.age_detection.completed
  → orchestrator-3 (persistencia BD)
  → cmd.pixelation
  → pixelation (OpenCV)
  → evt.pixelation.completed
  → orchestrator-4 (cierre solicitud)
  → GET /resultado/{guid} (api-2)
```

---

## Gestión de errores

### Dead Letter Queue
Todos los servicios capturan excepciones y publican un mensaje en el topic `images.failed` con el GUID de la solicitud, el error y el nombre del servicio que falló. Esto permite auditoría y reintento manual.

### Reintentos automáticos
Todos los servicios de aplicación tienen configurado `restart: on-failure` en Docker Compose, por lo que se reiniciarán automáticamente si fallan durante el arranque.

### Tolerancia a fallos de Kafka
Los consumidores de Kafka tienen configurado `auto.offset.reset: earliest`, lo que garantiza que los mensajes no se pierdan aunque un servicio esté temporalmente caído.

---

## Infraestructura

| Servicio | Puerto | Descripción |
|----------|--------|-------------|
| Kafka 4.2.0 | 9092 | Bus de mensajería (modo KRaft, sin Zookeeper) |
| MinIO | 9000 / 9001 | Almacenamiento de objetos compatible con S3 |
| PostgreSQL 15 | 5432 | Base de datos relacional |
| api-1 | 8000 | API de entrada |
| api-2 | 8001 | API de consulta |

### Credenciales MinIO
- **Usuario:** admin
- **Contraseña:** password123
- **Consola web:** http://localhost:9001

### Credenciales PostgreSQL
- **Base de datos:** db_rostros
- **Usuario:** user_rostros
- **Contraseña:** password_rostros

---

## Modelo de IA

El modelo de clasificación de edad es una red neuronal MobileNetV2 entrenada con transfer learning sobre el dataset Facial Age de Kaggle (~9.778 imágenes de caras etiquetadas por edad). Alcanza un 91.8% de accuracy en el conjunto de validación.

**Limitaciones conocidas:** el modelo puede tener dificultades para discriminar entre adultos y menores en fotos de grupo donde los rostros son pequeños o tienen características similares. En producción se recomienda el uso de modelos más especializados como DeepFace o APIs de estimación de edad.
