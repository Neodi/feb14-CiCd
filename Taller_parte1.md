# Pipeline CI/CD con GitHub Actions, Ansible y GKE

## Despliegue Automatizado del Crypto Chatbot en Google Kubernetes Engine

---

## 1. Arquitectura General

### Flujo del Pipeline

```
Developer Push (main)
        ↓
GitHub Actions Runner
  ├─ Checkout código
  ├─ Crear .env backend
  ├─ Auth GCP (Service Account)
  ├─ Build imagen backend (Python)
  ├─ Build imagen frontend (Nginx)
  ├─ Push ambas imágenes → Artifact Registry
  ├─ Instalar Ansible en el runner
  └─ ansible-playbook deploy-gke.yml
              ↓
        GCP API (via gcloud / kubectl)
              ↓
        GKE Autopilot Cluster
          ├─ Namespace: ucm-master
          ├─ Deployment: agent-backend  (ClusterIP :80)
          ├─ Deployment: agent-frontend (LoadBalancer :80)
          └─ Docker Pull ← Artifact Registry
              ↓
        End Users → http://EXTERNAL_IP
```


## 2. Prerrequisitos

Antes de comenzar, asegúrate de tener lo siguiente:

- Repositorio en GitHub con el código del chatbot (frontend + backend).
- Proyecto en GCP con facturación habilitada.
- APIs habilitadas: Container, Artifact Registry, IAM.

Configurar las variables de entorno (en Cloud Shell):

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export CLUSTER_NAME=test-cluster-a
```

Habilitar las APIs necesarias:

```bash
gcloud services enable \
  container.googleapis.com \
  artifactregistry.googleapis.com \
  iam.googleapis.com
```

---

## Paso 1 — Service Account de CI en GCP

Creamos una service account dedicada exclusivamente para GitHub Actions. Esta cuenta se usará para autenticarse contra GCP, subir las imágenes Docker al Artifact Registry y desplegar en GKE.

### Crear la Service Account

```bash
gcloud iam service-accounts create github-ci \
  --display-name="GitHub Actions CI"
```

### Asignar Roles

La service account necesita permisos para escribir en Artifact Registry **y** gestionar el clúster GKE:

```bash
# Escritura en Artifact Registry
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-ci@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

# Acceso al clúster GKE
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-ci@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/container.developer"

# Obtener credenciales del clúster
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-ci@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/container.clusterViewer"
```

### Generar Clave JSON

```bash
gcloud iam service-accounts keys create github-ci-key.json \
  --iam-account=github-ci@${PROJECT_ID}.iam.gserviceaccount.com
```

>  **Importante:** Guarda este archivo JSON. Lo necesitarás para configurar los secretos de GitHub. No lo subas al repositorio.

---

## Paso 2 — Artifact Registry

Crear el repositorio Docker en Artifact Registry (una sola vez, quizás lo tengas de la práctica del día anterior):

```bash
gcloud artifacts repositories create agent-repo \
  --repository-format=docker \
  --location=$REGION \
  --description="Repositorio de imágenes del Crypto Chatbot"
```

La ruta del registro será:  


```
us-central1-docker.pkg.dev/PROJECT_ID/agent-repo
```

Verificar que se ha creado correctamente:

```bash
gcloud artifacts repositories list --location=$REGION
```

Lo hacemos público (para que GKE pueda hacer pull sin configuración adicional de secretos de imagen; en producción se usaría Workload Identity):

```bash
gcloud artifacts repositories add-iam-policy-binding agent-repo \
  --location=us-central1 \
  --member="allUsers" \
  --role="roles/artifactregistry.reader"
```

---

## Paso 3 — Crear el Clúster GKE Autopilot

GKE Autopilot gestiona automáticamente los nodos, el escalado y las actualizaciones. 

```bash
gcloud container clusters create-auto $CLUSTER_NAME \
  --project=$PROJECT_ID \
  --region=$REGION \
  --release-channel=regular
```

>  La creación del clúster tarda entre 5 y 10 minutos.

Verificar que el clúster está listo:

```bash
gcloud container clusters list --region=$REGION
```

Obtener las credenciales de kubectl (desde Cloud Shell):

```bash
gcloud container clusters get-credentials $CLUSTER_NAME \
  --region=$REGION \
  --project=$PROJECT_ID
