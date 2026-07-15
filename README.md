# Tienda de Perritos — DevOps EV3 (AWS EKS + CI/CD)
Gestión de productos para una tienda de alimentos para perritos, desplegada como una arquitectura de 3 capas (Frontend, Backend, Base de Datos) sobre **Amazon EKS**, con infraestructura versionada mediante **CloudFormation (IaC)** y un pipeline de **CI/CD automatizado con GitHub Actions**.

Proyecto desarrollado para la Evaluación de Introducción a Herramientas DevOps (ISY1101), en el marco del caso de estudio **Innovatech Chile**.

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

Los servicios definidos corren como pods independientes dentro de un mismo clúster EKS, comunicándose mediante el DNS interno integrado de Kubernetes (Services), sin necesidad de IPs fijas ni de una instancia EC2 dedicada por servicio.

<img src="https://raw.githubusercontent.com/ToniC-3PO/miniature-tribble-img/refs/heads/main/arquitectura_v3.png" alt="Arquitectura" width="900" />

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
| Observabilidad | Amazon CloudWatch (Container Insights) |

## Estructura del repositorio

```
.
├── .github/workflows/
│   └── deploy.yml                # Pipeline CI/CD: build → push (ECR) → deploy (EKS)
├── infra/
│   └── duoc-devops-act3.yaml     # Plantilla CloudFormation (IaC) — VPC, ECR, EKS, NodeGroup, Addons
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js                 # API REST (Express) — CRUD de productos
├── frontend/
│   ├── Dockerfile
│   ├── default.conf              # Configuración Nginx (proxy /api → backend)
│   ├── index.html
│   └── app.js
├── db/
│   ├── Dockerfile
│   └── init.sql                  # Esquema inicial + datos de ejemplo
├── k8s/
│   ├── namespace.yaml
│   ├── mysql-secret.example.yaml # Plantilla del Secret (sin credenciales reales, ver sección Configuración)
│   ├── mysql-deployment.yaml
│   ├── mysql-service.yaml
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── backend-hpa.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   └── frontend-hpa.yaml
├── docs/img/                      # Capturas y diagramas usados en este README
└── deploy.sh                      # Script de despliegue manual (uso inicial vía CloudShell)
```

