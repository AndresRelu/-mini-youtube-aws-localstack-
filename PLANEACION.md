# Práctica 2 — Mini YouTube con AWS (LocalStack)

## Objetivo General

Construir un sistema tipo "mini YouTube" donde el usuario suba un video en 1080p y el sistema, de forma **automática y asíncrona**, genere versiones en 720p y 480p utilizando FFMPEG. Todo orquestado mediante servicios de mensajería de AWS (SNS + SQS), simulados localmente con **LocalStack**.

---

## Requisitos de la Práctica

| Requisito | Tecnología |
|---|---|
| Subir video 1080p | S3 (bucket input) |
| Generar 720p y 480p | FFMPEG + Lambda |
| Almacenamiento de resultados | S3 (buckets output) |
| Metadatos de los videos | DynamoDB (LocalStack) |
| Mensajería entre sistemas (A2A) | SNS + SQS |
| Notificación al usuario (A2P) | SNS |
| Infraestructura como código | Terraform (IaC) |
| Controlar paralelismo | Concurrencia Reservada en Lambda |
| Frontend básico | React |
| Simulación de AWS local | LocalStack + Docker |
| Orquestación avanzada (opcional) | AWS Step Functions |

---

## Glosario de Conceptos

### AWS / Cloud

| Concepto | Definición |
|---|---|
| **AWS** | Amazon Web Services. Plataforma de servicios en la nube de Amazon. Ofrece cómputo, almacenamiento, mensajería, bases de datos, etc., todos accesibles por API. |
| **S3 (Simple Storage Service)** | Servicio de almacenamiento de objetos. Los archivos se guardan en "buckets" (contenedores). Ideal para guardar videos, imágenes, backups. No es un sistema de archivos tradicional; cada objeto tiene una clave (key) única. |
| **Lambda** | Servicio de cómputo **serverless** (sin servidor). Ejecutas código (funciones) solo cuando ocurre un evento. No pagas por tiempo en espera, solo por ejecución. Escala automáticamente. |
| **SNS (Simple Notification Service)** | Servicio de mensajería tipo **pub/sub** (publicar/suscribir). Un publicador manda un mensaje a un "topic" y todos los suscriptores lo reciben simultáneamente. Ideal para disparar múltiples acciones en paralelo con un solo evento. |
| **SQS (Simple Queue Service)** | Cola de mensajes **FIFO** (o estándar). Los mensajes se almacenan en la cola hasta que un consumidor (ej. Lambda) los procese. Garantiza que **ningún mensaje se pierda**, incluso si el consumidor falla temporalmente. |
| **DynamoDB** | Base de datos NoSQL de AWS. Almacena documentos JSON. Sin esquema fijo. Excelente para guardar metadatos de videos (estado, URLs, timestamps). |
| **API Gateway** | Servicio que expone tus Lambdas como endpoints HTTP/REST. Es el "puente" entre el frontend y los servicios internos de AWS. |
| **Presigned URL** | URL temporal y firmada que permite a un cliente subir o descargar un archivo directamente desde/hacia S3, sin pasar por tu servidor. Tiene fecha de expiración configurada. |
| **Step Functions** *(opcional)* | Servicio de orquestación de workflows. Define flujos con estados (esperar, procesar, reintentar, fallar). Cada paso puede ser una Lambda, una espera, una condición, etc. Ideal para flujos complejos con manejo de errores. |
| **IAM (Identity and Access Management)** | Sistema de permisos de AWS. Define qué servicios pueden hacer qué cosas. Ej: "esta Lambda tiene permiso para escribir en S3". En LocalStack se simplifica, pero en producción es crítico. |

### Mensajería A2A y A2P

| Concepto | Definición |
|---|---|
| **A2A (Application-to-Application)** | Notificaciones entre sistemas internos del backend. Ej: cuando la Lambda termina el transcode, notifica al servicio de metadatos para que actualice el estado del video. El usuario no se entera directamente. |
| **A2P (Application-to-Person)** | Notificaciones dirigidas al usuario final. Ej: "Tu video está listo". Puede ser un email, SMS, o un webhook que el frontend escucha via polling/WebSocket. |
| **Pub/Sub Pattern** | Patrón de diseño donde un "publicador" emite eventos sin saber quién los recibe. Los "suscriptores" se registran a los temas que les interesan. SNS implementa este patrón. Desacopla servicios. |
| **Dead Letter Queue (DLQ)** | Cola especial donde van a parar los mensajes que fallaron en procesarse después de N intentos. Permite revisar errores sin perder mensajes. |

