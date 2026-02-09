# Prompts del Proyecto

## Pipeline CI/CD con GitHub Actions y AWS EC2

### Contexto del Proyecto
Este es un proyecto full-stack con:
- **Backend**: Node.js + TypeScript ubicado en el directorio `backend/`
  - Framework de testing: Jest (configurado en `jest.config.js`)
  - Base de datos: PostgreSQL con Prisma ORM
  - Build configurado en `package.json` del backend
  - Tests ubicados en `src/application/services/*.test.ts` y `src/presentation/controllers/*.test.ts`

- **Estructura del repositorio**:
  ```
  /backend
    /src
    /prisma
    package.json
    jest.config.js
    tsconfig.json
  ```

### Objetivo
Crear un archivo de configuración de GitHub Actions ubicado en `.github/workflows/pipeline.yml` que automatice el proceso de CI/CD para el backend del proyecto.

### Requisitos del Pipeline

#### 1. Trigger del Workflow
El pipeline debe dispararse **únicamente cuando se realiza un push a una rama que tiene un Pull Request abierto** hacia la rama principal. Esto se puede lograr usando:
```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]
```

#### 2. Jobs y Steps del Pipeline

El workflow debe ejecutar las siguientes fases en orden secuencial:

##### Fase 1: Testing
- Configurar el entorno de Node.js (versión 18.x o superior)
- Navegar al directorio `backend/`
- Instalar las dependencias con `npm ci` (instalación limpia más rápida que `npm install`)
- Ejecutar los tests del backend con `npm test`
- El pipeline debe **fallar si algún test no pasa**

##### Fase 2: Build
- Esta fase solo debe ejecutarse si los tests pasan correctamente
- Navegar al directorio `backend/`
- Generar el build de producción del backend con `npm run build`
- El build debe generar los archivos compilados (verificar que la carpeta de distribución se cree correctamente)

##### Fase 3: Deployment a AWS EC2
- Esta fase solo debe ejecutarse si el build es exitoso
- Desplegar el código del backend a una instancia de AWS EC2
- Utilizar la acción `easingthemes/ssh-deploy@main` para el despliegue vía SSH
- Configurar las siguientes variables de entorno usando los secrets de GitHub:
  - `SSH_PRIVATE_KEY`: `${{ secrets.EC2_SSH_KEY }}` - Clave SSH (.pem) para autenticación
  - `REMOTE_HOST`: `${{ secrets.HOST_DNS }}` - DNS público de la instancia EC2
  - `REMOTE_USER`: `${{ secrets.USERNAME }}` - Usuario del servidor (ej: ubuntu)
  - `TARGET`: `${{ secrets.TARGET_DIR }}` - Directorio destino en el EC2
- Configurar variables de entorno en el servidor:
  - Crear archivo `.env` en el directorio de destino con:
    - `NODE_ENV=production`
    - `DATABASE_URL=${{ secrets.DATABASE_URL }}` - URL de conexión a PostgreSQL en AWS
    - `PORT=3000` u otras variables necesarias
- Después del despliegue, ejecutar comandos post-deploy en el servidor:
  - Instalar dependencias de producción: `npm ci --production`
  - Generar cliente de Prisma: `npx prisma generate`
  - Ejecutar migraciones de Prisma: `npx prisma migrate deploy`
  - Reiniciar el servicio de la aplicación (usando PM2 o el gestor de procesos configurado)

### Configuración de Secrets ya Disponibles
Los siguientes secrets ya están configurados en el repositorio de GitHub:
- `EC2_SSH_KEY`: Clave privada SSH para conectar al EC2
- `HOST_DNS`: DNS público de la instancia EC2 (formato: ec2-xx-xxx-xxx-xxx.region.compute.amazonaws.com)
- `TARGET_DIR`: Directorio de destino en el servidor EC2 donde se desplegará la aplicación
- `USERNAME`: Usuario SSH del servidor EC2
- `DATABASE_URL`: URL de conexión a la base de datos PostgreSQL en AWS RDS (formato: postgresql://user:password@host:port/database)

### Requisitos Técnicos Adicionales

1. **Contexto de ejecución**: Los jobs deben ejecutarse en `ubuntu-latest`

2. **Optimización**:
   - Usar `actions/checkout@v3` o superior para checkout del código
   - Usar `actions/setup-node@v3` para configurar Node.js
   - Implementar caché de dependencias de npm para acelerar el proceso
   - Especificar el directorio de trabajo como `backend/` para evitar navegación repetitiva

3. **Logging y Visibilidad**:
   - Cada step debe tener un nombre descriptivo
   - Los jobs deben tener nombres claros que indiquen su propósito

4. **Manejo de Errores**:
   - El workflow debe fallar inmediatamente si cualquier step falla
   - No continuar con deployment si tests o build fallan

5. **Source de Archivos**:
   - Solo desplegar los archivos necesarios (dist/, node_modules de producción, package.json, prisma/)
   - Excluir archivos de desarrollo y tests

### Consideraciones de Seguridad
- Nunca exponer las claves o secretos en logs
- Usar únicamente GitHub Secrets para información sensible
- La comunicación con EC2 debe ser exclusivamente por SSH

### Referencias y Documentación
- Documentación base para deployment: https://lightrains.com/blogs/deploy-aws-ec2-using-github-actions/
- GitHub Actions - Pull Request Events: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#pull_request
- Action SSH Deploy: https://github.com/easingthemes/ssh-deploy

### Resultado Esperado
Un archivo `.github/workflows/pipeline.yml` completo y funcional que automatice el proceso de testing, build y deployment del backend a AWS EC2, ejecutándose únicamente en pushes a ramas con Pull Requests abiertos.

El workflow debe ser robusto, eficiente y proporcionar feedback claro sobre cada etapa del proceso de CI/CD.
