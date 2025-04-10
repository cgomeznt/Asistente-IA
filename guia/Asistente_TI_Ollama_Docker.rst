Guía Completa: Asistente de TI con Ollama en Docker
==================================================



.. contents:: Tabla de Contenidos
   :depth: 3
   :local:

Configuración Inicial
---------------------

Requisitos Previos
~~~~~~~~~~~~~~~~~

- Docker instalado
- 8GB RAM mínimo (16GB recomendado)
- Conocimientos básicos de línea de comandos

Paso 1: Configuración de Docker
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Verificar instalación
   docker --version
   docker run hello-world

   # Crear estructura de proyecto
   mkdir ollama-assistant && cd ollama-assistant

Paso 2: Instalación de Ollama
-----------------------------

.. code-block:: bash

   docker pull ollama/ollama
   docker volume create ollama_data
   docker run -d --name ollama -p 11434:11434 -v ollama_data:/root/.ollama ollama/ollama

Paso 3: Descarga de Modelos
---------------------------

.. code-block:: bash

   docker exec ollama ollama pull llama3
   docker exec ollama ollama list

Solución de Errores Comunes
---------------------------

Error: "ModuleNotFoundError: No module named 'ollama'"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. Modificar ``requirements.txt``:

   .. code-block:: text

      ollama

2. Reconstruir contenedores:

   .. code-block:: bash

      docker-compose down
      docker-compose build --no-cache
      docker-compose up

Error: "No space left on device"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Limpieza de Docker
   docker system prune -a
   docker volume prune

   # Aumentar espacio en Docker Desktop:
   Settings → Resources → Disk image size

Error: "langchain_community not found"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Actualizar ``requirements.txt``:

.. code-block:: text

   langchain-community

Error: "model 'llama3' not found"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Descargar modelo manualmente
   docker exec ollama ollama pull llama3

   # Verificar modelos
   docker exec ollama ollama list

Pruebas del Sistema
-------------------

Pruebas Básicas vía Terminal
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. Probar Ollama directamente:

   .. code-block:: bash

      curl http://localhost:11434/api/chat -d '{
        "model": "llama3",
        "messages": [{"role": "user", "content": "Hola"}]
      }'

2. Probar la API del asistente:

   .. code-block:: bash

      # Subir archivo
      curl -X POST -F "file=@documento.pdf" http://localhost:8000/upload/

      # Hacer pregunta
      curl -X POST -H "Content-Type: application/json" -d '{
        "question": "Resume el documento"
      }' http://localhost:8000/ask/

Interfaz Web para Pruebas
~~~~~~~~~~~~~~~~~~~~~~~~~

1. Crear archivo ``index.html``:

   .. code-block:: html

      <!DOCTYPE html>
      <html>
      <head>
          <title>Asistente TI</title>
          <style>
              body { font-family: Arial; max-width: 800px; margin: 0 auto; padding: 20px; }
              button { background: #4CAF50; color: white; padding: 10px; border: none; cursor: pointer; }
          </style>
      </head>
      <body>
          <h1>Asistente TI</h1>
          <input type="file" id="fileInput">
          <button onclick="upload()">Subir</button>
          <textarea id="question" placeholder="Pregunta..."></textarea>
          <button onclick="ask()">Preguntar</button>
          <div id="response"></div>

          <script>
              async function upload() {
                  const file = document.getElementById('fileInput').files[0];
                  const formData = new FormData();
                  formData.append('file', file);
                  await fetch('http://localhost:8000/upload/', { method: 'POST', body: formData });
              }

              async function ask() {
                  const question = document.getElementById('question').value;
                  const response = await fetch('http://localhost:8000/ask/', {
                      method: 'POST',
                      headers: { 'Content-Type': 'application/json' },
                      body: JSON.stringify({ question })
                  });
                  document.getElementById('response').innerText = (await response.json()).answer;
              }
          </script>
      </body>
      </html>

2. Acceder via navegador en ``http://localhost:8000``

Configuración Completa docker-compose.yml
----------------------------------------

.. code-block:: yaml

   version: '3.8'
   services:
     ollama:
       image: ollama/ollama
       ports: ["11434:11434"]
       volumes: ["ollama_data:/root/.ollama"]
       command: sh -c "ollama pull llama3 && ollama serve"
     
     assistant:
       build: .
       ports: ["8000:8000"]
       volumes: ["./uploads:/app/uploads", "./db:/app/db"]
       depends_on: ["ollama"]
       environment:
         OLLAMA_HOST: "http://ollama:11434"
     
   volumes:
     ollama_data:

Estructura de Archivos
----------------------

::

   ollama-assistant/
   ├── docker-compose.yml
   ├── Dockerfile
   ├── requirements.txt
   ├── app.py
   ├── index.html      (opcional)
   ├── uploads/        (creado automáticamente)
   └── db/             (creado automáticamente)
