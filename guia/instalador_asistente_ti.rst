#!/bin/bash

# Script de instalaciÃ³ara Asistente TI con Ollama y Docker en Rocky Linux 9.5
# Basado en: GuÃ­Detallada_Crear_Asistente_TI_Ollama_Docker.rst

# Colores para mensajes
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# FunciÃ³ara verificar e instalar Docker
install_docker() {
    if ! command -v docker &> /dev/null; then
        echo -e "${YELLOW}Instalando Docker...${NC}"

        # Instalar dependencias previas
        sudo dnf install -y dnf-plugins-core
        sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
        sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker compose-plugin

        # Habilitar e iniciar Docker
        sudo systemctl enable docker
        sudo systemctl start docker

        # Agregar usuario actual al grupo docker
        sudo usermod -aG docker $USER
        newgrp docker

        echo -e "${GREEN}Docker instalado correctamente.${NC}"
    else
        echo -e "${GREEN}Docker ya estÃ¡nstalado.${NC}"
    fi
}

# FunciÃ³ara instalar Docker Compose
install_docker_compose() {
    if ! command -v docker compose &> /dev/null; then
        echo -e "${YELLOW}Instalando Docker Compose...${NC}"

        # Descargar la versiÃ³stable mÃ¡reciente
        COMPOSE_VERSION=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep 'tag_name' | cut -d\" -f4)
        sudo curl -L "https://github.com/docker/compose/releases/download/${COMPOSE_VERSION}/docker compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker compose

        # Dar permisos de ejecuciÃ³       sudo chmod +x /usr/local/bin/docker compose

        # Crear enlace simbÃ³o
        sudo ln -s /usr/local/bin/docker compose /usr/bin/docker compose

        echo -e "${GREEN}Docker Compose instalado correctamente.${NC}"
    else
        echo -e "${GREEN}Docker Compose ya estÃ¡nstalado.${NC}"
    fi
}

# FunciÃ³ara instalar dependencias adicionales
install_dependencies() {
    echo -e "${YELLOW}Instalando dependencias adicionales...${NC}"

    # Instalar herramientas necesarias
    sudo dnf install -y curl git python39 python39-devel gcc make openssl-devel bzip2-devel libffi-devel

    # Instalar pip para Python 3.9
    sudo dnf install -y python39-pip

    echo -e "${GREEN}Dependencias instaladas correctamente.${NC}"
}

