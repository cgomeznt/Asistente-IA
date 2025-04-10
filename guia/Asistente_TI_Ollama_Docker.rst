Asistente de TI con Ollama en Docker: Guía Completa
###################################################

.. contents::
   :depth: 4
   :local:
   :backlinks: top

Introducción
============

Este documento proporciona una guía completa para implementar un asistente de TI capaz de procesar y aprender de documentos, utilizando Ollama en contenedores Docker.

Requisitos del Sistema
----------------------

Hardware
~~~~~~~~

- Mínimo 8GB de RAM (16GB recomendado para modelos grandes)
- 20GB de espacio libre en disco
- CPU con soporte para virtualización

Software
~~~~~~~~

- Docker Engine 20.10.0+
- Docker Compose 2.0.0+
- Git (opcional para control de versiones)

Configuración Inicial
=====================

1. Preparación del Entorno
--------------------------

.. code-block:: bash

   # Crear estructura de directorios
   mkdir -p ollama-assistant/{uploads,db}
   cd ollama-assistant

2. Archivo Dockerfile
---------------------

.. code-block:: dockerfile

   # Dockerfile
   FROM python:3.9-slim

   WORKDIR /app

   # Instalar dependencias del sistema
   RUN apt-get update && \
       apt-get install -y \
       tesseract-ocr \
       poppler-utils \
       libmagic-dev \
       && rm -rf /var/lib/apt/lists/*

   # Copiar e instalar requisitos Python
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt

   # Copiar aplicación
   COPY . .

   # Puerto de exposición
   EXPOSE 8000

   # Comando de inicio
   CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]

3. Archivo requirements.txt
---------------------------

.. code-block:: text

   # Core
   fastapi==0.104.1
   uvicorn==0.24.0

   # Procesamiento de documentos
   langchain==0.1.0
   langchain-community==0.0.20
   unstructured==0.12.0
   pdf2image==1.17.0
   pytesseract==0.3.10
   pymupdf==1.23.21

   # Vector DB
   chromadb==0.4.22
   sentence-transformers==2.3.1

   # Ollama
   ollama==0.1.4

Implementación del Sistema
=========================

1. Configuración de docker-compose.yml
-------------------------------------

.. code-block:: yaml

   version: '3.8'

   services:
     ollama:
       image: ollama/ollama
       ports:
         - "11434:11434"
       volumes:
         - ollama_data:/root/.ollama
       deploy:
         resources:
           reservations:
             memory: 8G
       command: >
         sh -c "ollama pull llama3 && ollama serve"

     assistant:
       build: .
       ports:
         - "8000:8000"
       volumes:
         - ./uploads:/app/uploads
         - ./db:/app/db
       environment:
         - OLLAMA_HOST=http://ollama:11434
       depends_on:
         - ollama
       deploy:
         resources:
           reservations:
             memory: 4G

   volumes:
     ollama_data:
       driver: local

2. Archivo app.py (Core)
------------------------

.. code-block:: python

   """
   Módulo principal del Asistente de TI
   """
   from fastapi import FastAPI, UploadFile, File, HTTPException
   from fastapi.middleware.cors import CORSMiddleware
   from pydantic import BaseModel
   from typing import List
   import os
   import ollama
   from langchain.document_loaders import DirectoryLoader
   from langchain.text_splitter import RecursiveCharacterTextSplitter
   from langchain.vectorstores import Chroma
   from langchain.embeddings import HuggingFaceEmbeddings

   app = FastAPI(title="Asistente TI con Ollama")

   # Configuración CORS
   app.add_middleware(
       CORSMiddleware,
       allow_origins=["*"],
       allow_methods=["*"],
       allow_headers=["*"],
   )

   # Modelos Pydantic
   class Question(BaseModel):
       question: str
       context: bool = True

   # Configuración embeddings
   embeddings = HuggingFaceEmbeddings(
       model_name="sentence-transformers/all-MiniLM-L6-v2"
   )

   def process_documents():
       """Procesa documentos en el directorio uploads"""
       loader = DirectoryLoader(
           'uploads/',
           glob="**/*.*",
           use_multithreading=True
       )
       documents = loader.load()
       
       text_splitter = RecursiveCharacterTextSplitter(
           chunk_size=1000,
           chunk_overlap=200
       )
       texts = text_splitter.split_documents(documents)
       
       db = Chroma.from_documents(
           texts,
           embeddings,
           persist_directory="db"
       )
       db.persist()
       return db

   @app.post("/upload/")
   async def upload_file(file: UploadFile = File(...)):
       """Endpoint para carga de documentos"""
       try:
           os.makedirs("uploads", exist_ok=True)
           file_path = f"uploads/{file.filename}"
           
           with open(file_path, "wb") as f:
               contents = await file.read()
               f.write(contents)
           
           process_documents()
           return {
               "status": "success",
               "filename": file.filename,
               "size": f"{len(contents)/1024:.2f} KB"
           }
       except Exception as e:
           raise HTTPException(
               status_code=500,
               detail=f"Error processing file: {str(e)}"
           )

   @app.post("/ask/")
   async def ask_question(query: Question):
       """Endpoint para preguntas"""
       try:
           db = Chroma(
               persist_directory="db",
               embedding_function=embeddings
           )
           
           if query.context:
               docs = db.similarity_search(query.question, k=3)
               context = "\n".join([d.page_content for d in docs])
               prompt = f"""
               Contexto:
               {context}

               Pregunta:
               {query.question}

               Respuesta:
               """
           else:
               prompt = query.question
           
           response = ollama.chat(
               model='llama3',
               messages=[{'role': 'user', 'content': prompt}],
               options={'temperature': 0.7}
           )
           
           return {
               "question": query.question,
               "answer": response['message']['content'],
               "context_used": query.context
           }
       except Exception as e:
           raise HTTPException(
               status_code=500,
               detail=f"Error generating answer: {str(e)}"
           )

   @app.get("/models")
   async def list_models():
       """Lista modelos disponibles en Ollama"""
       try:
           return ollama.list()
       except Exception as e:
           raise HTTPException(
               status_code=500,
               detail=f"Error listing models: {str(e)}"
           )

