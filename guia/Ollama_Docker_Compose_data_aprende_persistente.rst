Instalación de Ollama con Docker Compose con aprendizaje y Persistencia de Datos
=================================================================================


Estructura final del proyecto
-----------------------------------

.. code-block:: bash

  /ollama-docker/
  ├── docker-compose.yml
  ├── models/       # Persistencia de modelos
  └── uploads/      # Persistencia de tus archivos

Paso 1: Crear estructura de directorios
------------------------------------------

.. code-block:: bash

  mkdir -p /ollama-docker/{models,uploads}

Paso 2: Crear archivo docker-compose.yml
------------------------------------------

Crea el archivo en ~ollama-docker/docker-compose.yml con:

.. code-block:: bash

  version: '3.8'
  
  services:
    ollama:
      image: ollama/ollama:latest
      container_name: ollama
      ports:
        - "11434:11434"
      volumes:
        - ./models:/root/.ollama
        - ./uploads:/uploads
      restart: unless-stopped

Paso 3: Iniciar el servicio
-------------------------------

.. code-block:: bash

  cd /ollama-docker
  docker compose up -d

Paso 4: Descargar un modelo
----------------------------

Descarga el modelo llama3

.. code-block:: bash

  docker exec ollama ollama pull llama3

Para ver modelos disponibles: 

.. code-block:: bash

  docker exec ollama ollama list

Paso 5: Copiar tus archivos Markdown
---------------------------------------

.. code-block:: bash

  cp /ruta/de/tus/archivos/*.md /ollama-docker/uploads/

Paso 6: Ingresar archivos manualmente al modelo
----------------------------------------------

Acceder al contenedor interactivamente:

.. code-block:: bash

  docker exec -it ollama sh

Hay dos formas de que ollama aprenda, este es una: Esto creará un modelo personalizado que incluye tu archivo como parte de su contexto.

.. code-block:: bash

  # ollama create nombre-del-modelo -f Modfile
  docker exec -it ollama-container create zabbix-assistant -f zabbix_context.md

Alternativa: esta es la otra forma. Usar el archivo en tiempo real
Si prefieres no crear un modelo personalizado, puedes pasar el archivo directamente en tu consulta:

.. code-block:: bash

  # ollama run llama3 "Resume este archivo:" --file datos.txt

En este ejemplo, dentro del contenedor, procesar archivos (ejemplo básico):

.. code-block:: bash

  for file in /uploads/*.md; do
    ollama run llama3 "Aprende este contenido: $(cat $file)"
  done


Paso 7: Consultar el modelo
------------------------------

Desde tu host:

.. code-block:: bash

  docker exec ollama ollama run llama3 "Resume los conceptos clave de los documentos que te proporcioné"

Comandos útiles:
--------------------

.. code-block:: bash

  Detener servicio: docker compose down
  
  Iniciar servicio: docker compose up -d
  
  Ver logs: docker compose logs -f

  Backup: tar -czvf ollama-backup.tar.gz ~/ollama-docker

Notas importantes
-------------------

Los modelos persisten en ./models

Tus archivos persisten en ./uploads

El modelo NO aprenderá automáticamente de los archivos - debes ingerirlos manualmente

Para mejor aprendizaje considera:

Convertir Markdown a texto plano

Dividir contenido en chunks

Usar embeddings o fine-tuning para mejor retención

