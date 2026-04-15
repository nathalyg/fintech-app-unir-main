## FinTech App - Unir 

### Consideraciones para el backend (puerto 3001)
```
  Analizar el package.json antes de crear el Dockerfile.
  Identificar con precisión el comando de arranque de la aplicación.
  Comprobar si el backend necesita únicamente ejecución o también una fase previa de compilación.
  Verificar el puerto real utilizado por el servicio.
  Seleccionar una imagen base adecuada para Node.js.
  Organizar las instrucciones del Dockerfile de forma lógica para aprovechar la caché de Docker.
  Copiar primero los archivos de dependencias y después el resto del código.
  Revisar si existe package-lock.json y tener en cuenta su impacto en la instalación de dependencias.
  Evitar incluir en la imagen archivos o carpetas innecesarios.
  Proponer un .dockerignore coherente con el proyecto.
  Diferenciar entre dependencias necesarias para desarrollo y para ejecución.
  Justificar técnicamente cada instrucción incluida en el Dockerfile.
```
### Consideraciones para el frontend (puerto 80)
```
  Analizar el package.json antes de crear el Dockerfile.
  Identificar el comando exacto de construcción del frontend.
  Comprobar qué carpeta o artefacto genera el proceso de build.
  Determinar si el frontend produce archivos estáticos o si necesita un servidor de ejecución.
  Seleccionar la imagen base o las imágenes base adecuadas según la estrategia elegida.
  Valorar si la solución requiere una sola etapa o varias etapas.
  Organizar las instrucciones del Dockerfile para aprovechar la caché de Docker.
  Copiar primero los archivos de dependencias y después el resto del código.
  Revisar si existe package-lock.json y considerar su impacto en la instalación de dependencias.
  Tener en cuenta posibles variables de entorno necesarias en build o en runtime.
  Evitar incluir en la imagen archivos o carpetas innecesarios.
  Proponer un .dockerignore coherente con el proyecto.
  Distinguir entre los elementos necesarios para construir la aplicación y los necesarios para ejecutarla.
  Justificar técnicamente cada decisión adoptada en el Dockerfile.
```
## Consideraciones en AWS
```
Si lo lanzas en AWS en la llamada a la api del app.js hay que cambiar
const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:3001';
const API_URL = process.env.REACT_APP_API_URL || 'http://IP-PUBLICA:3001';
```

