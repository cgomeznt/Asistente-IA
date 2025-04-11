Guía Detallada para Crear un Asistente de TI con Ollama en Docker
===============================================================

A continuación te proporciono un paso a paso completo para crear un asistente de TI que pueda aprender de documentos que le suministres, utilizando Ollama en contenedores Docker.

Requisitos Previos
------------------

* Docker instalado en tu sistema
* Conocimientos básicos de línea de comandos
* Al menos 8GB de RAM disponibles (16GB recomendado para mejores resultados)

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

Paso 4: Configurar el sistema de ingesta de documentos
-----------------------------------------------------

Crea un Dockerfile para tu aplicación:

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
   RUN pip install --no-cache-dir -r requirements.txt
   
   COPY . .
   
   EXPOSE 8000
   CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]

Crea un archivo requirements.txt:

.. code-block:: text

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


Crea un archivo app.py con el siguiente contenido inicial:

.. code-block:: python

   from fastapi import FastAPI, UploadFile, File, HTTPException
   from fastapi.middleware.cors import CORSMiddleware
   import os
   from typing import List
   from pydantic import BaseModel
   import ollama
   # AÃ± estas importaciones al inicio del archivo
   from langchain.document_loaders import DirectoryLoader
   from langchain.text_splitter import RecursiveCharacterTextSplitter
   from langchain.embeddings import HuggingFaceEmbeddings
   from langchain.vectorstores import Chroma
   from langchain.chains import RetrievalQA
   import os
   
   app = FastAPI()
   
   app.add_middleware(
       CORSMiddleware,
       allow_origins=["*"],
       allow_credentials=True,
       allow_methods=["*"],
       allow_headers=["*"],
   )
   
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
   @app.post("/upload/")
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
   @app.post("/ask/")
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


Paso 5: Configuración del sistema RAG (Retrieval-Augmented Generation)
---------------------------------------------------------------------

Esto ya esta integrado en el punto 4:

Modifica el app.py para incluir procesamiento de documentos:

.. code-block:: python

   # Añade estas importaciones al inicio del archivo
   from langchain.document_loaders import DirectoryLoader
   from langchain.text_splitter import RecursiveCharacterTextSplitter
   from langchain.embeddings import HuggingFaceEmbeddings
   from langchain.vectorstores import Chroma
   from langchain.chains import RetrievalQA
   import os

   # Configuración de embeddings
   embeddings = HuggingFaceEmbeddings(model_name="sentence-transformers/all-MiniLM-L6-v2")

   # Configuración del procesamiento de documentos
   def process_documents():
       loader = DirectoryLoader('uploads/', glob="**/*.*")
       documents = loader.load()
       
       text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
       texts = text_splitter.split_documents(documents)
       
       # Crear y persistir la base de datos vectorial
       db = Chroma.from_documents(texts, embeddings, persist_directory="db")
       db.persist()
       return db

   # Modifica la función upload_file
   @app.post("/upload/")
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

   # Modifica la función ask_question para usar RAG
   @app.post("/ask/")
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


Paso 6: Construir y ejecutar el sistema
---------------------------------------

Construye y levanta los contenedores:

.. code-block:: bash

   docker-compose up --build

Verifica que ambos servicios estén funcionando:

* Ollama: http://localhost:11434
* Asistente: http://localhost:8000

Paso 7: Uso del asistente
-------------------------

Sube documentos:

.. code-block:: bash

   curl -X POST -F "file=@contactos.txt" http://localhost:8000/upload/

Haz preguntas:

.. code-block:: bash

   curl -X POST -H "Content-Type: application/json" -d '{
  "question": "Qui es Carlos Gomez?"
}' http://localhost:8000/ask/