### Herramientas de Desarrollo

| Concepto | Definición |
|---|---|
| **LocalStack** | Herramienta open source que emula servicios de AWS localmente usando Docker. Expone los mismos endpoints que AWS real (en `localhost:4566`). Tu código Python/JS con boto3/SDK funciona sin cambios al pasar a AWS real. |
| **Docker** | Plataforma de contenedores. Empaqueta aplicaciones con todas sus dependencias en una imagen portable. LocalStack corre dentro de un contenedor Docker. |
| **Docker Compose** | Herramienta para definir y levantar múltiples contenedores con un solo comando (`docker compose up`). Útil para correr LocalStack + el frontend juntos. |
| **FFMPEG** | Herramienta CLI de código abierto para procesar video y audio. Puede transcodificar resoluciones, cambiar formatos, ajustar bitrates, recortar, etc. Es el motor central del transcode en esta práctica. |
| **Lambda Layer** | "Capa" de dependencias reutilizables que se adjunta a una Lambda. En esta práctica, el binario de FFMPEG se empaqueta como un Layer para que las Lambdas puedan usarlo sin incluirlo en cada función. |
| **IaC (Infrastructure as Code)** | Filosofía de definir toda la infraestructura (buckets, colas, funciones, permisos) en archivos de código versionables (`.tf`, `.yaml`, etc.), en lugar de crearla manualmente en la consola. **Terraform** es la herramienta que usaremos. |
| **Terraform** | Herramienta de IaC de HashiCorp. Define recursos de infraestructura en archivos `.tf` (HCL). Tiene un provider oficial para LocalStack (`localstack/aws`). Permite destruir y recrear toda la infraestructura con un comando. |
| **Concurrencia Reservada** | Configuración de Lambda que limita cuántas instancias de una función pueden ejecutarse en paralelo. Ej: si suben 100 videos a la vez, sin límite, AWS lanzaría 100 Lambdas simultáneas. La concurrencia reservada evita que esto sature recursos o costos. |
| **boto3** | SDK oficial de AWS para Python. Permite interactuar con S3, SQS, SNS, DynamoDB, etc. desde código Python. Las Lambdas lo usan para descargar/subir archivos y publicar mensajes. |

---

## Arquitectura del Sistema

```
┌─────────────────────────────────────────────────────────┐
│                    USUARIO (Navegador)                   │
│                    [React Frontend]                      │
└──────────────────────────┬──────────────────────────────┘
                           │ 1. Solicita presigned URL
                           ▼
               ┌───────────────────────┐
               │  API Gateway + Lambda  │
               │  (genera presigned URL)│
               └───────────┬───────────┘
                           │ 2. Sube video directo a S3
                           ▼
               ┌───────────────────────┐
               │  S3: bucket-input     │  ◄── video_1080p.mp4
               └───────────┬───────────┘
                           │ 3. Evento S3 PUT
                           ▼
               ┌───────────────────────┐
               │  SNS: video-uploaded  │  ◄── Topic (pub/sub)
               └─────────┬─────┬──────┘
                         │     │ 4. Fan-out a 2 colas en paralelo
               ┌─────────┘     └──────────┐
               ▼                          ▼
  ┌────────────────────┐     ┌────────────────────┐
  │  SQS: queue-720p   │     │  SQS: queue-480p   │
  └─────────┬──────────┘     └──────────┬─────────┘
            │ 5a. Trigger                │ 5b. Trigger
            ▼                            ▼
  ┌────────────────────┐     ┌────────────────────┐
  │ Lambda: transcode  │     │ Lambda: transcode  │
  │       720p         │     │       480p         │
  │  (FFMPEG Layer)    │     │  (FFMPEG Layer)    │
  └─────────┬──────────┘     └──────────┬─────────┘
            │ 6a. Sube resultado         │ 6b. Sube resultado
            ▼                            ▼
  ┌─────────────────┐        ┌─────────────────────┐
  │ S3: bucket-720p │        │ S3: bucket-480p     │
  └─────────┬───────┘        └──────────┬──────────┘
            │                           │
            └──────────┬────────────────┘
                       │ 7. Ambas publican en SNS (A2A)
                       ▼
          ┌────────────────────────┐
          │  SNS: video-ready      │
          └────────┬───────┬───────┘
                   │       │
         ┌─────────┘       └──────────────┐
         ▼                                ▼
┌─────────────────────┐        ┌──────────────────────┐
│ SQS: metadata-queue │        │ SNS A2P:             │
│ Lambda actualiza    │        │ notificar al usuario │
│ DynamoDB            │        │ (webhook / email)    │
└─────────────────────┘        └──────────────────────┘
```

