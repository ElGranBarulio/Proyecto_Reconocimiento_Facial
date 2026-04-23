# Proyecto_Reconocimiento_Facial

proyecto_rostros/
│
├── 📂 api-1/                      # Servicio de subida de imágenes
│   ├── main.py
│   ├── Dockerfile
│   └── requirements.txt
│
├── 📂 api-2/                      # Servicio de consulta de resultados
│   ├── main.py
│   ├── Dockerfile
│   └── requirements.txt
│
├── 📂 detection-service/           # Detector de rostros (OpenCV)
│   ├── ⭐️ main.py                  # (Ajustado con scaleFactor=1.1, minNeighbors=7)
│   ├── Dockerfile
│   └── requirements.txt
│
├── 📂 age-detection/              # Tu IA de menores
│   ├── main.py
│   ├── modelo_menores.h5          # Tu modelo entrenado
│   ├── Dockerfile
│   └── requirements.txt
│
├── 📂 orchestrator-3/             # EL ARTISTA (Aplica el pixelado)
│   ├── ⭐️ main.py                  # (Código corregido con OpenCV y Boto3)
│   ├── ⭐️ Dockerfile              # (Corregido con libgl1 y libglib2.0-0)
│   └── ⭐️ requirements.txt        # (Debe incluir opencv-python-headless y boto3)
│
├── 📂 orchestrator-1/             # Otros orquestadores de flujo
├── 📂 orchestrator-2/
│
├── 📂 db/                         # Configuración de base de datos
│   └── init.sql
│
└── 📄 docker-compose.yml          # Orquestación de todos los contenedores