```

Verificar la conexión:

```bash
kubectl cluster-info
kubectl get namespaces
```

---

## Paso 4 — Manifiestos de Kubernetes

Estos ficheros YAML se almacenan en el repositorio GitHub, dentro de una carpeta `k8s/`.

### Estructura del repositorio

```
repo/
├── backend/
│   ├── Dockerfile
│   └── ...
├── frontend/
│   ├── Dockerfile
│   └── ...
├── k8s/
│   ├── namespace.yaml
│   ├── backend.yaml
│   └── frontend.yaml
├── ansible/
│   └── deploy-gke.yml
└── .github/
    └── workflows/
        └── build-deploy.yml
```

### `k8s/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ucm-master
```

### `k8s/backend.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agent-backend
  namespace: ucm-master
  labels:
    app: agent-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: agent-backend
  template:
    metadata:
      labels:
        app: agent-backend
    spec:
      containers:
        - name: agent-backend
          image: us-central1-docker.pkg.dev/PROJECT_ID/agent-repo/agent-backend:IMAGE_TAG
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 250m
              memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: agent-backend
  namespace: ucm-master
spec:
  selector:
    app: agent-backend
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

### `k8s/frontend.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agent-frontend
  namespace: ucm-master
  labels:
    app: agent-frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: agent-frontend
  template:
    metadata:
      labels:
        app: agent-frontend
    spec:
      containers:
        - name: agent-frontend
          image: us-central1-docker.pkg.dev/PROJECT_ID/agent-repo/agent-frontend:IMAGE_TAG
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 250m
              memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: agent-frontend
  namespace: ucm-master
spec:
  selector:
    app: agent-frontend
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
```

> **Nota:** Los placeholders `PROJECT_ID` e `IMAGE_TAG` se sustituyen dinámicamente durante el despliegue por Ansible con `sed`. No hace falta modificarlos.

---

## Paso 5 — Playbook de Ansible (Deploy en GKE)

Este playbook se ejecuta **directamente en el runner de GitHub Actions** (no en una VM externa). Ansible usa `gcloud` y `kubectl` (ya instalados en el runner tras la autenticación GCP) para desplegar los manifiestos en GKE.

### `ansible/deploy-gke.yml`

```yaml
---
- name: Deploy Crypto Chatbot to GKE
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    project_id: "{{ lookup('env', 'PROJECT_ID') }}"
    region: "us-central1"
    cluster_name: "test-cluster-a"
    registry_base: "us-central1-docker.pkg.dev/{{ project_id }}/agent-repo"
    namespace: "ucm-master"
    k8s_dir: "{{ playbook_dir }}/../k8s"
    # image_tag se pasa como --extra-vars

  tasks:

    # ─────────────────────────────────────
    # Obtener credenciales del clúster GKE
    # ─────────────────────────────────────
    - name: Obtener credenciales de GKE
      command: >
        gcloud container clusters get-credentials {{ cluster_name }}
          --region {{ region }}
          --project {{ project_id }}
      changed_when: false

    # ─────────────────────────────────────
    # Crear namespace (idempotente)
    # ─────────────────────────────────────
    - name: Aplicar namespace
      command: kubectl apply -f {{ k8s_dir }}/namespace.yaml
      changed_when: false

    # ─────────────────────────────────────
    # Preparar manifiestos con imagen real
    # ─────────────────────────────────────
    - name: Crear directorio temporal para manifiestos
      tempfile:
        state: directory
        suffix: k8s-deploy
      register: tmp_dir

    - name: Procesar backend.yaml con imagen correcta
      shell: |
        sed -e 's|PROJECT_ID|{{ project_id }}|g' \
            -e 's|IMAGE_TAG|{{ image_tag }}|g' \
            {{ k8s_dir }}/backend.yaml > {{ tmp_dir.path }}/backend.yaml

    - name: Procesar frontend.yaml con imagen correcta
      shell: |
        sed -e 's|PROJECT_ID|{{ project_id }}|g' \
            -e 's|IMAGE_TAG|{{ image_tag }}|g' \
            {{ k8s_dir }}/frontend.yaml > {{ tmp_dir.path }}/frontend.yaml

    # ─────────────────────────────────────
    # Desplegar en GKE
    # ─────────────────────────────────────
    - name: Desplegar backend
      command: kubectl apply -f {{ tmp_dir.path }}/backend.yaml
      register: backend_deploy

    - name: Desplegar frontend
      command: kubectl apply -f {{ tmp_dir.path }}/frontend.yaml
      register: frontend_deploy

    - name: Mostrar resultado del despliegue
      debug:
        msg:
          - "Backend: {{ backend_deploy.stdout }}"
          - "Frontend: {{ frontend_deploy.stdout }}"

    # ─────────────────────────────────────
    # Verificar estado del despliegue
    # (Autopilot puede tardar varios minutos
    #  en aprovisionar nodos la primera vez)
    # ─────────────────────────────────────
    - name: Esperar a que el backend esté disponible
      command: >
        kubectl get deployment agent-backend -n {{ namespace }}
          -o jsonpath='{.status.availableReplicas}'
      register: backend_status
      retries: 20
      delay: 30
      until: backend_status.stdout | default('0') | int >= 1
      changed_when: false

    - name: Esperar a que el frontend esté disponible
      command: >
        kubectl get deployment agent-frontend -n {{ namespace }}
          -o jsonpath='{.status.availableReplicas}'
      register: frontend_status
      retries: 20
      delay: 30
      until: frontend_status.stdout | default('0') | int >= 1
      changed_when: false

    - name: Obtener pods en ejecución
      command: kubectl get pods -n {{ namespace }} -o wide
      register: pod_status

    - name: Mostrar estado de pods
      debug:
        msg: "{{ pod_status.stdout_lines }}"

    # ─────────────────────────────────────
    # Obtener IP externa del frontend
    # ─────────────────────────────────────
    - name: Obtener servicio frontend
      command: kubectl get svc agent-frontend -n {{ namespace }} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
      register: frontend_ip
      retries: 10
      delay: 15
      until: frontend_ip.stdout | length > 0

    - name: Mostrar URL de acceso
      debug:
        msg: "🚀 Chatbot disponible en: http://{{ frontend_ip.stdout }}"

    # ─────────────────────────────────────
    # Limpieza
    # ─────────────────────────────────────
    - name: Eliminar directorio temporal
      file:
        path: "{{ tmp_dir.path }}"
        state: absent
