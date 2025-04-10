Asistente TI con Ollama en Docker
################################

.. contents::
   :depth: 4
   :local:
   :backlinks: top

1. Configuración Inicial
=======================

1.1 Requisitos Previos
---------------------

- Docker instalado (versión 20.10.0+)
- Docker Compose (versión 2.0.0+)
- 8GB de RAM como mínimo (16GB recomendado)
- 20GB de espacio disponible en disco
- CPU con soporte para virtualización

1.2 Estructura del Proyecto
---------------------------

.. code-block:: bash

   mkdir ollama-assistant
   cd ollama-assistant
   mkdir -p {uploads,db,static}

2. Configuración de Docker
=========================

2.1 Dockerfile
-------------

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
   RUN pip install --no-cache-dir -r requirements.txt

   COPY . .

   EXPOSE 8000
   CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]

2.2 docker-compose.yml
----------------------

.. code-block:: yaml

   version: '3.8'

   services:
     ollama:
       image: ollama/ollama
       ports:
         - "11434:11434"
       volumes:
         - ollama_data:/root/.ollama
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
         OLLAMA_HOST: "http://ollama:11434"
       depends_on:
         - ollama

   volumes:
     ollama_data:

3. Implementación del Asistente
==============================

3.1 requirements.txt
-------------------

.. code-block:: text

   fastapi==0.104.1
   uvicorn==0.24.0
   python-multipart==0.0.6
   langchain==0.1.0
   langchain-community==0.0.20
   sentence-transformers==2.3.1
   unstructured==0.12.0
   pdf2image==1.17.0
   pytesseract==0.3.10
   pymupdf==1.23.21
   chromadb==0.4.22
   ollama==0.1.4

3.2 app.py (Implementación Principal)
------------------------------------

.. code-block:: python

   from fastapi import FastAPI, UploadFile, File, HTTPException
   from fastapi.middleware.cors import CORSMiddleware
   from pydantic import BaseModel
   import os
   import ollama
   from langchain.document_loaders import DirectoryLoader
   from langchain.text_splitter import RecursiveCharacterTextSplitter
   from langchain.vectorstores import Chroma
   from langchain.embeddings import HuggingFaceEmbeddings

   app = FastAPI()

   app.add_middleware(
       CORSMiddleware,
       allow_origins=["*"],
       allow_methods=["*"],
       allow_headers=["*"],
   )

   embeddings = HuggingFaceEmbeddings(
       model_name="sentence-transformers/all-MiniLM-L6-v2"
   )

   def process_documents():
       loader = DirectoryLoader('uploads/', glob="**/*.*")
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
       try:
           os.makedirs("uploads", exist_ok=True)
           contents = await file.read()
           with open(f"uploads/{file.filename}", "wb") as f:
               f.write(contents)
           
           process_documents()
           return {
               "filename": file.filename,
               "message": "File uploaded and processed successfully"
           }
       except Exception as e:
           raise HTTPException(status_code=500, detail=str(e))

   @app.post("/ask/")
   async def ask_question(question: Question):
       try:
           db = Chroma(
               persist_directory="db",
               embedding_function=embeddings
           )
           docs = db.similarity_search(question.question, k=3)
           context = "\n\n".join([doc.page_content for doc in docs])
           
           prompt = f"""
           Basado en el siguiente contexto, responde la pregunta.
           Contexto: {context}
           Pregunta: {question.question}
           Respuesta:
           """
           
           response = ollama.chat(
               model='llama3',
               messages=[{'role': 'user', 'content': prompt}]
           )
           return {"answer": response['message']['content']}
       except Exception as e:
           raise HTTPException(status_code=500, detail=str(e))

4. Interfaz Web para Pruebas
============================

4.1 static/index.html
---------------------

