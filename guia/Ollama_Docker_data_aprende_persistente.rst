Instalación de Ollama en Docker con aprendizaje y Persistencia de Datos 
====================================================

Requisitos previos
--------------------
Docker instalado en tu sistema

Acceso a terminal/linea de comandos

Archivos Markdown que deseas cargar

Paso 1: Crear estructura de directorios
---------------------------------------

.. code-block:: bash

  mkdir -p ~/ollama-docker/{models,data,uploads}

Paso 2: Crear un Dockerfile personalizado
-------------------------------------------

Crea un archivo Dockerfile en ~/ollama-docker con:

.. code-block:: dockerfile

  FROM ollama/ollama:latest
  VOLUME /root/.ollama
  VOLUME /app/uploads
  EXPOSE 11434

Paso 3: Construir la imagen personalizada
--------------------------------------------

.. code-block:: bash

  cd ~/ollama-docker
  docker build -t custom-ollama .

Paso 4: Ejecutar el contenedor con volúmenes persistentes
-------------------------------------------------------------

.. code-block:: bash

  docker run -d
  --name ollama-container
  -v ~/ollama-docker/models:/root/.ollama
  -v ~/ollama-docker/uploads:/app/uploads
  -p 11434:11434
  custom-ollama

Paso 5: Descargar un modelo
-------------------------------

.. code-block:: bash

  docker exec ollama-container ollama pull llama3

(Opcional) Para ver modelos disponibles:

.. code-block:: bash

  docker exec ollama-container ollama list

Paso 6: Copiar tus archivos Markdown al contenedor
------------------------------------------------------

.. code-block:: bash

  cp /ruta/a/tus/archivos/*.md ~/ollama-docker/uploads/

Paso 7: Ingrasar información al modelo de ollama
------------------------------------------------

.. code-block:: python

  docker exec -it ollama-container create zabbix-assistant -f zabbix_context.md

Paso 9: Consultar el modelo con tu data
------------------------------------------

.. code-block:: bash

  docker exec ollama-container ollama run llama3 "Pregunta sobre el contenido de tus archivos Markdown"

Mantenimiento y backup
--------------------------

Para hacer backup de tus datos:

.. code-block:: bash

  tar -czvf ollama-backup.tar.gz ~/ollama-docker/models

Para restaurar:

.. code-block:: bash

  tar -xzvf ollama-backup.tar.gz -C ~/

Notas importantes
---------------------

Los volúmenes montados aseguran la persistencia incluso si el contenedor se destruye

El directorio models contiene los modelos descargados

El directorio uploads contiene tus archivos Markdown

Ajusta los nombres de modelos según tus necesidades (llama2, mistral, etc.)

Este enfoque garantiza que:

Los modelos descargados persistan en ~/ollama-docker/models

Tus archivos Markdown persistan en ~/ollama-docker/uploads

El contenedor pueda ser recreado sin pérdida de datos