Despliegue y Pruebas
====================

1. Iniciar el Sistema
---------------------

.. code-block:: bash

   docker-compose up --build -d

2. Pruebas de Funcionalidad
---------------------------

Prueba de Carga de Documentos:

.. code-block:: bash

   curl -X POST -F "file=@manual_tecnico.pdf" \
   http://localhost:8000/upload/

Prueba de Consulta:

.. code-block:: bash

   curl -X POST -H "Content-Type: application/json" \
   -d '{"question":"¿Qué medidas de seguridad menciona el documento?"}' \
   http://localhost:8000/ask/

3. Interfaz Web de Pruebas
--------------------------

Crear ``static/index.html``:

.. code-block:: html

   <!-- Contenido completo del HTML proporcionado anteriormente -->

Solución de Problemas
=====================

1. Errores Comunes
------------------

+--------------------------------+-----------------------------------------------+
| Error                          | Solución                                      |
+================================+===============================================+
| ``ModuleNotFoundError``        | Reconstruir contenedores con:                 |
|                                | ``docker-compose build --no-cache``           |
+--------------------------------+-----------------------------------------------+
| ``No space left on device``    | Limpiar espacio con:                          |
|                                | ``docker system prune -a --volumes``          |
+--------------------------------+-----------------------------------------------+
| ``Model not found``            | Descargar modelo manualmente:                 |
|                                | ``docker exec ollama ollama pull llama3``     |
+--------------------------------+-----------------------------------------------+
| ``CUDA out of memory``         | Reducir tamaño de modelo o aumentar RAM       |
+--------------------------------+-----------------------------------------------+

2. Monitoreo del Sistema
------------------------

Comandos útiles:

.. code-block:: bash

   # Ver uso de recursos
   docker stats

   # Ver logs de Ollama
   docker logs ollama -f --tail 100

   # Ver logs del asistente
   docker-compose logs -f assistant

Optimización y Mejoras
======================

1. Configuración Avanzada
-------------------------

Ajuste de Modelos:

.. code-block:: python

   response = ollama.chat(
       model='llama3',
       messages=[...],
       options={
           'temperature': 0.5,  # Controla creatividad (0-1)
           'num_ctx': 4096,     # Tamaño de contexto
           'num_predict': 512   # Longitud máxima de respuesta
       }
   )

2. Seguridad
------------

Configuración básica de autenticación:

.. code-block:: python

   from fastapi.security import HTTPBasic

   security = HTTPBasic()

   @app.post("/upload/")
   async def secure_upload(
       file: UploadFile = File(...),
       credentials: HTTPBasicCredentials = Depends(security)
   ):
       # Validar credenciales
       ...

Estructura Final del Proyecto
=============================

::

   ollama-assistant/
   ├── docker-compose.yml
   ├── Dockerfile
   ├── requirements.txt
   ├── app.py
   ├── static/
   │   └── index.html
   ├── uploads/
   ├── db/
   └── docs/
       └── manual.rst

Referencias
===========

- `Documentación Oficial de Ollama <https://ollama.ai/>`_
- `Documentación de LangChain <https://python.langchain.com/>`_
- `FastAPI Documentation <https://fastapi.tiangolo.com/>`_