.. code-block:: html

   <!DOCTYPE html>
   <html>
   <head>
       <title>Asistente TI</title>
       <style>
           body { font-family: Arial; max-width: 800px; margin: 0 auto; padding: 20px; }
           .container { display: flex; flex-direction: column; gap: 20px; }
           textarea, input, button { padding: 10px; font-size: 16px; }
           button { cursor: pointer; background: #4CAF50; color: white; border: none; }
           #response { border: 1px solid #ddd; padding: 15px; min-height: 100px; }
       </style>
   </head>
   <body>
       <div class="container">
           <h1>Asistente TI con Ollama</h1>
           
           <h2>Subir Documento</h2>
           <input type="file" id="fileInput">
           <button onclick="uploadFile()">Subir Archivo</button>
           
           <h2>Hacer Pregunta</h2>
           <textarea id="questionInput" placeholder="Escribe tu pregunta aquí..."></textarea>
           <button onclick="askQuestion()">Preguntar</button>
           
           <h2>Respuesta:</h2>
           <div id="response"></div>
       </div>

       <script>
           async function uploadFile() {
               const fileInput = document.getElementById('fileInput');
               const responseDiv = document.getElementById('response');
               
               if (!fileInput.files.length) {
                   responseDiv.textContent = "Por favor selecciona un archivo";
                   return;
               }

               const formData = new FormData();
               formData.append('file', fileInput.files[0]);

               try {
                   const response = await fetch('http://localhost:8000/upload/', {
                       method: 'POST',
                       body: formData
                   });
                   const data = await response.json();
                   responseDiv.textContent = `Archivo subido: ${JSON.stringify(data)}`;
               } catch (error) {
                   responseDiv.textContent = `Error: ${error.message}`;
               }
           }

           async function askQuestion() {
               const question = document.getElementById('questionInput').value;
               const responseDiv = document.getElementById('response');
               
               if (!question) {
                   responseDiv.textContent = "Por favor escribe una pregunta";
                   return;
               }

               try {
                   const response = await fetch('http://localhost:8000/ask/', {
                       method: 'POST',
                       headers: {
                           'Content-Type': 'application/json',
                       },
                       body: JSON.stringify({ question: question })
                   });
                   const data = await response.json();
                   responseDiv.textContent = data.answer;
               } catch (error) {
                   responseDiv.textContent = `Error: ${error.message}`;
               }
           }
       </script>
   </body>
   </html>

5. Solución de Problemas
========================

5.1 Tabla de Errores Comunes
----------------------------

+--------------------------------+-----------------------------------------------+
| Error                          | Solución                                      |
+================================+===============================================+
| ModuleNotFoundError            | Verificar e instalar dependencias faltantes   |
|                                | Reconstruir imágenes Docker                   |
+--------------------------------+-----------------------------------------------+
| No space left on device        | Limpiar espacio en Docker:                    |
|                                | ``docker system prune -a --volumes``          |
+--------------------------------+-----------------------------------------------+
| Model 'llama3' not found       | Descargar modelo manualmente:                 |
|                                | ``docker exec ollama ollama pull llama3``     |
+--------------------------------+-----------------------------------------------+
| CUDA out of memory            | Reducir tamaño del modelo o aumentar RAM      |
+--------------------------------+-----------------------------------------------+

5.2 Comandos de Diagnóstico
---------------------------

.. code-block:: bash

   # Ver logs de Ollama
   docker logs ollama -f --tail 100

   # Ver logs del asistente
   docker-compose logs -f assistant

   # Ver uso de recursos
   docker stats

   # Listar modelos instalados
   docker exec ollama ollama list

6. Despliegue y Pruebas
=======================

6.1 Iniciar el Sistema
----------------------

.. code-block:: bash

   docker-compose up --build -d

6.2 Pruebas desde Terminal
--------------------------

Prueba de carga de documentos:

.. code-block:: bash

   curl -X POST -F "file=@documento.pdf" http://localhost:8000/upload/

Prueba de consulta:

.. code-block:: bash

   curl -X POST -H "Content-Type: application/json" \
   -d '{"question":"¿Qué contiene el documento?"}' \
   http://localhost:8000/ask/

6.3 Pruebas desde Navegador
---------------------------

Acceder a la interfaz web en:
``http://localhost:8000/static/index.html``

7. Referencias
=============

- `Documentación Oficial de Ollama <https://ollama.ai/>`_
- `Documentación de Docker <https://docs.docker.com/>`_
- `Documentación de FastAPI <https://fastapi.tiangolo.com/>`_