**Flujo resumido:**
1. Usuario sube video → S3 input
2. S3 dispara evento → SNS `video-uploaded`
3. SNS hace fan-out → SQS 720p y SQS 480p (en paralelo)
4. Lambdas consumen las colas → FFMPEG transcode → S3 outputs
5. Lambdas publican en SNS `video-ready` (A2A)
6. SNS notifica al usuario (A2P) y actualiza metadatos en DynamoDB

---

## Estructura del Proyecto

```
p2/
├── docker-compose.yml          # LocalStack + servicios
├── PLANEACION.md               # Este archivo
│
├── infra/                      # Terraform (IaC)
│   ├── main.tf                 # Provider LocalStack, backend
│   ├── variables.tf            # Variables reutilizables
│   ├── s3.tf                   # Definición de los 3 buckets
│   ├── sns.tf                  # Topics SNS
│   ├── sqs.tf                  # Colas SQS y suscripciones
│   ├── dynamodb.tf             # Tabla de metadatos
│   ├── lambda.tf               # Funciones Lambda + layers
│   ├── api_gateway.tf          # Endpoint para presigned URL
│   └── outputs.tf              # URLs y ARNs de recursos creados
│
├── lambdas/
│   ├── transcode/
│   │   ├── handler.py          # Lógica principal de transcode
│   │   └── requirements.txt    # boto3, etc.
│   ├── presigned_url/
│   │   └── handler.py          # Genera presigned URL para subir
│   ├── metadata/
│   │   └── handler.py          # Actualiza DynamoDB
│   └── layers/
│       └── ffmpeg/             # Binario FFMPEG para el Layer
│           └── bin/
│               └── ffmpeg
│
└── frontend/                   # React App
    ├── public/
    ├── src/
    │   ├── App.jsx
    │   ├── components/
    │   │   ├── Upload.jsx       # Formulario de subida
    │   │   └── VideoList.jsx    # Galería de videos
    │   └── api/
    │       └── s3.js            # Lógica para presigned URL + subida
    ├── package.json
    └── Dockerfile
```

---

## Fases de Implementación

### Fase 1 — Setup del Entorno Local

**Objetivo:** Tener LocalStack corriendo y verificar conectividad.

```bash
# Levantar LocalStack
docker run --rm -it \
  -p 127.0.0.1:4566:4566 \
  -p 127.0.0.1:4510-4559:4510-4559 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  localstack/localstack
```

> **¿Por qué `-v /var/run/docker.sock`?**
> Este flag monta el socket de Docker del host dentro del contenedor de LocalStack.
> Permite que LocalStack cree contenedores Docker internamente cuando ejecuta Lambdas.
> Sin esto, las Lambdas no pueden correr.

**Verificar que funciona:**
```bash
# Instalar awslocal (wrapper de AWS CLI para LocalStack)
pip install awscli-local

# Crear un bucket de prueba
awslocal s3 mb s3://test-bucket

# Listarlo
awslocal s3 ls
```

**Configurar AWS CLI para LocalStack:**
```bash
aws configure
# AWS Access Key ID: test
# AWS Secret Access Key: test
# Default region name: us-east-1
# Default output format: json
```

---

### Fase 2 — IaC con Terraform

**Objetivo:** Definir toda la infraestructura en código, reproducible con un comando.

**Instalar Terraform:**
```bash
# En Linux
wget https://releases.hashicorp.com/terraform/1.8.0/terraform_1.8.0_linux_amd64.zip
unzip terraform_1.8.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/
```

**`infra/main.tf` — Configuración del provider:**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true

  endpoints {
    s3       = "http://localhost:4566"
    sqs      = "http://localhost:4566"
    sns      = "http://localhost:4566"
    lambda   = "http://localhost:4566"
    dynamodb = "http://localhost:4566"
    iam      = "http://localhost:4566"
  }
}
```

**`infra/s3.tf` — Los 3 buckets:**
```hcl
resource "aws_s3_bucket" "input" {
  bucket = "video-input"
}

