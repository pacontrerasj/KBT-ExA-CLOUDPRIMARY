# KBT-ExA-CLOUDPRIMARY

Repositorio de plantillas reutilizables de CI/CD para proyectos de infraestructura en AWS.

## Plantillas Disponibles

| Plantilla | Descripción |
|-----------|-------------|
| `terraform-deploy.yml` | Deploy de infraestructura con Terraform |
| `terraform-destroy.yml` | Destrucción de infraestructura con Terraform |
| `docker-build-push.yml` | Build y push de imágenes Docker a ECR |

## Requisitos Previos

### Secrets de GitHub

Estos secrets deben estar configurados en tu repositorio:

| Secret | Descripción |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | ID de acceso de AWS |
| `AWS_SECRET_ACCESS_KEY` | Clave secreta de AWS |
| `AWS_SESSION_TOKEN` | Token de sesión (temporal) |
| `SSH_PRIVATE_KEY` | Clave privada SSH para EC2 |
| `MI_IP` | Tu IP pública en formato CIDR (ej: `201.123.45.67/32`) |
| `KEY_NAME` | Nombre del Key Pair en AWS |

### Variables de GitHub

| Variable | Descripción |
|----------|-------------|
| `EXISTING_VPC_ID` | ID de VPC existente (opcional, dejar vacío para crear nueva) |

## Uso

### Terraform Deploy

En tu repositorio, crea un archivo `.github/workflows/deploy.yml`:

```yaml
name: Deploy Infraestructura

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: pacontrerasj/KBT-ExA-CLOUDPRIMARY/.github/workflows/terraform-deploy.yml@main
    with:
      tf_root: ./terraform
      aws_region: us-east-1
    secrets: inherit
```

### Terraform Destroy

Crea un archivo `.github/workflows/destroy.yml`:

```yaml
name: Destruir Infraestructura

on:
  workflow_dispatch:
    inputs:
      confirm:
        description: 'Escribe "yes" para confirmar'
        required: true
        type: string

jobs:
  destroy:
    if: github.event.inputs.confirm == 'yes'
    uses: pacontrerasj/KBT-ExA-CLOUDPRIMARY/.github/workflows/terraform-destroy.yml@main
    with:
      tf_root: ./terraform
      aws_region: us-east-1
    secrets: inherit
```

### Docker Build & Push

Crea un archivo `.github/workflows/docker.yml`:

```yaml
name: Build & Push Docker

on:
  push:
    branches: [main]

jobs:
  build:
    uses: pacontrerasj/KBT-ExA-CLOUDPRIMARY/.github/workflows/docker-build-push.yml@main
    with:
      frontend_context: ./tienda-vehiculos-frontend
      frontend_dockerfile: ./tienda-vehiculos-frontend/Dockerfile
      backend_context: ./tienda-vehiculos-backend
      backend_dockerfile: ./tienda-vehiculos-backend/Dockerfile
      aws_region: us-east-1
      ecr_frontend_url: ${{ needs.deploy.outputs.ecr_frontend }}
      ecr_backend_url: ${{ needs.deploy.outputs.ecr_backend }}
    secrets: inherit
```

## Estructura de Terraform Requerida

Tu proyecto Terraform debe tener esta estructura:

```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── state.tf
└── modules/
    ├── networking/
    ├── compute/
    ├── database/
    └── ecr/
```

### Outputs Requeridos

Tu `outputs.tf` debe exponer estos valores:

```hcl
output "ec2_public_ip" {
  description = "IP pública de la EC2"
  value       = module.compute.ec2_public_ip
}

output "ecr_frontend_url" {
  description = "URL del repositorio ECR frontend"
  value       = module.ecr.ecr_frontend_url
}

output "ecr_backend_url" {
  description = "URL del repositorio ECR backend"
  value       = module.ecr.ecr_backend_url
}

output "rds_endpoint" {
  description = "Endpoint de RDS"
  value       = module.database.rds_endpoint
}

output "rds_port" {
  description = "Puerto de RDS"
  value       = module.database.rds_port
}

output "db_password" {
  description = "Password generado para RDS"
  value       = module.database.db_password
  sensitive   = true
}
```

## Notas

- Las credenciales AWS Academy expiran cada ~4 horas
- El state de Terraform se almacena en S3 (configurar bucket y DynamoDB previamente)
- El password de RDS se genera automáticamente con `random_password`
- Usa `existing_vpc_id` si ya tienes una VPC creada para evitar límites de cuenta
