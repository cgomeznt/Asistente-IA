==================================================
Asistente Técnico con Deepseek y RAG (Documentos)
==================================================

Objetivo
--------
Crear un asistente que:
1. Aprenda de documentos técnicos proporcionados
2. Responda preguntas basándose exclusivamente en la documentación cargada
3. Utilice Deepseek como modelo base
4. Implemente RAG (Retrieval-Augmented Generation)

Arquitectura del Sistema
------------------------
.. graphviz::
   digraph architecture {
      rankdir=LR;
      node [shape=box];
      
      Documentos -> VectorDB [label="Embeddings"];
      VectorDB -> Retriever;
      Pregunta -> Retriever;
      Retriever -> Deepseek [label="Contexto relevante"];
      Deepseek -> Respuesta;
   }

-------------------------
Paso 1: Preparación del Entorno
-------------------------

Requisitos
^^^^^^^^^^
- Python 3.10+
- Deepseek API Key (``dsk-xxxxx``)
- Librerías esenciales:

.. code-block:: bash

   pip install langchain deepseek-rag python-dotenv chromadb

Estructura de directorios
^^^^^^^^^^^^^^^^^^^^^^^^^
::

   /asistente_tecnico/
   │── documentos/          # Archivos a aprender (PDF, DOCX, TXT)
   │── vector_db/           # Base de datos de embeddings
   │── .env                 # Configuración
   └── asistente.py         # Código principal

-------------------------
Paso 2: Configuración Inicial
-------------------------

Archivo .env
^^^^^^^^^^^
.. code-block:: ini

   # Deepseek Configuration
   DEEPSEEK_API_KEY=tu_api_key_aqui
   MODEL_NAME=deepseek-technical-7b

   # RAG Settings
   CHUNK_SIZE=1500
   CHUNK_OVERLAP=200

-------------------------
Paso 3: Código del Asistente
-------------------------

.. code-block:: python

   from langchain.document_loaders import DirectoryLoader
   from langchain.text_splitter import RecursiveCharacterTextSplitter
   from langchain.embeddings import DeepseekEmbeddings
   from langchain.vectorstores import Chroma
   from langchain.chains import RetrievalQA
   from deepseek_api import DeepseekLLM

   def cargar_documentos():
       loader = DirectoryLoader('./documentos', glob="**/*.*")
       return loader.load()

   def crear_vector_db(docs):
       text_splitter = RecursiveCharacterTextSplitter(
           chunk_size=1500,
           chunk_overlap=200
       )
       splits = text_splitter.split_documents(docs)
       
       embeddings = DeepseekEmbeddings(
           model="text-embedding-3-large"
       )
       
       return Chroma.from_documents(
           documents=splits,
           embedding=embeddings,
           persist_directory="./vector_db"
       )

   def inicializar_asistente():
       llm = DeepseekLLM(api_key=os.getenv('DEEPSEEK_API_KEY'))
       vectordb = Chroma(
           persist_directory="./vector_db",
           embedding_function=DeepseekEmbeddings()
       )
       
       return RetrievalQA.from_chain_type(
           llm=llm,
           chain_type="stuff",
           retriever=vectordb.as_retriever(),
           return_source_documents=True
       )

   if __name__ == "__main__":
       # Primera ejecución
       documentos = cargar_documentos()
       crear_vector_db(documentos)
       
       # Uso normal
       qa = inicializar_asistente()
       while True:
           pregunta = input("> ")
           resultado = qa({"query": pregunta})
           print(f"Respuesta: {resultado['result']}")
           print(f"Fuentes: {[doc.metadata['source'] for doc in resultado['source_documents']]}")

-------------------------
Paso 4: Despliegue
-------------------------

Opción 1: CLI Local
^^^^^^^^^^^^^^^^^^^
.. code-block:: bash

   python asistente.py

   # Carga inicial de documentos
   > ¿Cómo configurar el servidor Zabbix?
   Respuesta: Según doc-zabbix.pdf, la configuración requiere...
   Fuentes: ['documentos/doc-zabbix.pdf']

Opción 2: API REST (FastAPI)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. code-block:: python

   from fastapi import FastAPI
   app = FastAPI()
   qa = inicializar_asistente()

   @app.post("/preguntar")
   async def preguntar(pregunta: str):
       resultado = qa({"query": pregunta})
       return {
           "respuesta": resultado["result"],
           "fuentes": [doc.metadata["source"] for doc in resultado["source_documents"]]
       }

Ejecutar con:
.. code-block:: bash

   uvicorn api:app --reload

-------------------------
Paso 5: Mantenimiento
-------------------------

Actualización de Documentos
^^^^^^^^^^^^^^^^^^^^^^^^^^
1. Añadir nuevos archivos a ``/documentos``
2. Reindexar:

.. code-block:: python

   def actualizar_conocimiento():
       documentos = cargar_documentos()
       crear_vector_db(documentos)

Monitorización
^^^^^^^^^^^^^^
- Verificar consumo de API
- Log de preguntas/respuestas:

.. code-block:: python

   import logging
   logging.basicConfig(filename='asistente.log', level=logging.INFO)

   def log_interaccion(pregunta, respuesta, fuentes):
       logging.info(f"P: {pregunta} | R: {respuesta} | F: {fuentes}")

--------------------------------
Solución de Problemas Comunes
--------------------------------

+--------------------------------+-----------------------------------------------+
| Error                          | Solución                                      |
+================================+===============================================+
| Formato de documento no soportado| Usar UnstructuredFileLoader (PDF/DOCX)      |
+--------------------------------+-----------------------------------------------+
| "I don't know" respuestas      | Ajustar chunk_size y overlap                 |
+--------------------------------+-----------------------------------------------+
| Alta latencia                  | Implementar cache con Redis                  |
+--------------------------------+-----------------------------------------------+

Mejoras Adicionales
-------------------
1. **Interfaz Web**: Streamlit o Gradio
2. **Histórico**: SQLite para conversaciones
3. **Evaluación**: Dataset de preguntas de prueba

Flujo de Trabajo Recomendado:
---------------------------------

Primera ejecución:

..  mkdir documentos
  cp manuales/*.pdf documentos/
  python asistente.py


Ejemplo de interacción:

.. ¿Cuál es el puerto por defecto de Zabbix?
  Respuesta: Según la documentación técnica (v7.2), Zabbix utiliza el puerto 10051 para...
  Fuentes: ['documentos/zabbix_manual.pdf', 'documentos/redes.docx']

Para producción:

.. docker build -t asistente-tecnico .
  docker run -p 8000:8000 -e DEEPSEEK_API_KEY=tu_key asistente-tecnico