```

### Notas del Playbook

- Se ejecuta en `localhost` (el propio runner de GitHub Actions), no necesita SSH a ninguna VM.
- Usa `gcloud container clusters get-credentials` para configurar `kubectl` contra el clúster GKE.
- Los manifiestos se procesan con `sed` para inyectar el `PROJECT_ID` y el `image_tag` (SHA del commit).
- `kubectl rollout status` asegura que el despliegue se ha completado correctamente antes de continuar.
- La verificación usa `retries: 20` con `delay: 30` (hasta 10 minutos), suficiente para que Autopilot aprovisione nodos en el primer despliegue.
- Se obtiene la IP externa del LoadBalancer con reintentos (puede tardar hasta ~2 minutos en asignarse).

---

## Paso 6 — Secretos en GitHub

Dentro del repositorio en GitHub → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**.

Configuramos los siguientes secretos:

| Secreto | Descripción | Valor |
|---|---|---|
| `GCP_SA_KEY` | JSON completo de `github-ci-key.json` | Contenido del archivo JSON |
| `PROJECT_ID` | ID del proyecto de GCP | `tu-project-id` |
| `GOOGLE_API_KEY` | Clave de API de Google (Gemini) | `AIza...` |

> ✅ **Simplificación:** Ya no necesitamos `CONTROL_HOST`, `CONTROL_HOST_USER`, `CONTROL_HOST_SSH_KEY` ni `CONTROL_HOST_PORT` porque no hay VM de control. Todo se ejecuta en el runner.

---

## Paso 7 — GitHub Actions Workflow

### `.github/workflows/build-deploy.yml`

```yaml
name: Build and Deploy Crypto Chatbot to GKE

on:
  push:
    branches: [main]

