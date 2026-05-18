# Proyecto Semestral – Microservicios con Docker y AWS

## Arquitectura

```
┌──────────────────────────────────────────────┐
│              EC2 / Docker Compose            │
│                                              │
│  ┌────────────────┐    ┌──────────────────┐  │
│  │  front-despacho│    │  back-despachos  │  │
│  │  React + Nginx │───▶│  Spring Boot     │  │
│  │  :80           │    │  :8081           │  │
│  └────────────────┘    └──────────┬───────┘  │
│                                   │          │
│                        ┌──────────▼───────┐  │
│                        │  back-ventas     │  │
│                        │  Spring Boot     │  │
│                        │  :8080           │  │
│                        └──────────────────┘  │
└──────────────────────────┬───────────────────┘
                           │  AWS RDS MySQL
                           └──────────────────▶
```

## Estructura de archivos añadidos

```
proyecto/
├── .github/
│   └── workflows/
│       └── deploy.yml                        ← CI/CD completo
├── back-Despachos_SpringBoot/
│   └── Springboot-API-REST-DESPACHO/
│       └── Dockerfile                        ← Build multi-stage Java 17
├── back-Ventas_SpringBoot/
│   └── Springboot-API-REST/
│       └── Dockerfile                        ← Build multi-stage Java 17
├── front_despacho/
│   ├── Dockerfile                            ← Build Vite + Nginx
│   └── nginx.conf                            ← Proxy inverso + SPA routing
├── docker-compose.yml                        ← Desarrollo local (con MySQL)
├── docker-compose.prod.yml                   ← Producción (apunta a RDS + ECR)
└── .env.example                              ← Variables de entorno de ejemplo
```

---

## Primeros pasos (desarrollo local)

### 1. Clonar y configurar variables de entorno

```bash
git clone <tu-repo>
cd proyecto
cp .env.example .env
# Edita .env con tus valores locales
```

### 2. Levantar todos los servicios

```bash
docker compose up --build
```

| Servicio          | URL local                        |
|-------------------|----------------------------------|
| Frontend          | http://localhost:3000            |
| API Despachos     | http://localhost:8081/swagger-ui.html |
| API Ventas        | http://localhost:8080/swagger-ui.html |

### 3. Detener los servicios

```bash
docker compose down
# Para eliminar también los volúmenes de BD:
docker compose down -v
```

---

## Configuración AWS

### Prerequisitos en AWS

1. **ECR** – Crear tres repositorios:
   - `back-despachos`
   - `back-ventas`
   - `front-despacho`

2. **EC2** – Una instancia (Ubuntu 22.04 recomendado) con:
   - Docker y Docker Compose instalados
   - AWS CLI configurado con permisos para ECR
   - El repositorio clonado en `~/app`
   - Un archivo `~/app/.env` con las variables de producción (incluyendo `DB_ENDPOINT` apuntando a RDS)

3. **RDS** – MySQL 8.0, accesible desde el Security Group de la EC2

### Secrets de GitHub necesarios

En **Settings → Secrets and variables → Actions** del repositorio, agregar:

| Secret                  | Descripción                                    |
|-------------------------|------------------------------------------------|
| `AWS_ACCESS_KEY_ID`     | Access key de IAM con permisos ECR + EC2       |
| `AWS_SECRET_ACCESS_KEY` | Secret key del usuario IAM                    |
| `AWS_ACCOUNT_ID`        | ID de cuenta AWS (12 dígitos)                  |
| `EC2_HOST`              | IP pública o DNS de la instancia EC2           |
| `EC2_USER`              | Usuario SSH (ej: `ubuntu`)                     |
| `EC2_SSH_KEY`           | Contenido completo del archivo `.pem` privado  |

### Flujo del pipeline (GitHub Actions)

```
push a main
    │
    ├─ JOB 1: Tests (mvn test en ambos backends)
    │
    ├─ JOB 2: Build de imágenes Docker → push a ECR
    │          (tag = SHA corto del commit)
    │
    └─ JOB 3: SSH a EC2 → git pull → docker compose pull → docker compose up
```

Los **pull requests** solo ejecutan los tests (Job 1), sin push ni deploy.

---

## Variables de entorno de producción en EC2

Crea el archivo `~/app/.env` directamente en la instancia EC2 (nunca lo subas al repo):

```env
# RDS
DB_ENDPOINT=xxxxx.rds.amazonaws.com
DB_PORT=3306
DB_NAME_DESPACHOS=despachos_db
DB_NAME_VENTAS=ventas_db
DB_USERNAME=citt_user
DB_PASSWORD=tu_password_seguro

# AWS
AWS_ACCOUNT_ID=123456789012
AWS_REGION=us-east-1
```
