==================================================
Asistente Tecnológico con Deepseek y Búsqueda en Documentos
==================================================

Objetivo
--------
Implementar un asistente que:
1. Aprenda de documentos técnicos en formatos PDF, DOCX, TXT
2. Realice búsquedas semánticas dentro del contenido
3. Genere respuestas precisas extrayendo información directa de los documentos
4. Utilice Deepseek como motor de inferencia

Arquitectura Detallada
----------------------
.. graphviz::
   digraph architecture {
      rankdir=LR;
      node [shape=box];
      
      Documentos -> "Procesamiento\n(División en chunks)" -> "Embeddings\n(vectores)";
      "Embeddings" -> "VectorDB\n(Chroma)";
      "Consulta" -> "Búsqueda Semántica" -> "Contexto Relevante";
      "Contexto Relevante" -> "Deepseek\n(Generación)";
      "Deepseek" -> "Respuesta\n(con citas exactas)";
   }

----------------------------
Paso 1: Configuración del Entorno
----------------------------

Requisitos Mínimos
^^^^^^^^^^^^^^^^^^
- Python 3.10+
- Deepseek API Key (obtenida desde platform.deepseek.com)
- Librerías esenciales:

.. code-block:: bash

   pip install "unstructured[local-inference]" langchain chromadb deepseek-rag sentence-transformers pypdf

Estructura de Directorios
^^^^^^^^^^^^^^^^^^^^^^^^^
::

   /asistente/
   │── documentos/                  # Almacena los archivos originales
   │── database/                    # Base de datos vectorial
   │── config.py                    # Parámetros del sistema
   │── processor.py                 # Procesamiento de documentos
   └── app.py                       # Interfaz principal

----------------------------
Paso 2: Procesamiento de Documentos
----------------------------

Configuración (config.py)
^^^^^^^^^^^^^^^^^^^^^^^^^
.. code-block:: python

   from langchain.text_splitter import MarkdownHeaderTextSplitter

   CHUNK_SIZE = 1500  # Tamaño de fragmentos
   CHUNK_OVERLAP = 200  # Solapamiento entre chunks
   EMBEDDING_MODEL = "sentence-transformers/all-mpnet-base-v2"

Procesador de Documentos (processor.py)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. code-block:: python

   from unstructured.partition.auto import partition
   from langchain.schema import Document

   def process_file(filepath: str) -> list[Document]:
       elements = partition(filename=filepath)
       chunks = []
       
       for element in elements:
           if hasattr(element, "text"):
               chunks.append(Document(
                   page_content=element.text,
                   metadata={
                       "source": filepath,
                       "page": getattr(element, "page_number", 0)
                   }
               ))
       return chunks

----------------------------
Paso 3: Sistema de Búsqueda
----------------------------

Indexación de Contenido
^^^^^^^^^^^^^^^^^^^^^^^
.. code-block:: python

   from langchain.vectorstores import Chroma
   from langchain.embeddings import HuggingFaceEmbeddings

   def create_vector_db(documents):
       embeddings = HuggingFaceEmbeddings(model_name=EMBEDDING_MODEL)
       return Chroma.from_documents(
           documents=documents,
           embedding=embeddings,
           persist_directory="./database"
       )

Búsqueda Semántica Avanzada
^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. code-block:: python

   def semantic_search(query, vectordb, top_k=3):
       results = vectordb.similarity_search_with_relevance_scores(
           query, 
           k=top_k,
           score_threshold=0.7
       )
       return [
           {
               "content": doc.page_content,
               "source": doc.metadata["source"],
               "page": doc.metadata.get("page", 0),
               "score": score
           } for doc, score in results
       ]

----------------------------
Paso 4: Integración con Deepseek
----------------------------

Generación de Respuestas
^^^^^^^^^^^^^^^^^^^^^^^^
.. code-block:: python

   from deepseek_api import Deepseek

   def generate_answer(contexts, question):
       llm = Deepseek(
           api_key=os.getenv("DEEPSEEK_API_KEY"),
           model="deepseek-tech-7b",
           temperature=0.3
       )
       
       context_str = "\n\n".join(
           f"DOCUMENTO {i+1} (Fuente: {ctx['source']}, Página {ctx['page']}):\n{ctx['content']}"
           for i, ctx in enumerate(contexts)
       )
       
       prompt = f"""
       Basado EXCLUSIVAMENTE en los siguientes documentos:
       {context_str}
       
       Responde esta pregunta: {question}
       - Sé preciso y técnico
       - Cita las fuentes exactas
       - Si no hay información clara, indica 'No encontrado en la documentación'
       """
       
       return llm.generate(prompt)

