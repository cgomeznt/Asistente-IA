Guía Detallada para Crear un Asistente de TI con Ollama en Docker Compose 
===============================================================

A continuación te proporciono un paso a paso completo para crear un asistente de TI que pueda aprender de documentos que le suministres, utilizando Ollama en contenedores Docker.

Esto utiliza
-------------------

ollama con el modelo llama3

ngnix

python

docker compose

Requisitos Previos
------------------

* Docker instalado en tu sistema
* Conocimientos básicos de línea de comandos.
* 8GB de RAM disponibles (16GB recomendado para mejores resultados)

Estructura Final del Proyecto
---------------------------
::

   ollama-assistant/
   ├── docker-compose.yml
   ├── Dockerfile
   ├── requirements.txt
   ├── app.py
   ├── index.html
   ├── nginx.conf
   ├── uploads/   (creado automáticamente)
   ├── db/         (creado automáticamente)
   └── docs/
       └── contactos.txt

Paso 1: Configuración inicial de Docker
---------------------------------------

Verifica que Docker esté instalado y funcionando:

.. code-block:: bash

   docker --version
   docker run hello-world

Crea una carpeta para tu proyecto:

.. code-block:: bash

   mkdir ollama-assistant && cd ollama-assistant

Paso 2: Configuración de Ollama en Docker
-----------------------------------------

Descarga la imagen de Ollama:

.. code-block:: bash

   docker pull ollama/ollama

Crea un volumen para persistir los modelos:

.. code-block:: bash

   docker volume create ollama_data

Inicia el contenedor de Ollama:

.. code-block:: bash

   docker run -d --name ollama -p 11434:11434 -v ollama_data:/root/.ollama ollama/ollama

Paso 3: Descargar e instalar un modelo de lenguaje
--------------------------------------------------

Descarga un modelo adecuado (por ejemplo, llama3 o mistral):

.. code-block:: bash

   docker exec ollama ollama pull llama3

(Este paso puede tomar varios minutos dependiendo de tu conexión a internet)

Verifica que el modelo se haya descargado correctamente:

.. code-block:: bash

   docker exec ollama ollama list

Paso 4: Configurar el sistema para el Asistente de TI con ingesta de documentos
-----------------------------------------------------

Crea el archivo **Dockerfile** para la aplicación:

