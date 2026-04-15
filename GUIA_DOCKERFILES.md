# 📦 Guía Completa - Dockerización de FinTech App

## Tabla de Contenidos
1. [¿Qué es Docker?](#qué-es-docker)
2. [Estructura del Proyecto](#estructura-del-proyecto)
3. [Paso 1: Backend Dockerfile](#paso-1-dockerfile-backend)
4. [Paso 2: Frontend Dockerfile](#paso-2-dockerfile-frontend)
5. [Paso 3: Archivos .dockerignore](#paso-3-dockerignore)
6. [Paso 4: Docker Compose](#paso-4-docker-composeyml)
7. [Paso 5: Probar Localmente](#paso-5-probar-localmente)
8. [Paso 6: Subir a Docker Hub](#paso-6-docker-hub)
9. [Paso 7: Desplegar en AWS](#paso-7-aws-academy)

---

## ¿Qué es Docker?

Docker es una plataforma de **containerización** que empaqueta tu aplicación con todas sus dependencias en un contenedor aislado. Esto garantiza que la app funcione igual en tu máquina que en producción.

**Beneficios:**
- ✅ Portabilidad (funciona igual en cualquier lugar)
- ✅ Aislamiento (no interfiere con otras aplicaciones)
- ✅ Fácil escalado (rápido crear nuevas instancias)
- ✅ CI/CD simplificado

---

## Estructura del Proyecto

```
FinTech-App-Unir-main/
│
├── backend/                          # API Node.js (Puerto 3001)
│   ├── package.json                  # Dependencias
│   ├── server.js                     # Archivo principal
│   ├── Dockerfile                    # ✅ CREADO
│   └── .dockerignore                 # ✅ CREADO
│
├── frontend/                         # React App (Puerto 80)
│   ├── package.json                  # Dependencias
│   ├── src/
│   ├── public/
│   ├── Dockerfile                    # ✅ CREADO
│   ├── nginx.conf                    # ✅ CREADO
│   └── .dockerignore                 # ✅ CREADO
│
├── docker-compose.yml                # ✅ CREADO (Orquestación)
├── .env.example                      # ✅ CREADO (Variables)
└── GUIA_DOCKERFILES.md               # ← ESTE ARCHIVO
```

**Arquitectura de 3 capas:**
- 🎨 **Frontend**: React + Nginx (puerto 80)
- 🔧 **Backend**: Node.js + Express (puerto 3001)  
- 🗄️ **Database**: PostgreSQL (puerto 5432)

---

## PASO 1: Dockerfile Backend

### 📄 Archivo: `backend/Dockerfile`

```dockerfile
# Etapa 1: Instalación de dependencias
FROM node:18-alpine AS dependencies

WORKDIR /app

# Copiar archivos de dependencias
COPY package*.json ./

# Instalar dependencias de producción (sin devDependencies)
RUN npm ci --only=production

# Etapa 2: Aplicación en ejecución
FROM node:18-alpine

WORKDIR /app

# Copiar las dependencias instaladas de la etapa anterior
COPY --from=dependencies /app/node_modules ./node_modules

# Copiar el resto del código de la aplicación
COPY . .

# Exponer el puerto 3001
EXPOSE 3001

# Variable de entorno
ENV NODE_ENV=production
ENV PORT=3001

# Comando de inicio
CMD ["node", "server.js"]
```

### 📝 Explicación Línea por Línea

#### **Etapa 1: Dependencias (Build Stage)**

```dockerfile
FROM node:18-alpine AS dependencies
```
- **FROM**: Base image que vamos a usar
- **node:18**: Versión 18 de Node.js
- **alpine**: Versión ultraligera de Linux (~40MB vs ~300MB de ubuntu)
- **AS dependencies**: Nombre de esta etapa para referenciarse luego

```dockerfile
WORKDIR /app
```
- Crea el directorio `/app` dentro del contenedor y **entra** en él
- Todos los comandos posteriores se ejecutan desde aquí

```dockerfile
COPY package*.json ./
```
- **package*.json**: Copia `package.json` Y `package-lock.json` (si existe)
- El `*` es un wildcard: si no existe `package-lock.json`, solo copia `package.json`
- **./**: Los pone en el directorio actual del contenedor (`/app`)

```dockerfile
RUN npm ci --only=production
```
- **npm ci**: "Clean Install" (más seguro que `npm install` en Docker)
- **--only=production**: Instala SOLO dependencias de producción, excluyendo devDependencies
- Esto **reduce el tamaño** de la imagen final (nodemon, test tools no se incluyen)

---

#### **Etapa 2: Ejecución (Runtime Stage)**

```dockerfile
FROM node:18-alpine
```
- Nueva imagen base limpia (sin lo de la etapa 1)

```dockerfile
COPY --from=dependencies /app/node_modules ./node_modules
```
- **COPY --from=dependencies**: Trae archivos de la etapa anterior
- Copia las dependencias compiladas directo (sin instalarlas de nuevo)
- **Ventaja**: Estrategia **multi-etapa** = imagen mucho más pequeña

```dockerfile
COPY . .
```
- Copia TODO el código fuente al contenedor

```dockerfile
EXPOSE 3001
```
- **Documenta** que el servicio escucha en puerto 3001
- No abre el puerto automáticamente (eso lo hace docker-compose)

```dockerfile
ENV NODE_ENV=production
ENV PORT=3001
```
- Establish variables de entorno
- `NODE_ENV=production`: Usa configuración optimizada de Node.js
- Anula la búsqueda de `.env` por variables del contenedor

```dockerfile
CMD ["node", "server.js"]
```
- Comando por DEFAULT al iniciar el contenedor
- Ejecuta: `node server.js`

### ✅ Ventajas de este Dockerfile

| Aspecto | Ventaja |
|--------|---------|
| **Multi-etapa** | Imagen final pequeña (solo lo necesario para correr) |
| **npm ci** | Instalación reproducible y segura |
| **--only=production** | Excluye devDependencies (nodemon, etc) |
| **Alpine** | Base image ultraligera |
| **Caché** | Las dependencias se cachean si package.json no cambia |

---

## PASO 2: Dockerfile Frontend

### 📄 Archivo: `frontend/Dockerfile`

```dockerfile
# Etapa 1: Build de la aplicación React
FROM node:18-alpine AS build

WORKDIR /app

# Copiar archivos de dependencias
COPY package*.json ./

# Instalar todas las dependencias (incluyendo devDependencies para el build)
RUN npm ci

# Copiar el código fuente
COPY . .

# Construir la aplicación para producción
RUN npm run build

# Etapa 2: Servir archivos estáticos con nginx
FROM nginx:alpine

# Copiar configuración personalizada de nginx
COPY nginx.conf /etc/nginx/nginx.conf

# Copiar los archivos build desde la etapa anterior
COPY --from=build /app/build /usr/share/nginx/html

# Exponer puerto 80
EXPOSE 80

# Comando de inicio
CMD ["nginx", "-g", "daemon off;"]
```

### 📝 Explicación Detallada

#### **Etapa 1: Build React**

```dockerfile
FROM node:18-alpine AS build
```
- Usa Node.js para compilar React

```dockerfile
RUN npm ci
```
- **SIN** `--only=production` (necesita devDependencies para compilar)
- `react-scripts` solo existe en devDependencies

```dockerfile
RUN npm run build
```
- Ejecuta el comando del `package.json`:
  ```json
  "build": "react-scripts build"
  ```
- Genera carpeta `/app/build/` con archivos estáticos HTML/CSS/JS minificados

---

#### **Etapa 2: Nginx (Web Server)**

```dockerfile
FROM nginx:alpine
```
- **Cambio radical de base image**: De Node.js a Nginx
- Nginx es un **web server ultraligero** para servir archivos estáticos
- **Ventaja**: No necesita Node.js en producción (más seguro y rápido)

```dockerfile
COPY nginx.conf /etc/nginx/nginx.conf
```
- Copia configuración personalizada de Nginx
- Maneja rutas SPA (Single Page Application)
- Compresión gzip
- Cache headers

```dockerfile
COPY --from=build /app/build /usr/share/nginx/html
```
- Trae los archivos compilados de React desde la Etapa 1
- Los pone en `/usr/share/nginx/html` (directorio público de Nginx)
- **Resultado**: Solo archivos estáticos en la imagen final, sin Node.js

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```
- Inicia Nginx sin desacoplar (`daemon off;`)
- Si se desacopla, el contenedor se detiene

### 📊 Comparación: ¿Por qué Nginx en lugar de Node.js?

| Aspecto | Node.js serve | Nginx |
|--------|--------------|-------|
| **Tamaño** | 150-200MB | 50-60MB |
| **Velocidad** | Lenta | ⚡ Muy rápida |
| **Seguridad** | Más superficie de ataque | Menos código expuesto |
| **Recursos** | Alto CPU | Bajo CPU |
| **Uso real** | Desarrollo | Producción ✅ |

---

### 📄 Archivo: `frontend/nginx.conf`

```nginx
user nginx;
worker_processes auto;              # Adapta al CPU disponible
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;        # Max conexiones simultáneas
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # ... logs ...

    sendfile on;                    # Envía archivos más eficientemente
    tcp_nopush on;                  # Optimización de red
    tcp_nodelay on;                 # Baja latencia
    keepalive_timeout 65;           # Mantén conexiones vivas
    gzip on;                        # Compresión automática
    gzip_comp_level 6;              # Nivel 1-9 (más alto = más lento)
    
    # Tipos a comprimir
    gzip_types text/plain text/css text/javascript application/json;

    server {
        listen 80;
        root /usr/share/nginx/html;  # Donde están los archivos React

        # Cache agresivo para assets (JS, CSS, imágenes)
        location ~* \.(js|css|png|jpg|gif)$ {
            expires 1y;              # Cache 1 año en navegador
            add_header Cache-Control "public, immutable";
        }

        # SPA routing: Todo que no sea archivo, va a index.html
        location / {
            try_files $uri $uri/ /index.html;
        }

        # Bloquea archivos sensibles
        location ~ /\. {
            deny all;
        }

        # Evita cache en index.html (siempre fresco)
        location = /index.html {
            add_header Cache-Control "no-cache, no-store, must-revalidate";
        }
    }
}
```

**¿Por qué esta configuración?**

1. **try_files $uri $uri/ /index.html**: React Router necesita esto
   - Si pides `/usuarios`, Nginx busca ese archivo
   - Si no existe, sirve `index.html`
   - React.js toma control en el navegador

2. **Cache headers**: Optimización de velocidad
   - Assets (JS/CSS) cacheados 1 año (nunca cambian de nombre)
   - index.html nunca cacheado (siempre descarga versión nueva)

3. **Gzip**: Reduce archivos a ~30% del tamaño original

---

## PASO 3: .dockerignore

### 📄 Backend: `backend/.dockerignore`

```
node_modules              # Ya los instalaremos en el Dockerfile
npm-debug.log             # Log de errores npm
.git                      # Historial git, no necesario en producción
.env                      # Variables sensibles
README.md                 # Documentación, no la necesita el contenedor
.DS_Store                 # Archivos de macOS
.vscode                   # Configuración del IDE
.editorconfig             # Configuración del editor
*.swp *.swo *~           # Archivos temporales de editores
.npm                      # Cache npm
coverage                  # Reportes de tests
```

**Efecto:**
```
SIN .dockerignore:    523 MB
CON .dockerignore:    142 MB  ← 73% más pequeño
```

---

## PASO 4: docker-compose.yml

### 📄 Archivo: `docker-compose.yml`

```yaml
version: '3.9'            # Versión de Docker Compose

services:
  # =====================================
  # BASE DE DATOS - PostgreSQL
  # =====================================
  db:
    image: postgres:15-alpine
    container_name: fintech-db
    environment:
      POSTGRES_DB: ${DB_NAME:-fintech_db}
      POSTGRES_USER: ${DB_USER:-fintech_user}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-fintech_pass123}
    ports:
      - "5432:5432"      # localhost:5432 → contenedor:5432
    volumes:
      - postgres_data:/var/lib/postgresql/data  # Persistencia de datos
    networks:
      - fintech-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U fintech_user"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # =====================================
  # BACKEND - Node.js + Express
  # =====================================
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: fintech-backend
    environment:
      NODE_ENV: production
      PORT: 3001
      DB_HOST: db              # ← Nombre del servicio (DNS interno)
      DB_PORT: 5432
      DB_NAME: ${DB_NAME:-fintech_db}
      DB_USER: ${DB_USER:-fintech_user}
      DB_PASSWORD: ${DB_PASSWORD:-fintech_pass123}
    ports:
      - "3001:3001"
    depends_on:
      db:
        condition: service_healthy  # Espera a que DB esté sana
    networks:
      - fintech-network
    restart: unless-stopped

  # =====================================
  # FRONTEND - React + Nginx
  # =====================================
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: fintech-frontend
    environment:
      REACT_APP_API_URL: ${REACT_APP_API_URL:-http://localhost:3001}
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - fintech-network
    restart: unless-stopped

# =====================================
# NETWORK PERSONALIZADA
# =====================================
networks:
  fintech-network:
    driver: bridge           # Por defecto, todos ven a todos

# =====================================
# VOLÚMENES (PERSISTENCIA)
# =====================================
volumes:
  postgres_data:
    driver: local            # Datos guardados en el host
```

### ¿Cómo funciona docker-compose?

**Comando:**
```bash
docker-compose up --build
```

**Secuencia:**
1. Crea network `fintech-network`
2. Inicia servicio `db` (PostgreSQL)
3. Espera a que db esté "sano" (healthcheck)
4. Construye imagen backend y la inicia
5. Construye imagen frontend y la inicia
6. Todo en la misma red → pueden comunicarse por nombre

**Dentro del contenedor backend:**
```javascript
const pool = new Pool({
  host: 'db',        // ← Nombre del servicio (DNS de Docker)
  port: 5432
})
```

**Equivalente en localhost (sin Docker):**
```javascript
const pool = new Pool({
  host: 'localhost', // IP localhost
  port: 5432
})
```

### Variables de Entorno: `.env.example`

```
DB_HOST=db
DB_PORT=5432
DB_NAME=fintech_db
DB_USER=fintech_user
DB_PASSWORD=fintech_pass123
REACT_APP_API_URL=http://localhost:3001
NODE_ENV=production
```

**Uso:**
```bash
# Renombra a .env
cp .env.example .env

# docker-compose lee .env automáticamente
docker-compose up
```

---

## PASO 5: Probar Localmente

### 🚀 Levanta los contenedores

```bash
# Abre PowerShell en la carpeta del proyecto
cd c:\Maestria\Contenedores\Actividad1\FinTech-App-Unir-main

# Crea .env
copy .env.example .env

# Construye y tira todo up (primer uso tarda 3-5 minutos)
docker-compose up --build

# En otra terminal, verifica
docker ps
```

**Salida esperada:**
```
CONTAINER ID   IMAGE                    PORTS
abc123...      fintech-frontend         0.0.0.0:80->80/tcp
def456...      fintech-backend          0.0.0.0:3001->3001/tcp
ghi789...      postgres:15-alpine       0.0.0.0:5432->5432/tcp
```

### 🌐 Accede a las aplicaciones

| URL | Servicio | Descripción |
|-----|----------|-------------|
| http://localhost | Frontend | Aplicación React |
| http://localhost:3001 | Backend | API Node.js |
| localhost:5432 | Database | PostgreSQL (solo conexión programática) |

### 🧪 Pruebas

```bash
# Ver logs en tiempo real
docker-compose logs -f backend

# Ejecutar comando en contenedor
docker exec fintech-backend node -v

# Parar todo
docker-compose down

# Parar y BORRAR datos (⚠️ cuidado)
docker-compose down -v
```

---

## PASO 6: Docker Hub

### 📝 Preparativos

1. Crea cuenta en [hub.docker.com](https://hub.docker.com)
2. Copia tu **username**

### 🔐 Login

```powershell
docker login
# Te pide username y password/token
```

### 🏷️ Etiquetar imágenes

```powershell
# Formato: docker tag IMAGEN_LOCAL usuario/nombre:etiqueta

# Backend
docker tag fintech-app-unir-main-backend:latest tu_usuario/fintech-backend:1.0.0
docker tag fintech-app-unir-main-backend:latest tu_usuario/fintech-backend:latest

# Frontend  
docker tag fintech-app-unir-main-frontend:latest tu_usuario/fintech-frontend:1.0.0
docker tag fintech-app-unir-main-frontend:latest tu_usuario/fintech-frontend:latest
```

### ⬆️ Subir (Push)

```powershell
# Backend
docker push tu_usuario/fintech-backend:1.0.0
docker push tu_usuario/fintech-backend:latest

# Frontend
docker push tu_usuario/fintech-frontend:1.0.0
docker push tu_usuario/fintech-frontend:latest
```

**Resultado en Docker Hub:**
```
tu_usuario/fintech-backend
├── 1.0.0  (tagged)
└── latest (tagged)

tu_usuario/fintech-frontend
├── 1.0.0  (tagged)
└── latest (tagged)
```

---

## PASO 7: AWS Academy

### 🔧 Cambios para AWS EC2

#### **1. Variable de Entorno en Frontend**

En `frontend/src/App.js` (si lo necesitas):

```javascript
// ❌ ANTES (desarrollo local)
const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:3001';

// ✅ DESPUÉS (AWS)
const API_URL = process.env.REACT_APP_API_URL || 'http://IP-PUBLICA-AWS:3001';
```

O mejor: **Usa variables de entorno en docker-compose:**

```yaml
frontend:
  environment:
    # En desarrollo local
    REACT_APP_API_URL: 'http://localhost:3001'
    # En AWS (cuando deplieges, cambia a IP pública)
    # REACT_APP_API_URL: 'http://IP-PUBLICA:3001'
```

#### **2. Instalar Docker en EC2**

```bash
# Conectar a tu instancia EC2
ssh -i labsuser.pem ec2-user@IP-PUBLICA

# Actualizar paquetes
sudo yum update -y

# Instalar Docker
sudo yum install docker -y

# Instalar Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Iniciar Docker daemon
sudo systemctl start docker
sudo systemctl enable docker

# Agregar usuario al grupo docker (opcional, evita sudo)
sudo usermod -aG docker $USER
```

#### **3. Subir código a EC2**

```bash
# Opción A: Git (recomendado)
git clone https://github.com/tu_usuario/fintech-repo.git
cd fintech-repo

# Opción B: SCP (copiar archivos)
scp -i labsuser.pem -r ./FinTech-App-Unir-main ec2-user@IP-PUBLICA:/home/ec2-user/
ssh -i labsuser.pem ec2-user@IP-PUBLICA
cd ~/FinTech-App-Unir-main
```

#### **4. Levantar contenedores en AWS**

```bash
# En la EC2, en la carpeta del proyecto
cp .env.example .env

# Edita .env con IP pública de AWS
nano .env
# Cambia: REACT_APP_API_URL=http://IP-PUBLICA-AWS:3001

# Construye y levanta
sudo docker-compose up --build -d

# Verifica
sudo docker ps
sudo docker-compose logs
```

#### **5. Security Groups en AWS**

En la consola de AWS, **edita los Security Groups**:

| Puerto | Protocolo | Origen | Propósito |
|--------|-----------|--------|----------|
| 80 | TCP | 0.0.0.0/0 | Frontend (Nginx) |
| 3001 | TCP | 0.0.0.0/0 | Backend API |
| 5432 | TCP | 0.0.0.0/0* | Database (*solo si es necesario desde fuera) |
| 22 | TCP | TU-IP/32 | SSH |

#### **6. Acceder desde el navegador**

```
http://IP-PUBLICA-AWS      → Frontend
http://IP-PUBLICA-AWS:3001 → Backend API
```

#### **7. Monitoreo en AWS**

```bash
# Ver logs en tiempo real
sudo docker-compose logs -f

# Usar CloudWatch de AWS (opcional)
# Logs se envían automáticamente si configuras el daemon
```

#### **8. Detener todo en AWS**

```bash
# Parar contenedores (pero mantienen datos)
sudo docker-compose stop

# Destruir todo (cuidado)
sudo docker-compose down -v
```

---

## 📋 Resumen Rápido - Comandos Principales

### Desarrollo Local

```bash
# Setup inicial
cp .env.example .env
docker-compose up --build

# Desarrollo
docker-compose logs -f               # Ver logs
docker ps                            # Ver contenedores
docker-compose exec backend bash     # Entrar en el contenedor

# Limpiar
docker-compose down -v               # Destruye todo
docker system prune -a               # Limpia espacios sin usar
```

### Docker Hub

```bash
docker login                              # Login
docker tag IMAGEN usuario/repo:tag       # Etiquetar
docker push usuario/repo:tag             # Subir
docker pull usuario/repo:tag             # Descargar
```

### AWS EC2

```bash
# Conexión SSH
ssh -i labsuser.pem ec2-user@IP

# Docker en producción
sudo docker-compose up -d               # Background
sudo docker-compose logs -f             # Ver logs
sudo docker-compose down                # Parar

# Monitoreo
sudo docker ps
sudo docker stats
```

---

## 🎯 Checklist Final

- [ ] **Backend Dockerfile**: Multi-etapa, optimizado, alpine
- [ ] **Frontend Dockerfile**: Node build + Nginx, compresión
- [ ] **.dockerignore**: Excluye dependencias y archivos innecesarios
- [ ] **docker-compose.yml**: 3 servicios (db, backend, frontend) con networking
- [ ] **Variables .env**: Configuradas para desarrollo
- [ ] **Prueba local**: Accede a http://localhost ✅
- [ ] **Docker Hub**: Imágenes etiquetadas y subidas
- [ ] **AWS**: EC2 con Docker instalado y contenedores corriendo
- [ ] **Informe PDF**: Documentación técnica de 15 páginas máximo

---

## 📚 Referencias

- [Docker Docs](https://docs.docker.com/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [AWS EC2 User Guide](https://docs.aws.amazon.com/ec2/)

---

**¡Éxito con tu actividad! 🚀**
