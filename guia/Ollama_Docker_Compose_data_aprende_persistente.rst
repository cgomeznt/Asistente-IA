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

Hay dos formas de que ollama aprenda, este es una: Esto creará un modelo personalizado que incluye tu archivo como parte de su contexto.

Para crear el Modelo personalizado (en lo que se carguen más modelos, demandara más memoria). ollama carga los modelos en memoria:

Esta es la estructura dentro del contenedor y recuerda que debe tener permisos de LECTURA (rwx,r--,r--)

.. code-block:: bash

  /uploads/
    ├── Modfile
    └── contactos.txt  # Texto plano con datos simples


Esta es la estructura del Host:

.. code-block:: bash

  /ollama/uploads/
    ├── Modfile
    └── contactos.txt  # Texto plano con datos simples

Crear el archivo /uploads/Modfile con este contenido:

.. code-block:: bash

  FROM llama3
  SYSTEM """
  ### INSTRUCCIONES ESTRICTAS ###
  1. SOLO puedes responder con los datos del archivo adjunto.
  2. Si no hay datos, di: "No tengo informaciï¿½n".
  3. NUNCA inventes respuestas.
  
  ### CONTENIDO DEL ARCHIVO ###
  Nombre: Carlos, Telefono: 123456789, Email: carlos@example.com
  Nombre: Ana, Telefono: 987654321, Email: ana@example.com
  Nombre: Luis, Telefono: 5551234567
  """
  PARAMETER num_ctx 2048

**NOTA:** Aumenta num_ctx en el Modfile si el archivo es grande.

Para crear el modelo:

.. code-block:: bash

  # ollama create nombre-del-modelo -f Modfile
  docker exec -it ollama ollama create mis-contactos -f /uploads/Modfile

Para crear eliminar el modelo:

.. code-block:: bash

  docker exec -it ollama ollama rm mis-contactos 

Para probar el modelo:

.. code-block:: bash

  docker exec -it ollama ollama run mis-contactos "Dame el teléfono de Ana"
  # ó
  docker exec -it ollama ollama run mis-contactos "Dame el teléfono de Carlos"

Para listar todos los modelos

.. code-block:: bash

  docker exec -it ollama ollama list

Este es el contenido del archivo contenido.txt:

.. code-block:: bash

  Nombre: Carlos, Telefono: 123456789, Email: carlos@example.com
  Nombre: Ana, Telefono: 987654321, Email: ana@example.com
  Nombre: Luis, Telefono: 5551234567

El contenido se inyecta DIRECTAMENTE en el prompt del sistema usando $(cat /uploads/contactos.txt).

Eliminamos la dependencia de {{ .File }} que no estaba funcionando.

AL cargar el Modelo, queda persistente y aunque reinicies el contenedor o el Host, se tendrá la información del modelo cargado.


Este es otro ejemplo de un archivo Modfile

Alternativa: esta es la otra forma. Usar el archivo en tiempo real
Si prefieres no crear un modelo personalizado, puedes pasar el archivo directamente en tu consulta:

.. code-block:: bash

  # ollama run llama3 "Resume este archivo:" --file datos.txt

En este ejemplo, dentro del contenedor, procesar archivos (ejemplo básico):
Acceder al contenedor interactivamente:

.. code-block:: bash

  docker exec -it ollama sh
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

