============================================
Despliegue del Asistente Técnico en Docker
============================================

Objetivo
--------
Crear un contenedor Docker que:
1. Empaquete el asistente técnico basado en Deepseek
2. Permita la carga persistente de documentos
3. Exponga una API REST para consultas

----------------------------
Paso 1: Estructura del Proyecto
----------------------------

Estructura de archivos requerida:
::

   /asistente_docker/
   │── Dockerfile
   │── docker-compose.yml
   │── requirements.txt
   │── app/
   │   ├── documentos/          # Montado como volumen
   │   ├── vector_db/           # Persistencia de embeddings
   │   ├── main.py              # Código principal (FastAPI)
   │   └── .env                 # Configuración

----------------------------
Paso 2: Configuración de Archivos
----------------------------

Dockerfile
^^^^^^^^^^
.. code-block:: dockerfile

   # Base image
   FROM python:3.10-slim

   # Configuración inicial
   WORKDIR /app
   ENV PYTHONUNBUFFERED=1

   # Instalación de dependencias
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt

   # Copiar aplicación
   COPY . .

   # Variables de entorno (alternativa al .env)
   ENV DEEPSEEK_API_KEY=tu_api_key
   ENV MODEL_NAME=deepseek-technical-7b

   # Puerto de exposición
   EXPOSE 8000

   # Comando de inicio
   CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--reload"]

requirements.txt
^^^^^^^^^^^^^^^
.. code-block:: text

   fastapi>=0.95.0
   uvicorn>=0.21.0
   langchain>=0.0.200
   deepseek-rag>=1.2.0
   python-dotenv>=1.0.0
   unstructured>=0.7.0
   chromadb>=0.4.0

docker-compose.yml
^^^^^^^^^^^^^^^^^
.. code-block:: yaml

   version: '3.8'

   services:
     asistente:
       build: .
       ports:
         - "8000:8000"
       volumes:
         - ./app/documentos:/app/documentos
         - ./app/vector_db:/app/vector_db
       environment:
         - DEEPSEEK_API_KEY=${DEEPSEEK_API_KEY}
       restart: unless-stopped

----------------------------
Paso 3: Construcción del Contenedor
----------------------------

1. Construir la imagen:

.. code-block:: bash

   docker-compose build

2. Verificar la creación:

.. code-block:: bash

   docker images | grep asistente

Salida esperada:
::

   REPOSITORY       TAG       IMAGE ID       CREATED         SIZE
   asistente_docker latest    abc123def456   2 minutes ago   1.2GB

----------------------------
Paso 4: Inicialización del Sistema
----------------------------

Primer arranque (carga inicial):
.. code-block:: bash

   # Iniciar el contenedor
   docker-compose up -d

   # Ejecutar la carga inicial de documentos
   docker exec -it asistente_docker python /app/main.py --cargar

Verificar logs:
.. code-block:: bash

   docker logs -f asistente_docker

----------------------------
Paso 5: Uso del Asistente
----------------------------

Consulta vía API::
.. code-block:: bash

   curl -X POST "http://localhost:8000/preguntar" \
   -H "Content-Type: application/json" \
   -d '{"pregunta": "¿Cómo configurar alertas en Zabbix?"}'

Respuesta esperada::
.. code-block:: json

   {
     "respuesta": "Según el manual Zabbix 7.2...",
     "fuentes": ["documentos/zabbix_manual.pdf"]
   }

----------------------------
Paso 6: Mantenimiento
----------------------------

Actualización de documentos:
1. Copiar nuevos archivos al directorio local:
.. code-block:: bash

   cp nuevo_documento.pdf ./app/documentos/

2. Reindexar contenido:
.. code-block:: bash

   docker exec asistente_docker python /app/main.py --actualizar

Monitorización:
.. code-block:: bash

   # Ver uso de recursos
   docker stats asistente_docker

   # Acceder al shell
   docker exec -it asistente_docker /bin/bash

--------------------------------
Configuraciones Avanzadas
--------------------------------

Optimización de la Imagen
^^^^^^^^^^^^^^^^^^^^^^^^^
Añadir al Dockerfile:
.. code-block:: dockerfile

   # Multi-stage build
   FROM python:3.10 as builder
   RUN pip install --user -r requirements.txt

   FROM python:3.10-slim
   COPY --from=builder /root/.local /root/.local
   ENV PATH=/root/.local/bin:$PATH

Variables de Entorno Seguras
^^^^^^^^^^^^^^^^^^^^^^^^^^^
1. Crear archivo ``.env`` en host:
.. code-block:: ini

   DEEPSEEK_API_KEY=tu_api_key_real

2. Modificar docker-compose.yml:
.. code-block:: yaml

   env_file:
     - .env

--------------------------------
Solución de Problemas Comunes
--------------------------------

+--------------------------------+-----------------------------------------------+
| Error                          | Solución                                      |
+================================+===============================================+
| "Address already in use"       | Cambiar puerto en docker-compose.yml          |
+--------------------------------+-----------------------------------------------+
| "ModuleNotFoundError"          | Reconstruir imagen con --no-cache             |
+--------------------------------+-----------------------------------------------+
| Permisos denegados en volumen  | Ejecutar: ``chmod -R 777 ./app/documentos``   |
+--------------------------------+-----------------------------------------------+
| "CUDA out of memory"           | Limitar uso de GPU en docker-compose.yml      |
+--------------------------------+-----------------------------------------------+


Comandos clave para gestión diaria:
----------------------------------

# Detener el contenedor::

   docker-compose down

# Actualizar la imagen (tras cambios en código)::

   docker-compose up -d --build

# Borrar cache de embeddings::

   docker exec asistente_docker rm -rf /app/vector_db/*

# Ver espacio utilizado::

   docker system df

Recomendaciones para producción:
---------------------------------------

Usar un proxy inverso (Nginx) delante del contenedor

Implementar autenticación JWT para la API

Configurar backups periódicos del volumen vector_db