resource "aws_s3_bucket" "output_720p" {
  bucket = "video-720p"
}

resource "aws_s3_bucket" "output_480p" {
  bucket = "video-480p"
}

# Notificación: cuando se suba un objeto a input, disparar SNS
resource "aws_s3_bucket_notification" "input_notification" {
  bucket = aws_s3_bucket.input.id

  topic {
    topic_arn     = aws_sns_topic.video_uploaded.arn
    events        = ["s3:ObjectCreated:*"]
    filter_suffix = ".mp4"
  }
}
```

**`infra/sns.tf` — Topics:**
```hcl
resource "aws_sns_topic" "video_uploaded" {
  name = "video-uploaded"
}

resource "aws_sns_topic" "video_ready" {
  name = "video-ready"
}

# Suscribir SQS 720p al topic video-uploaded
resource "aws_sns_topic_subscription" "to_queue_720p" {
  topic_arn = aws_sns_topic.video_uploaded.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.queue_720p.arn
}

# Suscribir SQS 480p al topic video-uploaded
resource "aws_sns_topic_subscription" "to_queue_480p" {
  topic_arn = aws_sns_topic.video_uploaded.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.queue_480p.arn
}
```

**`infra/sqs.tf` — Colas:**
```hcl
resource "aws_sqs_queue" "queue_720p" {
  name                       = "transcode-720p"
  visibility_timeout_seconds = 300  # 5 min para que la Lambda procese
}

resource "aws_sqs_queue" "queue_480p" {
  name                       = "transcode-480p"
  visibility_timeout_seconds = 300
}

resource "aws_sqs_queue" "queue_metadata" {
  name = "metadata-update"
}
```

**Comandos Terraform:**
```bash
cd infra/
terraform init    # Descarga providers
terraform plan    # Preview de qué se va a crear
terraform apply   # Crear todo
terraform destroy # Destruir todo (para limpiar)
```

---

### Fase 3 — Lambda Layer con FFMPEG

**Objetivo:** Empaquetar el binario de FFMPEG para que las Lambdas lo usen sin incluirlo en cada una.

> **¿Por qué un Layer?**
> Las Lambdas tienen un límite de tamaño de paquete (~50MB comprimido).
> FFMPEG pesa ~80MB. Los Layers permiten hasta 250MB descomprimidos y se comparten entre funciones.

```bash
# Descargar FFMPEG estático (compatible con Amazon Linux 2)
mkdir -p lambdas/layers/ffmpeg/bin
cd lambdas/layers/ffmpeg/bin
wget https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz
tar -xf ffmpeg-release-amd64-static.tar.xz
cp ffmpeg-*/ffmpeg .
chmod +x ffmpeg

# Empaquetar el layer
cd lambdas/layers/ffmpeg
zip -r ffmpeg_layer.zip bin/
```

**En Terraform (`infra/lambda.tf`):**
```hcl
resource "aws_lambda_layer_version" "ffmpeg" {
  filename   = "../lambdas/layers/ffmpeg/ffmpeg_layer.zip"
  layer_name = "ffmpeg"
  compatible_runtimes = ["python3.12"]
}
```

---

### Fase 4 — Lambdas de Transcode

**Objetivo:** Función que lee de SQS, descarga video de S3, ejecuta FFMPEG, sube resultado.

**`lambdas/transcode/handler.py`:**
```python
import boto3
import subprocess
import os
import json

s3 = boto3.client("s3", endpoint_url="http://localhost:4566")
sns = boto3.client("sns", endpoint_url="http://localhost:4566")

FFMPEG_PATH = "/opt/bin/ffmpeg"   # /opt es donde se montan los Layers
OUTPUT_BUCKET_MAP = {
    "720p": os.environ.get("BUCKET_720P", "video-720p"),
    "480p": os.environ.get("BUCKET_480P", "video-480p"),
}
RESOLUTION_MAP = {
    "720p": "1280:720",
    "480p": "854:480",
}
SNS_VIDEO_READY_ARN = os.environ.get("SNS_VIDEO_READY_ARN")


