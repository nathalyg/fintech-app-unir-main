# Etapa 1: Instalación de dependencias
FROM node:18-alpine AS dependencies

WORKDIR /app

# Copiar archivos de dependencias
COPY package*.json ./

# Instalar dependencias de producción (sin devDependencies)
RUN npm install --omit=dev

# Etapa 2: Aplicación en ejecución
FROM node:18-alpine

WORKDIR /app

# Instalar curl para el healthcheck
RUN apk add --no-cache curl

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
