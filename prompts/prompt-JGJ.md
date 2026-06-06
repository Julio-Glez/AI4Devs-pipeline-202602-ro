# Prompt: Generar workflow de GitHub Actions para CI/CD

## Contexto del proyecto

Eres un experto en DevOps y GitHub Actions. Debes generar el archivo `.github/workflows/ci.yml` completo y funcional para un monorepo full-stack con las siguientes características:

### Stack tecnológico
- **Backend**: Node.js 18 + Express + TypeScript, compilado con `tsc` (`npm run build`), ejecutado con PM2 en producción
- **Frontend**: React 18 (Create React App) + TypeScript, construido con `npm run build`
- **Base de datos**: PostgreSQL gestionado con Prisma ORM
- **Servidor de producción**: Amazon Linux 2023 en EC2, con Nginx como proxy y PM2 para gestionar el proceso Node.js

### Estructura del monorepo
```
/ (raíz)
├── backend/           # Express + TypeScript
│   ├── src/
│   ├── prisma/
│   │   └── schema.prisma
│   ├── package.json
│   └── .env           # Variables de entorno del backend
├── frontend/          # React + TypeScript
│   ├── src/
│   └── package.json
├── docker-compose.yml # PostgreSQL local
└── .github/
    └── workflows/
        └── ci.yml     # ARCHIVO A GENERAR
```

### Comandos relevantes
| Entorno | Comando | Descripción |
|---------|---------|-------------|
| Backend | `npm install` | Instalar dependencias |
| Backend | `npm run build` | Compilar TypeScript → `dist/` |
| Backend | `npm test` | Ejecutar tests con Jest |
| Backend | `npx prisma generate` | Generar cliente Prisma |
| Backend | `npx prisma migrate deploy` | Aplicar migraciones en producción |
| Frontend | `npm install` | Instalar dependencias |
| Frontend | `npm run build` | Build de producción |
| Frontend | `npm test` | Tests con Jest (requiere `CI=true`) |

---

## Infraestructura de producción (EC2 ya configurada)

La instancia EC2 ya tiene instalado:
- Node.js v18.20.8
- PM2 v7.0.1
- Nginx v1.30.1

**Ruta de despliegue en EC2**: `/home/ec2-user/app`

El backend debe correr en el **puerto 3010** gestionado por PM2.
El frontend (build estático) debe servirse desde Nginx en el **puerto 80**.

---

## GitHub Secrets disponibles

El repositorio ya tiene configurados los siguientes secrets:
- `EC2_SSH_KEY` — contenido completo del archivo `.pem` para SSH
- `EC2_HOST` — hostname público de la instancia: `ec2-3-16-161-44.us-east-2.compute.amazonaws.com`
- `EC2_USER` — usuario SSH: `ec2-user`
- `AWS_ACCESS_ID` — Access Key ID de AWS
- `AWS_ACCESS_KEY` — Secret Access Key de AWS

---

## Requisitos del workflow

### Triggers
- Se debe ejecutar en **push** a las ramas `main` y `feature-JGJ`
- Se debe ejecutar en **pull_request** hacia `main`

### Jobs requeridos

#### 1. `test-backend`
- Ejecutar en `ubuntu-latest`
- Levantar PostgreSQL como servicio (imagen `postgres:15`) con:
  - `POSTGRES_USER: testuser`
  - `POSTGRES_PASSWORD: testpass`
  - `POSTGRES_DB: testdb`
  - Puerto mapeado: `5432:5432`
  - Health check incluido
- Pasos:
  1. Checkout del código
  2. Usar Node.js 18
  3. Instalar dependencias (`npm ci`)
  4. Generar cliente Prisma (`npx prisma generate`)
  5. Aplicar migraciones (`npx prisma migrate deploy`) con `DATABASE_URL=postgresql://testuser:testpass@localhost:5432/testdb`
  6. Ejecutar tests (`npm test`)

#### 2. `build-backend`
- Depende de `test-backend`
- Ejecutar en `ubuntu-latest`
- Pasos:
  1. Checkout del código
  2. Usar Node.js 18
  3. Instalar dependencias (`npm ci`)
  4. Compilar TypeScript (`npm run build`)
  5. Verificar que el directorio `dist/` existe

#### 3. `build-frontend`
- Ejecutar en `ubuntu-latest` (en paralelo con `build-backend`, sin dependencia entre ellos)
- Pasos:
  1. Checkout del código
  2. Usar Node.js 18
  3. Instalar dependencias (`npm ci`)
  4. Ejecutar tests con `CI=true npm test -- --watchAll=false`
  5. Build de producción (`npm run build`)

#### 4. `deploy`
- Solo ejecutar cuando el **push es a `main`** (no en PRs ni otras ramas)
- Depende de `build-backend` y `build-frontend`
- Ejecutar en `ubuntu-latest`
- Usar la action `appleboy/ssh-action@v1.0.3` para conectarse a la EC2 y ejecutar el script de despliegue
- El script de despliegue en la EC2 debe:
  1. Ir al directorio `/home/ec2-user/app` (crearlo si no existe)
  2. Hacer `git pull origin main` (o clonar si no existe el repo)
  3. Instalar dependencias del backend (`npm ci`)
  4. Generar cliente Prisma (`npx prisma generate`)
  5. Aplicar migraciones (`npx prisma migrate deploy`)
  6. Compilar el backend (`npm run build`)
  7. Instalar dependencias del frontend (`npm ci`)
  8. Construir el frontend (`npm run build`)
  9. Reiniciar (o iniciar) el backend con PM2: `pm2 restart backend || pm2 start dist/index.js --name backend`
  10. Guardar configuración de PM2: `pm2 save`
  11. Copiar el build del frontend a la carpeta de Nginx: `sudo cp -r build/* /usr/share/nginx/html/`

---

## Restricciones y buenas prácticas

- **NUNCA** incluir credenciales, contraseñas ni secrets directamente en el YAML — usar siempre `${{ secrets.NOMBRE }}`
- Usar `working-directory` para ejecutar comandos en `backend/` o `frontend/` según corresponda
- Usar `cache` de npm para acelerar las ejecuciones (action `actions/cache@v4` o `actions/setup-node@v4` con `cache: 'npm'`)
- Añadir nombres descriptivos (`name:`) a cada step
- Usar versiones fijas de las actions (no `@latest`)
- El job `deploy` solo corre en push a `main`, usar condición `if: github.ref == 'refs/heads/main' && github.event_name == 'push'`

---

## Resultado esperado

Genera el archivo `.github/workflows/ci.yml` completo, listo para hacer commit. El archivo debe ser YAML válido, bien indentado y seguir las mejores prácticas de GitHub Actions.