def handler(event, context):
    resolution = os.environ.get("RESOLUTION")  # "720p" o "480p"

    for record in event["Records"]:
        # SQS → SNS wrapping: el body del mensaje SQS contiene el mensaje SNS
        body = json.loads(record["body"])
        sns_message = json.loads(body["Message"])

        # El mensaje SNS de S3 contiene el nombre del bucket y la key
        s3_event = sns_message["Records"][0]["s3"]
        input_bucket = s3_event["bucket"]["name"]
        video_key = s3_event["object"]["key"]

        # Rutas temporales en /tmp (único directorio escribible en Lambda)
        input_path = f"/tmp/input_{os.path.basename(video_key)}"
        output_filename = video_key.replace(".mp4", f"_{resolution}.mp4")
        output_path = f"/tmp/{output_filename}"

        # 1. Descargar video del bucket input
        print(f"Descargando {video_key} de {input_bucket}")
        s3.download_file(input_bucket, video_key, input_path)

        # 2. Transcodificar con FFMPEG
        scale = RESOLUTION_MAP[resolution]
        cmd = [
            FFMPEG_PATH,
            "-i", input_path,
            "-vf", f"scale={scale}",
            "-c:v", "libx264",
            "-preset", "fast",    # balance velocidad/calidad
            "-crf", "23",         # calidad constante (0=mejor, 51=peor)
            "-c:a", "aac",
            "-y",                 # sobreescribir si existe
            output_path,
        ]
        print(f"Ejecutando FFMPEG: {' '.join(cmd)}")
        result = subprocess.run(cmd, capture_output=True, text=True)

        if result.returncode != 0:
            print(f"Error FFMPEG: {result.stderr}")
            raise Exception(f"FFMPEG falló con código {result.returncode}")

        # 3. Subir resultado al bucket output
        output_bucket = OUTPUT_BUCKET_MAP[resolution]
        print(f"Subiendo {output_filename} a {output_bucket}")
        s3.upload_file(output_path, output_bucket, output_filename)

        # 4. Publicar en SNS video-ready (A2A)
        sns.publish(
            TopicArn=SNS_VIDEO_READY_ARN,
            Message=json.dumps({
                "video_key": video_key,
                "resolution": resolution,
                "output_bucket": output_bucket,
                "output_key": output_filename,
            }),
            Subject="video-transcoded",
        )

        # Limpiar /tmp
        os.remove(input_path)
        os.remove(output_path)

    return {"statusCode": 200}
```

**Concurrencia Reservada en Terraform:**
```hcl
resource "aws_lambda_function" "transcode_720p" {
  filename      = "../lambdas/transcode/transcode.zip"
  function_name = "transcode-720p"
  role          = aws_iam_role.lambda_role.arn
  handler       = "handler.handler"
  runtime       = "python3.12"
  timeout       = 300
  memory_size   = 1024  # FFMPEG necesita memoria

  layers = [aws_lambda_layer_version.ffmpeg.arn]

  # CONCURRENCIA RESERVADA: máximo 5 ejecuciones simultáneas
  reserved_concurrent_executions = 5

  environment {
    variables = {
      RESOLUTION        = "720p"
      BUCKET_720P       = "video-720p"
      SNS_VIDEO_READY_ARN = aws_sns_topic.video_ready.arn
    }
  }
}
```

> **¿Por qué `memory_size = 1024`?**
> En AWS Lambda, la CPU asignada es proporcional a la memoria. A mayor memoria, más CPU.
> FFMPEG es intensivo en CPU; con 1024MB obtenemos suficiente para transcodificar en tiempo razonable.

---

### Fase 5 — Lambda de Metadatos

**`lambdas/metadata/handler.py`:**
```python
import boto3
import json
import os
from datetime import datetime

dynamodb = boto3.resource("dynamodb", endpoint_url="http://localhost:4566")
table = dynamodb.Table(os.environ.get("DYNAMODB_TABLE", "videos-metadata"))


def handler(event, context):
    for record in event["Records"]:
        body = json.loads(record["body"])
        message = json.loads(body["Message"])

        video_key = message["video_key"]
        resolution = message["resolution"]
        output_key = message["output_key"]
        output_bucket = message["output_bucket"]

        # Actualizar o crear item en DynamoDB
        table.update_item(
            Key={"video_id": video_key},
            UpdateExpression="SET #res = :url, updated_at = :ts",
            ExpressionAttributeNames={"#res": resolution},
            ExpressionAttributeValues={
                ":url": f"s3://{output_bucket}/{output_key}",
                ":ts": datetime.utcnow().isoformat(),
            },
        )
        print(f"Metadatos actualizados para {video_key} - {resolution}")