.. code-block:: dockerfile

    FROM python:3.9-slim
    
    WORKDIR /app
    
    # Instalar dependencias del sistema
    RUN apt-get update && \
        apt-get install -y \
        tesseract-ocr \
        poppler-utils \
        libmagic-dev \
        && rm -rf /var/lib/apt/lists/*
    
    # Instalar dependencias Python
    COPY requirements.txt .
    RUN pip install --upgrade pip
    RUN pip install -r requirements.txt
    
    COPY . .
    
    # Instalar un servidor web simple para servir el index.html
    RUN apt-get update && apt-get install -y nginx && \
        rm -rf /var/lib/apt/lists/* && \
        mv index.html /var/www/html/
    
    # Configurar Nginx para servir la interfaz y redirigir API a FastAPI
    COPY nginx.conf /etc/nginx/nginx.conf
    
    RUN chown -R www-data:www-data /var/www/html && \
        chmod -R 755 /var/www/html
    
    # Puerto para FastAPI (8000) y para Nginx (80)
    EXPOSE 8000 80
    
    CMD ["sh", "-c", "nginx && uvicorn app:app --host 0.0.0.0 --port 8000"

Crear el archivo **docker-compose.yml**:

.. code-block:: docker-compose.yml

    version: '3.8'
    
    services:
      ollama:
        image: ollama/ollama
        ports:
          - "11434:11434"
        volumes:
          - ./models:/root/.ollama
          - ./uploads:/uploads
          - ollama_data:/root/.ollama
        restart: unless-stopped
    
      assistant:
        build: .
        ports:
          - "8000:8000"
          - "80:80"
        volumes:
          - ./uploads:/app/uploads
          - ./db:/app/db
        depends_on:
          - ollama
        environment:
          - OLLAMA_HOST=http://ollama:11434
        restart: unless-stopped
    
    volumes:
      ollama_data:


Crear archivo de Configuraciones para el Nginx:

Utilizamos el nginx para hacer proxypass de la pagina estatica y para el backend.

El archivo **nginx.conf**:

.. code-block:: bash

  user www-data;
  worker_processes auto;
  
  events {
      worker_connections 1024;
  }
  
  http {
      include mime.types;
      default_type application/octet-stream;
      sendfile on;
      keepalive_timeout 65;
  
      server {
          listen 80;
          server_name localhost;
          root /var/www/html;
          index index.html;
  
          location / {
              try_files $uri $uri/ /index.html;
          }
  
          location /api/ {
              proxy_pass http://localhost:8000/;
              proxy_set_header Host $host;
              proxy_set_header X-Real-IP $remote_addr;
              proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          }
      }
  }


Crea un archivo **requirements.txt**:

.. code-block:: requirements.txt

   fastapi
   uvicorn
   python-multipart
   langchain
   langchain-community
   langchain-huggingface
   sentence-transformers
   unstructured
   pdf2image
   pytesseract
   pymupdf
   chromadb
   ollama

Crea un archivo **app.py**, este es el Backend RAG (Retrieval-Augmented Generation):

.. code-block:: python

  from fastapi import FastAPI, UploadFile, File, HTTPException
  from fastapi.middleware.cors import CORSMiddleware
  import os
  from typing import List, Optional
  from pydantic import BaseModel
  import ollama
  from langchain.document_loaders import DirectoryLoader
  from langchain.text_splitter import RecursiveCharacterTextSplitter
  from langchain.embeddings import HuggingFaceEmbeddings
  from langchain.vectorstores import Chroma
  import os
  
  app = FastAPI()
  
  # ConfiguraciÃ³ORS mÃ¡especÃ­ca
  origins = [
      "http://localhost",
      "http://localhost:8000",
      "http://127.0.0.1",
      "http://127.0.0.1:8000",
      "http://10.134.4.13",
      "http://10.134.4.13:8000",
      # Agrega aquÃ­ualquier otro origen que necesites permitir
  ]
  
  app.add_middleware(
      CORSMiddleware,
      allow_origins=origins,
      allow_credentials=True,
      allow_methods=["*"],  # Permite todos los mÃ©dos
      allow_headers=["*"],  # Permite todos los headers
  )
  
  @app.options("/ask")
  async def options_ask():
      return {"message": "OK"}
  
  @app.options("/upload")
  async def options_upload():
      return {"message": "OK"}
  
  class Question(BaseModel):
      question: str
  
  # ConfiguraciÃ³e embeddings
  embeddings = HuggingFaceEmbeddings(model_name="sentence-transformers/all-MiniLM-L6-v2")
  
  # ConfiguraciÃ³el procesamiento de documentos
  def process_documents():
      loader = DirectoryLoader('uploads/', glob="**/*.*")
      documents = loader.load()
  
      text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
      texts = text_splitter.split_documents(documents)
  
      # Crear y persistir la base de datos vectorial
      db = Chroma.from_documents(texts, embeddings, persist_directory="db")
      db.persist()
      return db
  
  # Modifica la funciÃ³pload_file
  @app.post("/upload")
  async def upload_file(file: UploadFile = File(...)):
      try:
          os.makedirs("uploads", exist_ok=True)
          contents = await file.read()
          with open(f"uploads/{file.filename}", "wb") as f:
              f.write(contents)
  
          # Procesar el documento
          process_documents()
          return {"filename": file.filename, "message": "File uploaded and processed successfully"}
      except Exception as e:
          raise HTTPException(status_code=500, detail=str(e))
  
  # Modifica la funciÃ³sk_question para usar RAG
  @app.post("/ask")
  async def ask_question(question: Question):
      try:
          # Cargar la base de datos vectorial
          db = Chroma(persist_directory="db", embedding_function=embeddings)
          retriever = db.as_retriever()
  
          # Obtener documentos relevantes
          docs = retriever.get_relevant_documents(question.question)
          context = "\n\n".join([doc.page_content for doc in docs])
  
          # Crear prompt con contexto
          prompt = f"""
          Basado en el siguiente contexto, responde la pregunta.
          Contexto: {context}
          Pregunta: {question.question}
          Respuesta:
          """
  
          response = ollama.chat(
              model='llama3',
              messages=[{
                  'role': 'user',
                  'content': prompt,
              }]
          )
          return {"answer": response['message']['content']}
      except Exception as e:
          raise HTTPException(status_code=500, detail=str(e))
  
  if __name__ == "__main__":
      import uvicorn
      uvicorn.run(app, host="0.0.0.0", port=8000)


Crear el archivo **index.html**:

.. code-block:: bash




Crear un archivo de **documento** para el ejemplo:

