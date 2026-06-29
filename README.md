# Tienda de Perritos — DevOps EV3 (AWS EKS + CI/CD)

Gestión de productos para una tienda de alimentos para perritos, desplegada como una arquitectura de 3 capas (Frontend, Backend, Base de Datos) sobre **Amazon EKS**, con infraestructura guardada mediante **CloudFormation (IaC)** y un pipeline de **CI/CD automatizado con GitHub Actions**.

Proyecto desarrollado para la Evaluación Parcial N°3 de Introducción a Herramientas DevOps (ISY1101), en el marco del caso de estudio **Innovatech Chile**.

## Arquitectura

```
                         ┌──────────────────────┐
   Internet ──────────▶ │   LoadBalancer (ELB)  │
                         └───────────┬──────────┘
                                     │
                         ┌───────────▼──────────┐
                         │   tienda-frontend     │  (Nginx + JS, 2-6 réplicas)
                         │   Service / HPA       │
                         └───────────┬──────────┘
                                     │ DNS interno (Service)
                         ┌───────────▼──────────┐
                         │   tienda-backend      │  (Node.js/Express, 2-10 réplicas)
                         │   Service / HPA       │
                         └───────────┬──────────┘
                                     │ DNS interno (Service)
                         ┌───────────▼──────────┐
                         │   tienda-db           │  (MySQL 8)
                         │   Service ClusterIP   │
                         └───────────────────────┘
```

los servicios definidos corren como pods independientes dentro de un mimso clúster EKS, comunicandose mediante el DNS interno integrado de Kubernetes (Services), sin tener la necesidad de IPs fijas como tampoco EC2 por cada servicio.

## Stack tecnológico

| Capa | Tecnología |
|---|---|
| Frontend | Nginx (alpine) + HTML/JS vanilla |
| Backend | Node.js 18 + Express + mysql2 |
| Base de datos | MySQL 8 |
| Orquestación | Amazon EKS (Kubernetes 1.35) |
| Infraestructura | AWS CloudFormation (VPC, ECR, EKS, NodeGroup, Addons) |
| Registro de imágenes | Amazon ECR |
| CI/CD | GitHub Actions |
| Autoscaling | Horizontal Pod Autoscaler (HPA) + Metrics Server |

## Estructura del repositorio

```
.
├── .github/workflows/
│   └── deploy.yml              # Pipeline CI/CD: build → push (ECR) → deploy (EKS)
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js                # API REST (Express) — CRUD de productos
├── frontend/
│   ├── Dockerfile
│   ├── default.conf             # Configuración Nginx (proxy /api → backend)
│   ├── index.html
│   └── app.js
├── db/
│   ├── Dockerfile
│   └── init.sql                 # Esquema inicial + datos de ejemplo
├── k8s/
│   ├── namespace.yaml
│   ├── mysql-secret.yaml
│   ├── mysql-deployment.yaml
│   ├── mysql-service.yaml
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── backend-hpa.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   └── frontend-hpa.yaml
└── deploy.sh                     # Script de despliegue manual (uso inicial vía CloudShell)
```

## Infraestructura (CloudFormation)

La infraestructura base se provisiona mediante IaC: se creó un stack de cloudformation (devops-ev3-stack) a partir de la plantilla proporcionada `duoc-devops-act3.yaml` (entregado por parte de materia de estudio), que crea:

- **VPC** con 2 zonas de disponibilidad, 2 subredes públicas y 4 subredes privadas, Internet Gateway y NAT Gateway.
- **3 repositorios ECR**: `tienda-frontend`, `tienda-backend`, `tienda-db`.
- **Clúster EKS** (`devopseks`, Kubernetes 1.35) con su Node Group administrado y los addons: VPC CNI, CoreDNS, Kube-proxy, Pod Identity y CloudWatch Observability.

y requiriendo integrar parametros de roles IAM (`LabEksClusterRole`, `LabEksNodeRole`) como requisitos al crear al stack.

| Parámetro | Valor |
|---|---|
| `EksClusterRoleName` | Nombre del rol `LabEksClusterRole-XXXX` (obtenido desde IAM → Roles) |
| `EksNodeRoleName` | Nombre del rol `LabEksNodeRole-XXXX` (obtenido desde IAM → Roles) |

> Estos nombres son distintos en cada sesión del Learner Lab, por lo que deben copiarse desde la consola de IAM antes de crear el stack.

## Configuración necesaria (Secrets de GitHub Actions) antes del despliegue

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access key del Learner Lab (temporal) |     <────┐
| `AWS_SECRET_ACCESS_KEY` | Secret key del Learner Lab (temporal) | <────── Credenciales sacadas de Lab AWS Academy -> AWS CLI (dentro de detalles de AWS)
| `AWS_SESSION_TOKEN` | Session token del Learner Lab (temporal) |  <────┘
| `AWS_REGION` | Región de AWS (`us-east-1`) |
| `EKS_CLUSTER_NAME` | Nombre del clúster EKS (`devopseks`) |
| `EKS_NAMESPACE` | Namespace de Kubernetes (`tienda`) |

> **Nota:** al tratarse de un entorno AWS Academy Learner Lab, las credenciales son temporales y deben actualizarse en los Secrets del repositorio cada vez que expira la sesión del laboratorio.

## Despliegue

### Opción 1 — Automático (CI/CD, recomendado)

Cada `push` a la rama `main` dispara automáticamente el workflow `.github/workflows/deploy.yml`, que:

1. Construye las 3 imágenes Docker (frontend, backend, db).
2. Las etiqueta con el hash corto del commit (`${GITHUB_SHA::7}`) y las sube a Amazon ECR.
3. Aplica los manifests base de Kubernetes (namespace, MySQL, Services, HPA).
4. Actualiza cada Deployment a la nueva imagen mediante `kubectl set image`, lo que dispara un *rolling update* sin downtime.
5. Muestra en el log el estado final de los pods, los HPA y la URL pública de la aplicación.

### Opción 2 — Manual (vía AWS CloudShell)

```bash
unzip deploy-tienda-perritos.zip
chmod +x deploy.sh
./deploy.sh
```

El script `deploy.sh` realiza el mismo flujo de build → push → deploy de forma manual, útil para el primer despliegue o para depuración.

## Autoscaling (HPA)

| Servicio | Min réplicas | Max réplicas | Umbral CPU |
|---|---|---|---|
| Backend | 2 | 10 | 70% |
| Frontend | 2 | 6 | 60% |

El umbral del backend se fijó más alto (70%) porque concentra la lógica de negocio y las consultas a la base de datos; el del frontend es más bajo (60%) dado que Nginx, al servir contenido estático, escala con cargas de tráfico relativamente menores.

## Uso de la aplicación

Una vez desplegada, la aplicación expone un CRUD de productos:

- **Frontend**: accesible vía la URL del LoadBalancer (`http://<hostname-elb>.amazonaws.com`).

- **API Backend**: expone `/api/productos` (GET, POST, PUT, DELETE) y `/api/health` para verificación de estado.
  
> **Nota**: en el Frontend el hostname queda distribuido en: (`http://<Load_balancer.AWS_REGION.elb>.amazonaws.com`)> El Learneb Lab lo genera aws y la region es us-east-1.

# Alumnos: Nicolas Cifuentes - Antonia Calderon