```

**`infra/dynamodb.tf`:**
```hcl
resource "aws_dynamodb_table" "videos_metadata" {
  name         = "videos-metadata"
  billing_mode = "PAY_PER_REQUEST"  # sin provisionar capacidad
  hash_key     = "video_id"

  attribute {
    name = "video_id"
    type = "S"  # String
  }
}
```

---

### Fase 6 — API Gateway + Lambda Presigned URL

**`lambdas/presigned_url/handler.py`:**
```python
import boto3
import json
import uuid
import os

s3 = boto3.client(
    "s3",
    endpoint_url="http://localhost:4566",
    region_name="us-east-1",
)
BUCKET_INPUT = os.environ.get("BUCKET_INPUT", "video-input")


def handler(event, context):
    body = json.loads(event.get("body", "{}"))
    filename = body.get("filename", f"{uuid.uuid4()}.mp4")
    video_key = f"uploads/{uuid.uuid4()}_{filename}"

    # Generar URL pre-firmada válida por 10 minutos
    presigned_url = s3.generate_presigned_url(
        "put_object",
        Params={"Bucket": BUCKET_INPUT, "Key": video_key},
        ExpiresIn=600,
    )

    return {
        "statusCode": 200,
        "headers": {"Access-Control-Allow-Origin": "*"},
        "body": json.dumps({
            "upload_url": presigned_url,
            "video_key": video_key,
        }),
    }
```

---

### Fase 7 — Frontend React

**Objetivo:** Interfaz mínima para subir videos y ver la galería con estados.

**`frontend/src/components/Upload.jsx`:**
```jsx
import { useState } from "react";

const API_URL = "http://localhost:4566"; // API Gateway local

export default function Upload() {
  const [file, setFile] = useState(null);
  const [status, setStatus] = useState("");
  const [progress, setProgress] = useState(0);

  const handleUpload = async () => {
    if (!file) return;
    setStatus("Obteniendo URL de subida...");

    // 1. Pedir presigned URL al backend
    const res = await fetch(`${API_URL}/upload-url`, {
      method: "POST",
      body: JSON.stringify({ filename: file.name }),
    });
    const { upload_url, video_key } = await res.json();

    // 2. Subir directamente a S3 con la presigned URL
    setStatus("Subiendo video...");
    await fetch(upload_url, {
      method: "PUT",
      body: file,
      headers: { "Content-Type": "video/mp4" },
    });

    setStatus(`¡Video subido! Procesando... (key: ${video_key})`);
  };

  return (
    <div className="upload-container">
      <h2>Subir Video</h2>
      <input
        type="file"
        accept="video/mp4"
        onChange={(e) => setFile(e.target.files[0])}
      />
      <button onClick={handleUpload} disabled={!file}>
        Subir
      </button>
      {status && <p>{status}</p>}
    </div>
  );
}
```

**`frontend/src/components/VideoList.jsx`:**
```jsx
import { useEffect, useState } from "react";

export default function VideoList() {
  const [videos, setVideos] = useState([]);

  // Polling cada 5 segundos para ver si los videos están listos
  useEffect(() => {
    const interval = setInterval(async () => {
      const res = await fetch("http://localhost:4566/videos");
      const data = await res.json();
      setVideos(data);
    }, 5000);
    return () => clearInterval(interval);
  }, []);

  return (
    <div>
      <h2>Mis Videos</h2>
      {videos.map((video) => (
        <div key={video.video_id} className="video-card">
          <h3>{video.video_id}</h3>
          <p>Estado: {video["720p"] && video["480p"] ? "✅ Listo" : "⏳ Procesando"}</p>
          {video["720p"] && <a href={video["720p"]}>Ver 720p</a>}
          {video["480p"] && <a href={video["480p"]}>Ver 480p</a>}
        </div>
      ))}
    </div>
  );
}
```

---

### Fase 8 — Docker Compose

**`docker-compose.yml`:**
```yaml
version: "3.8"