.. code-block:: bash

   vi contactos.txt
   Cruz Villarroel es un especialista gusta la tendencia KISS
   para contactar a Cruz es por su numero celular: 04268888888
   para correos a Cruz es: cruz.villarroel@gmail.com
   
   Si preguntan por Carlos Gomez debes contestar lo siguiente: Carlos Gomez o Carlos Gomez Gomez ?
   Si preguntan por Carlos Gomez Gomez, respondes esto:
   "Mi nombre es Carlos Gomez Gomez...!!!
   Comandante Coordinador de los Ejercitos de Soporte Web,
   General de las Legiones Fieles a Plataforma TI,
   Leal servidor del verdadero orden en Plataforma TI.
   Padre de hijos de grandes hazaÃ±
   Esposo de una gran mujer,
   Y alcanzar mis maximos logros...
   En esta vida o en la otra. Â¡Que empiece Ya!"
   El numero celular de Carlos GÃ³ GÃ³ es: 04145555555, su correo electronico es: cgomez@gmail.com.ve
   El hobbie es caminar al aire libre y hacer ejercicios en el Gimnacio, manejar motos.
   
   Lendder Osta es especialista en PCI DSS.
   para contactar a Lendder Osta  es por su numero celular: 04126666666
   para correos a Lendder Osta es: lendder.osta@gmail.com
   
   
   Felix Vega es especialista en Plataforma TI.
   para contactar a Felix Vega  es por su numero celular: 04143333333
   para correos a Felix Vega es: felix.vega@gmail.com



Paso 5: Construir y ejecutar el sistema
---------------------------------------

Construye y levanta los contenedores:

.. code-block:: bash

   docker compose up --build

Tambien se puede construir y levanta los contenedores así:

.. code-block:: bash

   docker compose build

   docker compose up

Verifica que ambos servicios estén funcionando:

* Ollama: http://localhost:11434
* Asistente: http://localhost:8000

Paso 6: Uso del asistente
-------------------------

Subir documentos:

.. code-block:: bash

   curl -X POST -F "file=@contactos.txt" http://localhost:8000/upload

NOTA: Cada vez que se haga la carga de documentos se debe reiniciar el Aistente TI

Haz preguntas:

.. code-block:: bash

   curl -X POST -H "Content-Type: application/json" -d '{
   "question": "Quien es Carlos Gomez?"
   }' http://localhost:8000/ask

Ir a un navegador y colocar la URL:

http://localhost/


.. _ollama-desventajas-servidor:

Desventajas del Asistente de TI con Ollama en el Rendimiento del Servidor
=========================================================================

Consumo de Recursos Elevado
---------------------------
- **CPU y RAM**: Ollama (con modelos como LLaMA 2 o Mistral) consume grandes cantidades de CPU y RAM, afectando el rendimiento en servidores no dimensionados adecuadamente.
- **GPU**: En aceleración por GPU (CUDA/Metal), modelos grandes pueden saturar la VRAM, causando lentitud o cierres inesperados.

Latencia en Respuestas
----------------------
- Los modelos LLM generan respuestas con alta latencia, especialmente en servidores con recursos limitados o múltiples consultas concurrentes.

Escalabilidad Limitada
----------------------
- Diseñado para entornos locales o pequeños. No soporta bien múltiples usuarios simultáneos sin configuración adicional (balanceo de carga, clusters).

Uso de Almacenamiento
---------------------
- Los modelos descargados ocupan espacio significativo (ej: LLaMA 2 7B ≈4GB, versiones mayores >20GB), problemático en servidores con SSD/HDD limitados.

Falta de Optimización para Producción
-------------------------------------
- Inadecuado para entornos de alto tráfico. Carece de:
  - Cache de respuestas.
  - Rate limiting.
  - Balanceo automático de carga.

Dependencia de la Conexión (Remoto)
-----------------------------------
- Si se accede vía red, la latencia se suma al tiempo de inferencia, degradando la experiencia del usuario.

.. _soluciones-ollama:

Soluciones Recomendadas
=======================
- **Asignar más recursos**: Aumentar RAM, CPU y GPU (si aplica).
- **Modelos más pequeños**: Usar Phi-2, TinyLlama o Mistral 7B para reducir consumo.
- **Quantización**: Cargar modelos en 4-bit/8-bit para menor uso de memoria.
- **Contenedores**: Docker/Kubernetes para aislar recursos.
- **Balanceo de carga**: NGINX como proxy para distribuir solicitudes.

Conclusión
==========
Ollama es ideal para prototipado, pero no para producción escalable. Alternativas recomendadas:

- **Nube**: APIs de OpenAI, Gemini o Claude.
- **Autoalojadas optimizadas**: vLLM o Text Generation Inference.
