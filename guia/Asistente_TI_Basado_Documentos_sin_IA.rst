==================================================
Asistente de TI Basado en Documentos (Sin IA)
==================================================

**Versión estable** - Docker Container - Última actualización: 2024-05-20

Requisitos Previos
------------------
- Docker Engine 24.0+
- Docker Compose 2.20+
- Git 2.40+
- 4GB RAM disponibles

Estructura del Proyecto
-----------------------
::

    /asistente-ti/
    ├── docker-compose.yml
    ├── Dockerfile
    ├── documentos/          # Aquí van tus manuales TI
    ├── scripts/
    │   ├── indexador.sh     # Script de indexación
    │   └── buscador.sh      # Script de consultas
    └── data/                # Datos indexados

Paso 1: Configuración Docker
----------------------------

1. **Dockerfile**:

.. code-block:: dockerfile

    FROM ubuntu:22.04

    RUN apt-get update && \
        apt-get install -y \
        grep \
        awk \
        sed \
        python3 \
        python3-pip \
        && rm -rf /var/lib/apt/lists/*

    WORKDIR /app
    COPY . .

    RUN chmod +x scripts/*.sh

    CMD ["python3", "-m", "http.server", "8000"]

2. **docker-compose.yml**:

.. code-block:: yaml

    version: '3.8'

    services:
      asistente:
        build: .
        volumes:
          - ./documentos:/app/documentos
          - ./data:/app/data
          - ./scripts:/app/scripts
        ports:
          - "8000:8000"

Paso 2: Scripts de Procesamiento
-------------------------------

1. **scripts/indexador.sh**:

.. code-block:: bash

    #!/bin/bash

    DOCS_DIR="/app/documentos"
    INDEX_FILE="/app/data/indice.txt"

    echo "Iniciando indexación de documentos..."

    # Borrar índice anterior
    rm -f $INDEX_FILE

    # Indexar todos los documentos
    find $DOCS_DIR -type f \( -name "*.txt" -o -name "*.pdf" -o -name "*.md" \) | while read file; do
        echo "Procesando: $file"
        if [[ "$file" == *.pdf ]]; then
            pdftotext "$file" - | awk '{print FILENAME ": " $0}' >> $INDEX_FILE
        else
            awk '{print FILENAME ": " $0}' "$file" >> $INDEX_FILE
        fi
    done

    echo "Indexación completada. Total líneas: $(wc -l < $INDEX_FILE)"

2. **scripts/buscador.sh**:

.. code-block:: bash

    #!/bin/bash

    INDEX_FILE="/app/data/indice.txt"

    if [ ! -f "$INDEX_FILE" ]; then
        echo "Error: Primero debe ejecutar el indexador"
        exit 1
    fi

    echo "Sistema de Búsqueda Documental"
    echo "Ingrese su consulta: "
    read query

    echo "Resultados para: '$query'"
    echo "--------------------------------"

    grep -i "$query" $INDEX_FILE | head -n 10 | awk -F: '{
        print "Documento: " $1
        print "Contenido: " substr($0, index($0,$2))
        print "--------------------------------"
    }'

Paso 3: Instalación de Dependencias
-----------------------------------

1. Instalar herramientas adicionales en el contenedor (modificar Dockerfile):

.. code-block:: dockerfile

    RUN apt-get update && \
        apt-get install -y \
        poppler-utils \    # Para PDF
        git \
        wget \
        && rm -rf /var/lib/apt/lists/*

Paso 4: Despliegue
------------------

1. Construir el sistema:

.. code-block:: bash

    mkdir -p asistente-ti/{documentos,data,scripts}
    cd asistente-ti
    # Coloca tus documentos en la carpeta /documentos
    docker-compose build

2. Indexar documentos:

.. code-block:: bash

    docker-compose run asistente /app/scripts/indexador.sh

3. Iniciar el servicio:

.. code-block:: bash

    docker-compose up -d

4. Realizar consultas:

.. code-block:: bash

    docker-compose exec asistente /app/scripts/buscador.sh

Características Clave
---------------------
- ✅ **Sin componentes de IA** (solo búsqueda de texto)
- ✅ **Indexado automático** de documentos
- ✅ **Soporte para PDF, TXT y Markdown**
- ✅ **Interfaz de consulta por terminal**
- ✅ **Persistencia de datos entre sesiones**

Notas Técnicas
--------------
1. Para documentos PDF se requiere `poppler-utils`
2. El índice se almacena en `/app/data/indice.txt`
3. Las búsquedas son case-insensitive
4. Límite de 10 resultados por consulta

Actualización de Documentos
---------------------------
1. Detener el contenedor:

.. code-block:: bash

    docker-compose down

2. Actualizar documentos en `/documentos`
3. Reindexar:

.. code-block:: bash

    docker-compose run asistente /app/scripts/indexador.sh
    docker-compose up -d