services:
  localstack:
    image: localstack/localstack
    ports:
      - "127.0.0.1:4566:4566"
      - "127.0.0.1:4510-4559:4510-4559"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - SERVICES=s3,sqs,sns,lambda,dynamodb,apigateway,iam
      - DEBUG=1
      - LAMBDA_EXECUTOR=docker
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4566/_localstack/health"]
      interval: 10s
      timeout: 5s
      retries: 5

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:4566
    depends_on:
      localstack:
        condition: service_healthy
```

---

### Fase 9 (Opcional) — AWS Step Functions

**Objetivo:** Reemplazar la coordinación manual SNS/SQS por un workflow explícito con estados.

```json
{
  "Comment": "Pipeline de transcode de video",
  "StartAt": "TranscodeParallel",
  "States": {
    "TranscodeParallel": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "Transcode720p",
          "States": {
            "Transcode720p": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:000000000000:function:transcode-720p",
              "End": true,
              "Retry": [{"ErrorEquals": ["States.ALL"], "MaxAttempts": 3}]
            }
          }
        },
        {
          "StartAt": "Transcode480p",
          "States": {
            "Transcode480p": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:000000000000:function:transcode-480p",
              "End": true,
              "Retry": [{"ErrorEquals": ["States.ALL"], "MaxAttempts": 3}]
            }
          }
        }
      ],
      "Next": "UpdateMetadata"
    },
    "UpdateMetadata": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:000000000000:function:update-metadata",
      "Next": "NotifyUser"
    },
    "NotifyUser": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:000000000000:video-ready",
        "Message.$": "$.video_key"
      },
      "End": true
    }
  }
}
```

> **Ventaja sobre SNS/SQS manual:**
> Si `Transcode720p` falla, Step Functions reintenta automáticamente hasta 3 veces.
> El estado es visible en un dashboard. Si falla definitivamente, puedes ver exactamente en qué paso y por qué.

---

## Comandos de Referencia Rápida

```bash
# Levantar todo
docker compose up -d

# Crear infraestructura
cd infra && terraform apply

# Verificar buckets
awslocal s3 ls

# Verificar topics SNS
awslocal sns list-topics

# Verificar colas SQS
awslocal sqs list-queues

# Subir video de prueba manualmente
awslocal s3 cp test_video.mp4 s3://video-input/test_video.mp4

# Ver mensajes en una cola
awslocal sqs receive-message --queue-url http://localhost:4566/000000000000/transcode-720p

# Ver logs de una Lambda
awslocal logs get-log-events \
  --log-group-name /aws/lambda/transcode-720p \
  --log-stream-name $(awslocal logs describe-log-streams \
    --log-group-name /aws/lambda/transcode-720p \
    --query 'logStreams[-1].logStreamName' --output text)

# Ver items en DynamoDB
awslocal dynamodb scan --table-name videos-metadata
```

---

## Orden de Implementación Recomendado

| Paso | Tarea | Validación |
|---|---|---|
| 1 | Levantar LocalStack | `curl http://localhost:4566/_localstack/health` |
| 2 | Terraform: S3 buckets | `awslocal s3 ls` |
| 3 | Terraform: SNS + SQS + suscripciones | `awslocal sns list-subscriptions` |
| 4 | Terraform: DynamoDB | `awslocal dynamodb list-tables` |
| 5 | Lambda presigned URL | Probar con curl, obtener URL |
| 6 | FFMPEG Layer | Descomprimir en Lambda y ejecutar `ffmpeg -version` |
| 7 | Lambda transcode (sin Lambda aún, probar local) | Ejecutar handler.py con event mock |
| 8 | Desplegar Lambdas + conectar triggers SQS | Subir video y ver que se procesan |
| 9 | Lambda metadatos + DynamoDB | Escanear tabla y ver registros |
| 10 | Frontend React: Upload + VideoList | Subir video desde UI |
| 11 | Docker Compose integrando todo | `docker compose up` y flujo completo |
| 12 | Step Functions (opcional) | Ver ejecución en dashboard LocalStack |

---

## Recursos Adicionales

- [LocalStack Docs](https://docs.localstack.cloud)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [FFMPEG Documentation](https://ffmpeg.org/documentation.html)
- [boto3 S3 Docs](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html)
- [AWS SNS Fanout Pattern](https://docs.aws.amazon.com/sns/latest/dg/sns-common-scenarios.html)
