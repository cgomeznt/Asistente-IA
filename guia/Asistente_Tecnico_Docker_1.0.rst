==================================================
Solución al Error "No matching distribution found for deepseek-rag"
==================================================

Causa del Error
---------------
El paquete ``deepseek-rag`` no está disponible en PyPI (el repositorio estándar de Python). Este es un paquete privado o requiere instalación desde una fuente alternativa.

----------------------------
Solución Paso a Paso
----------------------------

Paso 1: Modificar el archivo requirements.txt
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Reemplaza la línea ``deepseek-rag>=1.2.0`` con las dependencias equivalentes de código abierto:

.. code-block:: text
   :emphasize-lines: 3

   # requirements.txt actualizado
   langchain>=0.0.200
   sentence-transformers>=2.2.2
   unstructured[local-inference]>=0.7.0
   chromadb>=0.4.0
   fastapi>=0.95.0
   uvicorn>=0.21.0
   pypdf>=3.0.0
   python-dotenv>=1.0.0

Paso 2: Actualizar el Dockerfile
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Asegúrate que tu Dockerfile instale las dependencias específicas para procesamiento de documentos:

.. code-block:: dockerfile
   :emphasize-lines: 6-7

   FROM python:3.10-slim

   WORKDIR /app
   COPY . .

   RUN apt-get update && \
       apt-get install -y poppler-utils tesseract-ocr libgl1 && \
       rm -rf /var/lib/apt/lists/*

   RUN pip install --no-cache-dir -r requirements.txt

   EXPOSE 8000
   CMD ["uvicorn", "app:app", "--host", "0.0.0.0"]

Paso 3: Implementación Alternativa para Deepseek
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Usa directamente la API REST de Deepseek. Modifica tu código:

.. code-block:: python

   :emphasize-lines: 5-8

   # Reemplazo para deepseek-rag
   import requests

   def query_deepseek(prompt: str, api_key: str):
       headers = {"Authorization": f"Bearer {api_key}"}
       data = {
           "model": "deepseek-chat",
           "messages": [{"role": "user", "content": prompt}]
       }
       response = requests.post(
           "https://api.deepseek.com/v1/chat/completions",
           headers=headers,
           json=data
       )
       return response.json()["choices"][0]["message"]["content"]

----------------------------
Configuración Alternativa Completa
----------------------------

1. Nueva estructura de archivos:
   ::

     /app/
     │── documents/          # Directorio para documentos
     │── vector_db/          # Almacenamiento de embeddings
     ├── app.py              # Aplicación principal
     ├── requirements.txt    # Dependencias actualizadas
     └── .env               # Variables de entorno

2. Archivo ``app.py`` básico:

.. code-block:: python

   from fastapi import FastAPI
   from langchain.vectorstores import Chroma
   from langchain.embeddings import HuggingFaceEmbeddings

   app = FastAPI()
   embeddings = HuggingFaceEmbeddings(model_name="all-mpnet-base-v2")
   vectordb = Chroma(persist_directory="./vector_db", embedding_function=embeddings)

   @app.post("/query")
   async def query_endpoint(query: str):
       # 1. Búsqueda semántica
       docs = vectordb.similarity_search(query)
       
       # 2. Construir contexto
       context = "\n".join(doc.page_content for doc in docs)
       
       # 3. Consultar a Deepseek
       prompt = f"Contexto:\n{context}\n\nPregunta: {query}\nRespuesta:"
       answer = query_deepseek(prompt, os.getenv("DEEPSEEK_API_KEY"))
       
       return {"answer": answer}

----------------------------
Reconstrucción del Contenedor
----------------------------

.. code-block:: bash

   docker-compose down
   docker-compose build --no-cache
   docker-compose up -d

--------------------------------
Solución de Problemas Adicionales
--------------------------------

+--------------------------------+-----------------------------------------------+
| Error                          | Solución                                      |
+================================+===============================================+
| "libtesseract not found"       | Ejecutar:                                     |
|                                | ``apt-get install -y tesseract-ocr-all``      |
+--------------------------------+-----------------------------------------------+
| "CUDA not available"           | Usar imágenes Docker con soporte GPU o        |
|                                | configurar dispositivos en docker-compose.yml |
+--------------------------------+-----------------------------------------------+
| "API rate limit exceeded"      | Implementar caché con Redis o retry logic     |
+--------------------------------+-----------------------------------------------+

Recomendaciones Finales
----------------------- 
1. Para producción, considerar:
   - Autenticación en los endpoints
   - Limitación de tasa (rate limiting)
   - Monitorización del servicio

2. Para mejor rendimiento en búsquedas:
   ```python
   vectordb.similarity_search(
       query, 
       k=3, 
       filter={"source": {"$regex": ".*manual.*"}}  # Filtrado por nombre de archivo
   )