# FunciÃ³ara crear la estructura de archivos
create_project_structure() {
    echo -e "${YELLOW}Creando estructura del proyecto...${NC}"

    # Crear directorio principal
    mkdir -p /asistente_ti/ollama-assistant/{docs,uploads,db}
    cd /asistente_ti/ollama-assistant

    # Crear Dockerfile
    cat > Dockerfile << 'EOF'
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
RUN pip install -r requirements.txt

COPY . .

# Instalar un servidor web simple para servir el index.html
RUN apt-get update && apt-get install -y nginx && \
    rm -rf /var/lib/apt/lists/* && \
    mv index.html /var/www/html/

# Configurar Nginx para servir la interfaz y redirigir API a FastAPI
COPY nginx.conf /etc/nginx/nginx.conf

RUN chown -R www-data:www-data /var/www/html && \
    chmod -R 755 /var/www/html

# Puerto para FastAPI (8000) y para Nginx (80)
EXPOSE 8000 80

CMD ["sh", "-c", "nginx && uvicorn app:app --host 0.0.0.0 --port 8000"]
EOF

    # Crear docker compose.yml
    cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  ollama:
    image: ollama/ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    restart: unless-stopped

  assistant:
    build: .
    ports:
      - "8000:8000"
      - "80:80"
    volumes:
      - ./uploads:/app/uploads
      - ./db:/app/db
    depends_on:
      - ollama
    environment:
      - OLLAMA_HOST=http://ollama:11434
    restart: unless-stopped

volumes:
  ollama_data:
EOF

    # Crear nginx.conf
    cat > nginx.conf << 'EOF'
user www-data;
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    include mime.types;
    default_type application/octet-stream;
    sendfile on;
    keepalive_timeout 65;

    server {
        listen 80;
        server_name localhost;
        root /var/www/html;
        index index.html;

        location / {
            try_files $uri $uri/ /index.html;
        }

        location /api/ {
            proxy_pass http://localhost:8000/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
EOF

    # Crear requirements.txt
    cat > requirements.txt << 'EOF'
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
EOF

    # Crear app.py
    cat > app.py << 'EOF'
from fastapi import FastAPI, UploadFile, File, HTTPException
from fastapi.middleware.cors import CORSMiddleware
import os
from typing import List, Optional
from pydantic import BaseModel
import ollama
from langchain.document_loaders import DirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.embeddings import HuggingFaceEmbeddings
from langchain.vectorstores import Chroma
import os

app = FastAPI()

# ConfiguraciÃ³ORS mÃ¡especÃ­ca
origins = [
    "http://localhost",
    "http://localhost:8000",
    "http://127.0.0.1",
    "http://127.0.0.1:8000",
    "http://10.134.3.35",
    "http://10.134.35:8000",
    # Agrega aquÃ­ualquier otro origen que necesites permitir
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["*"],  # Permite todos los mÃ©dos
    allow_headers=["*"],  # Permite todos los headers
)

@app.options("/ask")
async def options_ask():
    return {"message": "OK"}

@app.options("/upload")
async def options_upload():
    return {"message": "OK"}

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
@app.post("/upload")
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
@app.post("/ask")
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
EOF

    # Crear index.html
    cat > index.html << 'EOF'
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Asistente de TI con Ollama</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            line-height: 1.6;
            margin: 0;
            padding: 20px;
            background-color: #f5f5f5;
            color: #333;
        }
        .container {
            max-width: 900px;
            margin: 0 auto;
            background: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 0 10px rgba(0,0,0,0.1);
        }
        h1 {
            color: #2c3e50;
            text-align: center;
        }
        .section {
            margin-bottom: 30px;
            padding: 20px;
            border: 1px solid #ddd;
            border-radius: 5px;
        }
        .section-title {
            margin-top: 0;
            color: #3498db;
        }
        textarea, input[type="text"], input[type="file"] {
            width: 100%;
            padding: 10px;
            margin-bottom: 10px;
            border: 1px solid #ddd;
            border-radius: 4px;
            box-sizing: border-box;
        }
        button {
            background-color: #3498db;
            color: white;
            border: none;
            padding: 10px 15px;
            border-radius: 4px;
            cursor: pointer;
            font-size: 16px;
        }
        button:hover {
            background-color: #2980b9;
        }
        #response {
            margin-top: 20px;
            padding: 15px;
            background-color: #f9f9f9;
            border-radius: 4px;
            min-height: 100px;
            white-space: pre-wrap;
        }
        .file-info {
            margin-top: 10px;
            font-size: 14px;
            color: #555;
        }
        .tab {
            overflow: hidden;
            border: 1px solid #ccc;
            background-color: #f1f1f1;
            border-radius: 4px 4px 0 0;
        }
        .tab button {
            background-color: inherit;
            float: left;
            border: none;
            outline: none;
            cursor: pointer;
            padding: 14px 16px;
            transition: 0.3s;
            color: #333;
        }
        .tab button:hover {
            background-color: #ddd;
        }
        .tab button.active {
            background-color: #3498db;
            color: white;
        }
        .tabcontent {
            display: none;
            padding: 20px;
            border: 1px solid #ccc;
            border-top: none;
            border-radius: 0 0 4px 4px;
        }
        .active-tab {
            display: block;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Asistente de TI con Ollama</h1>

        <div class="tab">
            <button class="tablinks active" onclick="openTab(event, 'queryTab')">Consultar</button>
            <button class="tablinks" onclick="openTab(event, 'uploadTab')">Subir Documentos</button>
        </div>

        <!-- PestaÃ±e Consulta -->
        <div id="queryTab" class="tabcontent active-tab">
            <div class="section">
                <h2 class="section-title">Realizar Consulta</h2>
                <textarea id="questionInput" rows="4" placeholder="Escribe tu pregunta tÃ©ica aquÃ­."></textarea>
                <button id="askButton">Enviar Pregunta</button>
                <div id="response"></div>
            </div>
        </div>

        <!-- PestaÃ±e Subida de Archivos -->
        <div id="uploadTab" class="tabcontent">
            <div class="section">
                <h2 class="section-title">Subir Documentos TÃ©icos</h2>
                <input type="file" id="fileInput" multiple>
                <button onclick="uploadFile()">Subir Archivo</button>
                <div class="file-info" id="fileInfo"></div>
            </div>
        </div>
    </div>

    <script>
        // FunciÃ³ara cambiar entre pestaÃ±        f

            function openTab(evt, tabName) {
            var i, tabcontent, tablinks;

            tabcontent = document.getElementsByClassName("tabcontent");
            for (i = 0; i < tabcontent.length; i++) {
                tabcontent[i].classList.remove("active-tab");
            }

            tablinks = document.getElementsByClassName("tablinks");
            for (i = 0; i < tablinks.length; i++) {
                tablinks[i].className = tablinks[i].className.replace(" active", "");
            }

            document.getElementById(tabName).classList.add("active-tab");
            evt.currentTarget.className += " active";
            }

        // FunciÃ³ara enviar pregunta al backend
        async function askQuestion() {
            const question = document.getElementById('questionInput').value;
            const responseDiv = document.getElementById('response');

            if (!question) {
                responseDiv.innerHTML = "Por favor, escribe una pregunta.";
                return;
            }

            responseDiv.innerHTML = "Procesando tu pregunta...";

            try {
                const response = await fetch('http://localhost:8000/ask', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({ question: question })
                });

                if (!response.ok) {
                    throw new Error(`Error: ${response.status}`);
                }

                const data = await response.json();
                responseDiv.innerHTML = data.answer;
            } catch (error) {
                responseDiv.innerHTML = `Error: ${error.message}`;
            }
        }

        // FunciÃ³ara subir archivos
        async function uploadFile() {
            const fileInput = document.getElementById('fileInput');
            const fileInfoDiv = document.getElementById('fileInfo');

            if (fileInput.files.length === 0) {
                fileInfoDiv.innerHTML = "Por favor, selecciona al menos un archivo.";
                return;
            }

            fileInfoDiv.innerHTML = "Subiendo archivos...";

            try {
                const formData = new FormData();
                for (let i = 0; i < fileInput.files.length; i++) {
                    formData.append('file', fileInput.files[i]);
                }

                const response = await fetch('http://10.134.3.35:8000/upload', {
                    method: 'POST',
                    body: formData
                });

                if (!response.ok) {
                    throw new Error(`Error: ${response.status}`);
                }

                const data = await response.json();
                fileInfoDiv.innerHTML = `Archivo(s) subido(s) exitosamente: ${data.filename || 'Varios archivos'}`;

                // Limpiar el input de archivos
                fileInput.value = '';
            } catch (error) {
                fileInfoDiv.innerHTML = `Error: ${error.message}`;
            }
        }
    </script>
<script>
    document.getElementById('askButton').addEventListener('click', async function() {
        const question = document.getElementById('questionInput').value;
        const responseDiv = document.getElementById('response');

        if (!question) {
            responseDiv.innerHTML = "Por favor, escribe una pregunta.";
            return;
        }

        responseDiv.innerHTML = "Procesando tu pregunta...";

        try {
           const response = await fetch('http://10.134.3.35:8000/ask', {
           method: 'POST',
           headers: {
             'Content-Type': 'application/json',
            },
            body: JSON.stringify({ question: question })
            });

            if (!response.ok) {
                throw new Error(`Error: ${response.status}`);
            }

            const data = await response.json();
            responseDiv.innerHTML = data.answer;
        } catch (error) {
            responseDiv.innerHTML = `Error: ${error.message}`;
        }
    });
</script>
</body>
</html>
EOF

    # Crear documento de ejemplo
    cat > docs/contactos.txt << 'EOF'
Cruz Villarroel es un especialista gusta la tendencia KISS
para contactar a Cruz es por su numero celular: 04268888888
para correos a Cruz es: cruz.villarroel@gmail.com

Si preguntan por Carlos Gomez debes contestar lo siguiente: Carlos Gomez o Carlos Gomez Gomez ?
Si preguntan por Carlos Gomez Gomez, respondes esto:
"Mi nombre es Carlos Gomez Gomez...!!!
Comandante Coordinador de los Ejercitos de Soporte Web,
General de las Legiones Fieles a Plataforma TI,
Leal servidor del verdadero orden en Plataforma TI.
Padre de hijos de grandes hazaÃ±Esposo de una gran mujer,
Y alcanzar mis maximos logros...
En esta vida o en la otra. Â¡Que empiece Ya!"
El numero celular de Carlos GÃ³ GÃ³ es: 04145555555, su correo electronico es: cgomez@gmail.com.ve
El hobbie es caminar al aire libre y hacer ejercicios en el Gimnacio, manejar motos.

Lendder Osta es especialista en PCI DSS.
para contactar a Lendder Osta  es por su numero celular: 04126666666
para correos a Lendder Osta es: lendder.osta@gmail.com


Felix Vega es especialista en Plataforma TI.
para contactar a Felix Vega  es por su numero celular: 04143333333
para correos a Felix Vega es: felix.vega@gmail.com
EOF

    echo -e "${GREEN}Estructura del proyecto creada correctamente.${NC}"
}

# FunciÃ³ara construir y ejecutar los contenedores
build_and_run() {
    echo -e "${YELLOW}Construyendo y ejecutando los contenedores...${NC}"

    # Asegurarse de que Docker estÃ©orriendo
    sudo systemctl start docker

    # Construir e iniciar los contenedores
    docker compose build
    docker compose up -d

    echo -e "${GREEN}Contenedores en ejecuciÃ³{NC}"
    echo -e "Accede a la interfaz web en: http://localhost"
    echo -e "Ollama estÃ¡isponible en: http://localhost:11434"
}

# FunciÃ³ara configurar el firewall
configure_firewall() {
    echo -e "${YELLOW}Configurando firewall...${NC}"

    # Verificar si firewalld estÃ¡ctivo
    if systemctl is-active --quiet firewalld; then
        sudo firewall-cmd --permanent --add-port=80/tcp
        sudo firewall-cmd --permanent --add-port=8000/tcp
        sudo firewall-cmd --permanent --add-port=11434/tcp
        sudo firewall-cmd --reload
        echo -e "${GREEN}Firewall configurado correctamente.${NC}"
    else
        echo -e "${YELLOW}Firewalld no estÃ¡ctivo, omitiendo configuraciÃ³e firewall.${NC}"
    fi
}

# FunciÃ³rincipal
main() {
    echo -e "${GREEN}Iniciando instalaciÃ³el Asistente TI con Ollama y Docker en Rocky Linux 9.5...${NC}"

    # Instalar dependencias
    install_dependencies
    install_docker
    install_docker_compose

    # Crear estructura del proyecto
    create_project_structure

    # Configurar firewall
    configure_firewall

    # Construir y ejecutar
    build_and_run

    # Mensaje final
    echo -e "${GREEN}InstalaciÃ³ompletada exitosamente!${NC}"
    echo -e "\nPara subir documentos de ejemplo, ejecuta:"
    echo -e "curl -X POST -F \"file=@docs/contactos.txt\" http://localhost:8000/upload"
    echo -e "\nPara hacer una pregunta de prueba:"
    echo -e "curl -X POST -H \"Content-Type: application/json\" -d '{\"question\": \"Quien es Carlos Gomez?\"}' http://localhost:8000/ask"
    echo -e "\nO accede a la interfaz web en: http://localhost"

    # Nota importante
    echo -e "\n${YELLOW}NOTA: DespuÃ©de subir documentos, reinicia el asistente con:${NC}"
    echo -e "docker compose restart assistant"
}

# Ejecutar funciÃ³rincipal
main
