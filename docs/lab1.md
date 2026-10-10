# Resumen laboratorio 1

## 1. Arquitectura de contenedores y CI/CD aplicados

**Aplicación y contenedor.** API FastAPI (`/` saludo personalizado, `/health`) empaquetada con `python:3.12-slim`, dependencias con Poetry y servida con Uvicorn en el puerto 8000. El `.dockerignore` excluye tests, docs y `.git` para una imagen más liviana.

**Infraestructura en AWS (ECS sobre EC2).** `scripts/deploy.sh` la crea de forma idempotente con AWS CLI, con todos los recursos nombrados con el sufijo del estudiante:

| Capa | Recurso |
|---|---|
| Registro | ECR `greeting-<sufijo>`, imágenes etiquetadas con `<sha>` y `latest` |
| Cómputo | Launch Template (`t3.micro`, AMI ECS-optimized AL2023) + Auto Scaling Group 1/1 |
| Orquestación | Cluster ECS + Capacity Provider (ASG) + Servicio (1 tarea) + Task Definition (bridge, 256 CPU/256 MiB) |
| Red | VPC por defecto, Security Group con puerto 8000 abierto |
| Permisos | Rol de instancia EC2 y rol de ejecución de tareas ECS |
| Logs | CloudWatch Logs `/ecs/greeting-<sufijo>` |

**Versiones y herramientas.**

| Componente | Versión | Uso |
|---|---|---|
| Python | 3.12 (`^3.12`), igual en CI y en la imagen `python:3.12-slim` | Lenguaje y runtime |
| Poetry | 2.3.4 (generó `poetry.lock`); en CI y Dockerfile se instala la última disponible (`pip install poetry`) | Gestión de dependencias |
| FastAPI / Uvicorn | 0.115.14 / 0.32.1 (fijadas en `poetry.lock`) | API y servidor ASGI |
| Pylint / Pytest / httpx | 4.0.5 / 9.0.3 / 0.28.1 (grupo `dev`) | Lint, pruebas y cliente de `TestClient` |
| Acciones de GitHub | `checkout@v4`, `setup-python@v5`, `setup-buildx-action@v3`, `configure-aws-credentials@v4`, `amazon-ecr-login@v2` | Pasos del pipeline |
| Runner | `ubuntu-latest` | Entorno de ejecución |

El `poetry.lock` garantiza las mismas versiones de librerías en local, CI y contenedor. En la imagen solo se instalan las dependencias de producción (`--only main`), sin las de desarrollo.

**CI/CD con GitHub Actions.** El pipeline `ci.yml` tiene cuatro jobs encadenados:

1. **`lint`:** instala Poetry y dependencias, y corre `pylint src` (calidad y estilo del código).
2. **`test`:** corre `pytest tests -vv` (3 pruebas: saludo con nombre, saludo genérico y `/health`). Corre en paralelo con `lint`.
3. **`build_and_push`:** requiere que `lint` y `test` pasen. Configura credenciales AWS desde *secrets*, inicia sesión en ECR, crea el repositorio si no existe, construye la imagen y la sube con dos etiquetas: el SHA del commit (trazable) y `latest`. Entrega la URI de la imagen como *output*.
4. **`deploy`:** recibe esa URI y ejecuta `deploy.sh`, que registra una nueva *task definition* con la imagen del commit y fuerza un nuevo despliegue del servicio ECS.

**Disparadores.** En *pull request* a `main` solo corren `lint` y `test` (validación sin tocar AWS). Con *push* a `main` o ejecución manual (`workflow_dispatch`, con sufijo opcional) corre todo el flujo. El sufijo del estudiante viene del input, de la variable `STUDENT_SUFFIX` del repositorio o de `test-student` por defecto, y nombra todos los recursos.

**Destrucción (`destroy.yml`).** Solo `workflow_dispatch`, para que sea bajo demanda. Exige escribir `DESTROY` como confirmación y ejecuta `destroy.sh`, que borra los recursos en orden inverso (servicio, capacity provider, ASG, cluster, launch template, security group, roles, ECR y logs).

## 2. Evidencia vidual de la ejecución del pipeline en GHA
### 2.1. Pipeline de despliegue

![Evidencia Pipeline Despliegue](<Evidencia Pipeline Despliegue.png>)

### 2.2. Pipeline de eliminación

![Evidencia Pipeline de eliminación](<Evidencia Pipeline de eliminación.png>)

### 2.3. EXTRA: Evidencia de los pipelines en GitHub Actions

![Evidencia Pipelines en GHA](<Evidencia Pipelines en GHA.png>)

## 3. Evidencia visual de la petición HTTP a la IP Pública

![Evidencia CURL](<Evidencia CURL.png>)

## 4. Propuestas de mejora

1. **Autenticación sin llaves estáticas:** reemplazar `AWS_ACCESS_KEY_ID/SECRET` por OIDC de GitHub con un rol IAM de mínimo privilegio, eliminando credenciales de larga duración.
2. **Imagen más segura:** usar build multi-stage, ejecutar como usuario no root y añadir un escaneo de vulnerabilidades (p. ej. Trivy) antes del push a ECR.
3. **Disponibilidad:** el ASG 1/1 con IP pública cambiante es un punto único de falla; añadir un Application Load Balancer y al menos 2 instancias/tareas en distintas zonas.