> **Nota:** `mysql-secret.yaml` con las credenciales reales **no se versiona** en el repositorio. Ver sección [Configuración y secretos](#configuración-y-secretos-de-la-base-de-datos) para generarlo antes del despliegue.

## Infraestructura (CloudFormation)

La infraestructura base se provisiona como código: el stack de CloudFormation (`devops-ev3-stack`) se crea a partir de la plantilla versionada en [`infra/duoc-devops-act3.yaml`](infra/duoc-devops-act3.yaml), que aprovisiona:

- **VPC** con 2 zonas de disponibilidad, 2 subredes públicas y 4 subredes privadas, Internet Gateway y NAT Gateway.
- **3 repositorios ECR**: `tienda-frontend`, `tienda-backend`, `tienda-db`.
- **Clúster EKS** (`devopseks`, Kubernetes 1.35) con su Node Group administrado y los addons: VPC CNI, CoreDNS, Kube-proxy, Pod Identity y CloudWatch Observability.

El stack requiere los roles IAM del Learner Lab como parámetros al momento de crearse:

| Parámetro | Valor |
|---|---|
| `EksClusterRoleName` | Nombre del rol `LabEksClusterRole-XXXX` (obtenido desde IAM → Roles) |
| `EksNodeRoleName` | Nombre del rol `LabEksNodeRole-XXXX` (obtenido desde IAM → Roles) |

> Estos nombres son distintos en cada sesión del Learner Lab, por lo que deben copiarse desde la consola de IAM antes de crear el stack.

<!-- FOTO
-->

## Configuración y secretos de la base de datos

Las credenciales de MySQL se gestionan mediante un `Secret` de Kubernetes que **no se sube al repositorio con valores reales**. Antes del primer despliegue, se debe generar localmente a partir de la plantilla:

```bash
kubectl create secret generic mysql-secret \
  --namespace tienda \
  --from-literal=MYSQL_ROOT_PASSWORD='<contraseña-fuerte-generada>' \
  --from-literal=MYSQL_DATABASE=tienda_perritos \
  --dry-run=client -o yaml > k8s/mysql-secret.yaml

kubectl apply -f k8s/mysql-secret.yaml
```

> La contraseña debe generarse de forma aleatoria (por ejemplo `openssl rand -base64 24`) y nunca debe ser un valor trivial o de diccionario. El archivo `k8s/mysql-secret.yaml` con el valor real queda ignorado en `.gitignore` y solo se versiona `mysql-secret.example.yaml` como referencia de formato.

## Configuración necesaria (Secrets de GitHub Actions) antes del despliegue

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access key del Learner Lab (temporal) |
| `AWS_SECRET_ACCESS_KEY` | Secret key del Learner Lab (temporal) |
| `AWS_SESSION_TOKEN` | Session token del Learner Lab (temporal) |
| `AWS_REGION` | Región de AWS (`us-east-1`) |
| `EKS_CLUSTER_NAME` | Nombre del clúster EKS (`devopseks`) |
| `EKS_NAMESPACE` | Namespace de Kubernetes (`tienda`) |

Las credenciales `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` y `AWS_SESSION_TOKEN` se obtienen desde AWS Academy Learner Lab → **AWS Details** → **AWS CLI**.

> **Nota:** al tratarse de un entorno AWS Academy Learner Lab, las credenciales son temporales y deben actualizarse en los Secrets del repositorio cada vez que expira la sesión del laboratorio.

## Despliegue

### Opción 1 — Automático (CI/CD, recomendado)

Cada `push` a la rama `main` dispara automáticamente el workflow [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml), que:

1. Construye las 3 imágenes Docker (frontend, backend, db).
2. Las etiqueta con el hash corto del commit (`${GITHUB_SHA::7}`) y las sube a Amazon ECR.
3. Aplica los manifests base de Kubernetes (namespace, MySQL, Services, HPA).
4. Actualiza cada Deployment a la nueva imagen mediante `kubectl set image`, lo que dispara un *rolling update* sin downtime.
5. Muestra en el log el estado final de los pods, los HPA y la URL pública de la aplicación.

También puede lanzarse manualmente desde la pestaña **Actions** vía `workflow_dispatch`.

<!-- FOTO
-->

<!-- FOTO O TABLA TIEMPOS
-->

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

El umbral del backend se fijó más alto (70%) porque concentra la lógica de negocio y las consultas a la base de datos; el del frontend es más bajo (60%) dado que Nginx, al servir contenido estático, opera con menos CPU en condiciones normales y necesita menos margen antes de escalar.

### Cómo reproducir la prueba de carga

```bash
kubectl run carga --image=busybox --namespace tienda -it --rm --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://tienda-backend:3001/api/productos; done"
```

Mientras la carga corre, observar el escalado en otra terminal:

```bash
kubectl get hpa -n tienda -w
```

<!-- FOTO PEAK
-->

## Observabilidad

### Logs del pipeline
Cada ejecución del workflow queda registrada en la pestaña **Actions**, incluyendo el detalle de build, push y deploy.

### Métricas en CloudWatch
El addon **CloudWatch Observability** (Container Insights) queda habilitado por el propio stack de CloudFormation, entregando métricas de CPU/memoria a nivel de clúster, nodos y pods.

<!-- FOTO CLOUD
-->

### Validación post-deploy (comunicación Frontend → Backend)
Tras cada despliegue se valida el flujo completo ejecutando operaciones CRUD reales contra la URL pública, confirmando que las peticiones del frontend al backend responden correctamente.

<!-- FOTO DEVTOOLS
-->

## Uso de la aplicación

Una vez desplegada, la aplicación expone un CRUD de productos:

- **Frontend**: accesible vía la URL del LoadBalancer (`http://<hostname-elb>.us-east-1.elb.amazonaws.com`). El hostname lo genera AWS automáticamente al crear el Service de tipo LoadBalancer; la región del Learner Lab es `us-east-1`.
- **API Backend**: expone `/api/productos` (`GET`, `POST`, `PUT`, `DELETE`) y `/api/health` para verificación de estado.

<!-- APP FUNCIONANDO
-->

## Autores

- Nicolás Cifuentes
- Antonia Calderón