env:
  REGION: us-central1
  REGISTRY: us-central1-docker.pkg.dev
  REPO: agent-repo
  CLUSTER_NAME: test-cluster-a

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      # ─────────────────────────────────────
      # Checkout del código
      # ─────────────────────────────────────
      - name: Checkout repo
        uses: actions/checkout@v4

      # ─────────────────────────────────────
      # Crear .env del backend dinámicamente
      # ─────────────────────────────────────
      - name: Create backend .env
        run: |
          echo "GOOGLE_API_KEY=${{ secrets.GOOGLE_API_KEY }}" > backend/.env

      # ─────────────────────────────────────
      # Autenticación en GCP
      # ─────────────────────────────────────
      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          credentials_json: '${{ secrets.GCP_SA_KEY }}'

      - name: Set up Cloud SDK with kubectl
        uses: google-github-actions/setup-gcloud@v2
        with:
          install_components: 'kubectl,gke-gcloud-auth-plugin'

      - name: Configure Docker for Artifact Registry
        run: |
          gcloud auth configure-docker ${{ env.REGISTRY }}

      # ─────────────────────────────────────
      # Build y Push — Backend (Python)
      # ─────────────────────────────────────
      - name: Build backend image
        run: |
          docker build \
            -t ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-backend:${{ github.sha }} \
            -t ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-backend:latest \
            backend/

      - name: Push backend image
        run: |
          docker push ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-backend:${{ github.sha }}
          docker push ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-backend:latest

      # ─────────────────────────────────────
      # Build y Push — Frontend (Nginx)
      # ─────────────────────────────────────
      - name: Build frontend image
        run: |
          docker build \
            -t ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-frontend:${{ github.sha }} \
            -t ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-frontend:latest \
            frontend/

      - name: Push frontend image
        run: |
          docker push ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-frontend:${{ github.sha }}
          docker push ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-frontend:latest

      # ─────────────────────────────────────
      # Instalar Ansible y plugin de GKE
      # ─────────────────────────────────────
      - name: Install Ansible
        run: |
          pip install ansible

      # ─────────────────────────────────────
      # Deploy via Ansible → GKE
      # ─────────────────────────────────────
      - name: Deploy to GKE via Ansible
        env:
          PROJECT_ID: ${{ secrets.PROJECT_ID }}
          USE_GKE_GCLOUD_AUTH_PLUGIN: "True"
        run: |
          ansible-playbook ansible/deploy-gke.yml \
            --extra-vars "image_tag=${{ github.sha }}"
```

### Notas sobre el Workflow

- Se etiquetan las imágenes tanto con el SHA del commit (trazabilidad) como con `latest` (conveniencia).
- Se usa `google-github-actions/auth@v2` y `setup-gcloud@v2` para configurar el SDK completo.
- `setup-gcloud` instala `kubectl` y `gke-gcloud-auth-plugin` en un solo paso via `install_components`.
- Ansible se instala con `pip` directamente en el runner de ubuntu-latest.
- La variable de entorno `USE_GKE_GCLOUD_AUTH_PLUGIN=True` activa el nuevo mecanismo de autenticación.
- **No hay SSH** — todo se comunica vía API de Google Cloud.

---

## Paso 8 — Verificación y Troubleshooting

### Verificar el Pipeline Completo

**1. Verificar Artifact Registry (desde Cloud Shell):**

```bash
gcloud artifacts docker images list \
  $REGION-docker.pkg.dev/$PROJECT_ID/agent-repo
```

**2. Verificar el clúster y los recursos Kubernetes:**

```bash
# Conectar al clúster
gcloud container clusters get-credentials $CLUSTER_NAME \
  --region=$REGION --project=$PROJECT_ID

# Ver namespace
kubectl get ns ucm-master

# Ver deployments
kubectl get deployments -n ucm-master

# Ver pods
kubectl get pods -n ucm-master

# Ver servicios (aquí se obtiene la EXTERNAL-IP del frontend)
kubectl get svc -n ucm-master
```

**3. Ver logs de los pods:**

```bash
# Logs del backend
kubectl logs -l app=agent-backend -n ucm-master

# Logs del frontend
kubectl logs -l app=agent-frontend -n ucm-master
```

**4. Test funcional del chatbot:**

```bash
# Obtener la IP externa del frontend
EXTERNAL_IP=$(kubectl get svc agent-frontend -n ucm-master \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Test del backend (a través del frontend o port-forward)
kubectl port-forward svc/agent-backend 8000:80 -n ucm-master &
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"input": "What is the price of Bitcoin?"}'

# Acceder al frontend desde el navegador
echo "Frontend URL: http://$EXTERNAL_IP"
```


## Paso 9 — Limpieza de Recursos

Para evitar costes innecesarios, eliminar los recursos cuando ya no se necesiten:

```bash
# Eliminar el clúster GKE (elimina también todos los recursos dentro)
gcloud container clusters delete $CLUSTER_NAME \
  --region=$REGION --project=$PROJECT_ID --quiet

# Eliminar las imágenes del Artifact Registry
gcloud artifacts docker images delete \
  $REGION-docker.pkg.dev/$PROJECT_ID/agent-repo/agent-backend --quiet
gcloud artifacts docker images delete \
  $REGION-docker.pkg.dev/$PROJECT_ID/agent-repo/agent-frontend --quiet

# Eliminar el repositorio de Artifact Registry
gcloud artifacts repositories delete agent-repo \
  --location=$REGION --quiet

# Eliminar la service account
gcloud iam service-accounts delete \
  github-ci@${PROJECT_ID}.iam.gserviceaccount.com --quiet
```

---