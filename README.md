# 🐶 Tienda de Alimentos para Perritos

Aplicación web de gestión CRUD de productos, desplegada con contenedores Docker en AWS usando un pipeline CI/CD completamente automatizado con GitHub Actions, Amazon ECR y Amazon EKS.

---

## 📋 Descripción del proyecto

Sistema de tres capas compuesto por:

| Componente | Tecnología | Puerto |
|------------|-----------|--------|
| Frontend | HTML + JavaScript + Nginx | 80 |
| Backend | Node.js + Express + mysql2 | 3001 |
| Base de datos | MySQL 8 | 3306 |

El sistema permite crear, leer, actualizar y eliminar productos de una tienda de alimentos para mascotas. Todo el ciclo de integración, empaquetado y despliegue está automatizado con GitHub Actions.

---

## 🏗️ Arquitectura AWS

### VPC y red — VPC-TiendaPerritos (10.0.0.0/16)

| Subred | Zona | CIDR | Propósito |
|--------|------|------|-----------|
| Subred-Publica-Front | us-east-1a | 10.0.1.0/24 | ec2-web (frontend), tráfico público |
| Subred-Publica-Front-1b | us-east-1b | 10.0.4.0/24 | Nodos EKS públicos (requerida por EKS) |
| Subred-Privada-APP | us-east-1a | 10.0.2.0/24 | ec2-app (backend), nodos EKS privados |
| Subred-Privada-APP-1b | us-east-1b | 10.0.3.0/24 | Nodos EKS privados (requerida por EKS) |
| Subred-Privada-Datos | us-east-1a | 10.0.5.0/24 | ec2-datos (MySQL), sin réplica en 1b |

### Entorno de desarrollo — EC2

Tres instancias EC2 (t3.micro) desplegadas con **User Data 100% automatizado**:

- **ec2-web**: Nginx sirviendo el frontend en el puerto 80 (subred pública, IP elástica)
- **ec2-app**: Backend Node.js/Express conectado a MySQL en ec2-datos
- **ec2-datos**: MySQL 8 con Named Volume para persistencia de datos

### Entorno de producción — Amazon EKS

Clúster `tienda-eks` con Node Group `workers` (t3.medium, 2 nodos):

- `tienda-frontend`: Deployment con 2 réplicas + HPA (2-6 pods, escala al 60% CPU)
- `tienda-backend`: Deployment con 2 réplicas + HPA (2-10 pods, escala al 70% CPU)
- `tienda-db`: Deployment con 1 réplica + headless Service
- Service LoadBalancer (ELB) expone el frontend públicamente

### Registro de imágenes — Amazon ECR

```
511433558605.dkr.ecr.us-east-1.amazonaws.com/tienda-frontend
511433558605.dkr.ecr.us-east-1.amazonaws.com/tienda-backend
511433558605.dkr.ecr.us-east-1.amazonaws.com/tienda-db
```

---

## 🔄 Pipeline CI/CD

El workflow `.github/workflows/deploy-eks.yml` se activa automáticamente con cada push a `main`:

```
Push a main
    │
    ▼
1. Checkout del código
    │
    ▼
2. Configurar credenciales AWS
    │
    ▼
3. Login a Amazon ECR
    │
    ▼
4. Build y push de 3 imágenes (tag = primeros 7 chars del SHA del commit)
    │
    ▼
5. Instalar kubectl y configurar kubeconfig del clúster EKS
    │
    ▼
6. kubectl apply (namespace, Secret MySQL, Deployments, Services)
    │
    ▼
7. kubectl set image → rolling update sin downtime
    │
    ▼
8. kubectl rollout status → verificación final
```

### Secrets de GitHub Actions requeridos

| Secret | Descripción |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | Credencial de AWS Academy |
| `AWS_SECRET_ACCESS_KEY` | Credencial de AWS Academy |
| `AWS_SESSION_TOKEN` | Token de sesión de AWS Academy |
| `AWS_REGION` | `us-east-1` |
| `EKS_CLUSTER_NAME` | `tienda-eks` |
| `EKS_NAMESPACE` | `tienda` |

---

## 🐳 Contenedores y Docker Compose

### Dockerfiles

Cada componente tiene su propio Dockerfile con imágenes base minimalistas:

- **Frontend**: `nginx:alpine` — multi-stage no aplica, imagen base liviana
- **Backend**: `node:18-alpine` — imagen minimalista Alpine
- **Base de datos**: `mysql:8` con script `init.sql` para carga inicial de datos

### Ejecución local con Docker Compose

```bash
# Levantar el stack completo
docker compose up -d --build

# Verificar estado
docker compose ps

# Ver logs del backend
docker compose logs tienda-backend --tail 20

# Detener
docker compose down
```

**URLs locales:**
- Frontend: http://localhost
- API health: http://localhost:3001/api/health
- API productos: http://localhost:3001/api/productos

---

## 🔒 Seguridad

### Security Groups

| Security Group | Instancia | Reglas de entrada |
|---------------|-----------|-------------------|
| SG-Frontend-Tienda | ec2-web | Puerto 80 desde 0.0.0.0/0, SSH 22 |
| SG-Backend-Tienda | ec2-app | Puerto 3001 desde SG-Frontend, SSH 22 |
| SG-Datos-Tienda | ec2-datos | Puerto 3306 desde SG-Backend, SSH 22 |