----------------------------
Paso 5: Interfaz de Consulta
----------------------------

API REST (app.py)
^^^^^^^^^^^^^^^^^
.. code-block:: python

   from fastapi import FastAPI
   from pydantic import BaseModel

   app = FastAPI()
   vectordb = Chroma(persist_directory="./database", 
                   embedding_function=HuggingFaceEmbeddings())

   class Query(BaseModel):
       question: str

   @app.post("/ask")
   async def ask_question(query: Query):
       contexts = semantic_search(query.question, vectordb)
       if not contexts:
           return {"answer": "No se encontró información relevante"}
       
       answer = generate_answer(contexts, query.question)
       return {
           "answer": answer,
           "sources": [{
               "document": ctx["source"],
               "page": ctx["page"],
               "relevance_score": ctx["score"]
           } for ctx in contexts]
       }

----------------------------
Paso 6: Despliegue
----------------------------

Ejecución Local
^^^^^^^^^^^^^^^
.. code-block:: bash

   # Procesar documentos iniciales
   python -c "from processor import process_file; import os; \
             docs = []; \
             for f in os.listdir('documentos'): \
                 docs.extend(process_file(f'documentos/{f}')); \
             create_vector_db(docs)"
   
   # Iniciar servidor
   uvicorn app:app --reload

Consulta de Ejemplo
^^^^^^^^^^^^^^^^^^^
.. code-block:: bash

   curl -X POST "http://localhost:8000/ask" \
   -H "Content-Type: application/json" \
   -d '{"question": "¿Cuál es el procedimiento exacto para configurar alertas por CPU en Zabbix según nuestros documentos?"}'

Respuesta Esperada
^^^^^^^^^^^^^^^^^^
.. code-block:: json

   {
       "answer": "Según el documento 'manual_zabbix.pdf' (página 42):\n1. Navegar a Configuration → Hosts\n2. Seleccionar el host deseado\n3. En la pestaña Triggers, hacer clic en Create Trigger\n4. Establecer la expresión: {host:system.cpu.load.avg(5m)}>5\n5. Definir severidad y acciones asociadas...",
       "sources": [
           {
               "document": "documentos/manual_zabbix.pdf",
               "page": 42,
               "relevance_score": 0.92
           }
       ]
   }

----------------------------
Mantenimiento Avanzado
----------------------------

Actualización de Documentos
^^^^^^^^^^^^^^^^^^^^^^^^^^
.. code-block:: python

   def update_documents(new_files):
       vectordb = Chroma(persist_directory="./database")
       for file in new_files:
           docs = process_file(file)
           vectordb.add_documents(docs)

Optimización de Búsqueda
^^^^^^^^^^^^^^^^^^^^^^^^
1. Filtrado por tipo de documento:
.. code-block:: python

   vectordb.similarity_search(
       query,
       filter={"source": {"$regex": ".*troubleshooting.*"}}
   )

2. Búsqueda híbrida (semántica + keywords):
.. code-block:: python

   from langchain.retrievers import BM25Retriever

   bm25_retriever = BM25Retriever.from_documents(documents)
   bm25_retriever.k = 2
   ensemble_retriever = EnsembleRetriever(
       retrievers=[vectordb.as_retriever(), bm25_retriever],
       weights=[0.7, 0.3]
   )

--------------------------------
Solución de Problemas Comunes
--------------------------------

+--------------------------------+-----------------------------------------------+
| Error                          | Solución                                      |
+================================+===============================================+
| "Unsupported file type"        | Instalar: ``pip install "unstructured[local-inference]"`` |
+--------------------------------+-----------------------------------------------+
| "Empty document content"       | Verificar permisos de lectura en archivos    |
+--------------------------------+-----------------------------------------------+
| "Low relevance scores"         | Ajustar CHUNK_SIZE y CHUNK_OVERLAP          |
+--------------------------------+-----------------------------------------------+
| "API timeout"                  | Implementar retry con backoff exponencial    |
+--------------------------------+-----------------------------------------------+