### Gestión de secretos

- Credenciales AWS almacenadas en GitHub Actions Secrets (nunca en el código)
- Contraseña MySQL almacenada como Kubernetes Secret (`mysql-secret`) en base64
- Principio de mínimo privilegio: rol IAM `LabRole` para acceso a ECR desde EC2 y EKS

---

## 📈 Observabilidad

### Orquestación y escalabilidad (EKS)

- **HPA backend**: escala automáticamente entre 2 y 10 réplicas según uso de CPU (umbral 70%)
- **HPA frontend**: escala automáticamente entre 2 y 6 réplicas según uso de CPU (umbral 60%)
- **Health checks**: liveness y readiness probes en `/api/health` para el backend

### Logs

```bash
# Logs del pipeline: GitHub → Actions → CI/CD Tienda Perritos EKS

# Logs del backend en EKS
kubectl logs deployment/tienda-backend -n tienda --tail 30

# Logs del frontend en EKS
kubectl logs deployment/tienda-frontend -n tienda --tail 10

# Estado de todos los recursos
kubectl get all -n tienda
```

### Métricas

```bash
# CPU y memoria de los pods
kubectl top pods -n tienda

# CPU y memoria de los nodos
kubectl top nodes
```

---

## 🗂️ Estructura del repositorio

```
tienda-perritos-eks/
├── .github/
│   └── workflows/
│       └── deploy-eks.yml       # Pipeline CI/CD completo
├── backend/
│   ├── Dockerfile               # Imagen node:18-alpine
│   ├── package.json
│   └── server.js                # API REST Express
├── db/
│   ├── Dockerfile               # Imagen mysql:8 con init.sql
│   └── init.sql                 # Carga inicial de 5 productos
├── frontend/
│   ├── Dockerfile               # Imagen nginx:alpine
│   ├── default.conf             # Nginx con proxy /api/ → backend
│   ├── index.html
│   └── app.js                   # CRUD de productos
├── k8s/
│   ├── namespace.yaml           # Namespace tienda
│   ├── mysql-secret.yaml        # Contraseña MySQL en base64
│   ├── mysql-deployment.yaml    # Pod MySQL
│   ├── mysql-service.yaml       # Headless Service :3306
│   ├── backend-deployment.yaml  # 2 réplicas Node.js
│   ├── backend-service.yaml     # ClusterIP :3001
│   ├── backend-hpa.yaml         # Autoescalado 2-10 pods
│   ├── frontend-deployment.yaml # 2 réplicas Nginx
│   ├── frontend-service.yaml    # LoadBalancer :80 (ELB)
│   └── frontend-hpa.yaml        # Autoescalado 2-6 pods
└── docker-compose.yml           # Entorno de desarrollo local
```

---

## 🚀 Despliegue manual (primera vez)

### Prerrequisitos

- AWS CLI v2 configurado con credenciales de AWS Academy
- Docker Desktop instalado y corriendo
- kubectl instalado
- Acceso al clúster EKS: `aws eks update-kubeconfig --region us-east-1 --name tienda-eks`

### Pasos

```bash
# 1. Autenticar con ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  511433558605.dkr.ecr.us-east-1.amazonaws.com

# 2. Build y push de imágenes
docker build -t tienda-db:latest ./db
docker tag tienda-db:latest 511433558605.dkr.ecr.us-east-1.amazonaws.com/tienda-db:latest
docker push 511433558605.dkr.ecr.us-east-1.amazonaws.com/tienda-db:latest

docker build -t tienda-backend:latest ./backend
docker tag tienda-backend:latest 511433558605.dkr.ecr.us-east-1.amazonaws.com/tienda-backend:latest
docker push 511433558605.dkr.ecr.us-east-1.amazonaws.com/tienda-backend:latest

docker build -t tienda-frontend:latest ./frontend
docker tag tienda-frontend:latest 511433558605.dkr.ecr.us-east-1.amazonaws.com/tienda-frontend:latest
docker push 511433558605.dkr.ecr.us-east-1.amazonaws.com/tienda-frontend:latest

# 3. Aplicar manifiestos en EKS (orden estricto)
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/mysql-secret.yaml -n tienda
kubectl apply -f k8s/mysql-deployment.yaml -n tienda
kubectl apply -f k8s/mysql-service.yaml -n tienda
# Esperar que MySQL esté Running
kubectl apply -f k8s/backend-deployment.yaml -n tienda
kubectl apply -f k8s/backend-service.yaml -n tienda
kubectl apply -f k8s/backend-hpa.yaml -n tienda
kubectl apply -f k8s/frontend-deployment.yaml -n tienda
kubectl apply -f k8s/frontend-service.yaml -n tienda
kubectl apply -f k8s/frontend-hpa.yaml -n tienda

# 4. Obtener URL pública del ELB
kubectl get svc tienda-frontend -n tienda
```

---

## 👥 Autores

- **Nicolás Lorca Salamanca**
- **Katherine Ramírez Carvajal**

Ingeniería en Informática — Desarrollo de Software  
DuocUC, sede San Bernardo  
Introducción a Herramientas DevOps (ISY1101) — 2026