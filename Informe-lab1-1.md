# Informe de Laboratorio DevOps: Práctica de GitOps con Kubernetes y ArgoCD

Este laboratorio tiene como objetivos:
- Introducir a los principios de GitOps en kubernetes.
- Configuración dee Minikube y ArgoCD
- Configuraciones declaratibas de kubernetes.
- Uso de overlays
- Automatización de despliegues y promoción mediante pull requests


## Fase 1: Configuración del Entorno

### 1.1 Instalación de Herramientas Base
Durante la practica he trabajado desde el entorno wsl2 con dockers.
se instalaron las siguientes herramientas:
- **Minikube**: Crea un clúster Kubernetes local para desarrollo y pruebas
- **kubectl**: Cliente oficial para interactuar con la API de Kubernetes

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && sudo install minikube-linux-amd64 /usr/local/bin/minikube
sudo apt-get update && sudo apt-get install -y kubectl
```
### 1.2 Inicialización del Clúster Kubernetes
  
**Comando ejecutado:**
```bash
minikube start --driver=docker --cpus=4 --memory=4096 --disk-size=20g
```

**Explicación detallada:**
- `--driver=docker`: Usa Docker como hypervisor (más eficiente que VM)
- `--cpus=4 --memory=4096`: Asigna recursos suficientes para ArgoCD y aplicaciones
- `--disk-size=20g`: Espacio para imágenes de contenedores y datos persistentes

**Resultado:** Clúster Kubernetes corriendo localmente

### 1.3 Habilitación de Addons Esenciales

**Comandos ejecutados:**
```bash
minikube addons enable metrics-server
minikube addons enable ingress
```

**¿Para qué sirven?**
- **metrics-server**: Recolecta métricas de CPU/memoria de pods y nodos
- **ingress**: Controlador para gestionar tráfico HTTP/HTTPS hacia servicios

---

## Fase 2: Instalación y Configuración de ArgoCD

### 2.1 Despliegue de ArgoCD

**Comandos ejecutados:**
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

**¿Qué hace cada comando?**
1. Crea un namespace dedicado para aislar ArgoCD
2. Despliega todos los componentes de ArgoCD (servidor, controlador, repo-server, etc.)
3. Espera hasta que el servidor esté listo antes de continuar

**Componentes instalados:**
- `argocd-server`: Interfaz web y API REST
- `argocd-application-controller`: Gestiona el estado de las aplicaciones
- `argocd-repo-server`: Clona y procesa repositorios Git
- `argocd-dex-server`: Servidor de autenticación
- `argocd-redis`: Cache para mejorar rendimiento

### 2.2 Acceso a ArgoCD

**Comandos ejecutados:**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 > /dev/null 2>&1 &
# Note: This runs in the background. To stop it later, find the process:
# ps aux | grep "port-forward"
# kill <PID>
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

**Explicación:**
- Port-forward permite acceder a ArgoCD desde localhost:8080
- La contraseña inicial se genera automáticamente y se almacena en un secreto
- Credenciales: `admin` / `1DRl3XUEVzeSRA47` 

Ya podemos aceder a argoCD en `localhost:8080`, y relizar el cambio de contraseña
---

## Fase 3: Estructura del Repositorio GitOps
Creamos el repositorio `k8s-gitops-demo` en github y lo clonamos en local para empezar a trabajar sobre ál.

### 3.1 Arquitectura Base con Kustomize

La estructura implementada sigue el patrón "base + overlays", con unas carpetas commo las mostradas a continuación:

```
k8s-gitops-demo/
├── base/                    # Configuración común a todos los entornos
│   ├── deployment.yaml      # Definición de la aplicación
│   ├── service.yaml         # Exposición del servicio
│   └── kustomization.yaml   # Configuración de Kustomize
├── overlays/                # Personalizaciones por entorno
│   ├── dev/
│   ├── staging/
│   └── prod/
└── argocd-app-*.yaml       # Definiciones de aplicaciones ArgoCD
```

### 3.2 Análisis Detallado de Archivos Base

#### `base/deployment.yaml`
```yaml
apiVersion: apps/v1  # Versión de API para Deployments
kind: Deployment  # Tipo de recurso Kubernetes (deployment es conjunto de pods, otros posibles son Service, ConfigMap, Secret)
metadata:
  name: webapp  # Nombre único del deployment
spec:
  replicas: 2  # Número de pods idénticos a ejecutar
  template:  # Plantilla para crear los pods
    spec:
      containers:
      - name: webapp  # Nombre del contenedor, para identificarlo en otros recursos
        image: nginxdemos/hello:latest  # Imagen Docker (evita :latest en prod)
        resources:
          requests:  # Recursos GARANTIZADOS por Kubernetes
            memory: "64Mi"  # RAM mínima reservada
            cpu: "50m"  # CPU mínima reservada (50m = 5% de 1 core)
          limits:  # Recursos MÁXIMOS permitidos
            memory: "128Mi"  # Si excede: pod es terminado (OOMKilled)
            cpu: "100m"  # Si excede: es ralentizado (throttled)
        livenessProbe:  # ¿Está vivo? Si falla → reinicia el contenedor
          httpGet:
            path: /  # Endpoint a verificar (debe devolver 200-399)
            port: 80  # Puerto donde verificar
        readinessProbe:  # ¿Listo para tráfico? Si falla → quita del load balancer
          httpGet:
            path: /  # Endpoint para verificar disponibilidad
            port: 80  # Puerto donde verificar
```

**Elementos clave:**
- **Image**: `nginxdemos/hello` - aplicación demo que muestra información del contenedor
- **Resources**: Limita uso de CPU/memoria para evitar consumo excesivo
- **Probes**: Verifican que la aplicación esté viva y lista para recibir tráfico

#### `base/service.yaml`
```yaml
apiVersion: v1  # API básica de Kubernetes (v1, no apps/v1)
kind: Service  # Tipo de recurso: punto de acceso a los pods
metadata:
  name: webapp-service  # Nombre único del servicio
spec:
  selector:  # Selector: encuentra los pods por sus etiquetas
    app: webapp  # Busca todos los pods con label "app=webapp"
  ports:
  - port: 80  # Puerto del SERVICE (donde otros pods se conectan)
    targetPort: 80  # Puerto del POD (donde escucha el contenedor)
  type: ClusterIP  # Tipo: acceso solo INTERNO al cluster (por defecto)
```
Este Service es el "balanceador de carga interno" que permite acceder a los pods del Deployment de forma estable. 

**Función:**
- Expone los pods internamente en el clúster
- `ClusterIP`: Solo accesible desde dentro del clúster
- El selector conecta con pods que tengan la etiqueta `app: webapp`

#### `base/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1  # API de Kustomize
kind: Kustomization  # Tipo: archivo de configuración Kustomize
resources:  # Lista de archivos YAML a gestionar
  - deployment.yaml  # Incluye el archivo deployment
  - service.yaml     # Incluye el archivo service
commonLabels:  # Etiquetas que se añaden AUTOMÁTICAMENTE a todos los recursos
  app: webapp           # Añade label "app: webapp" a deployment Y service
  managed-by: argocd    # Añade label "managed-by: argocd" a ambos
```

**Propósito:**
- Agrupa recursos relacionados
- Añade etiquetas comunes automáticamente
- Base para personalizaciones específicas por entorno

### 3.3 Overlays por Entorno

#### Ejemplo: `overlays/dev/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: dev
namePrefix: dev-
bases:
  - ../../base # Carga las kustomizations
patchesStrategicMerge:
  - replica-patch.yaml #Le aplica este parche
commonLabels:
  environment: dev
```

**Transformaciones aplicadas:**
1. **namespace: dev** → Despliega en namespace específico
2. **namePrefix: dev-** → Recursos se llaman `dev-webapp`, `dev-webapp-service`
3. **patchesStrategicMerge** → Aplica modificaciones específicas
4. **commonLabels** → Añade etiqueta de entorno

#### `overlays/dev/replica-patch.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 1  # Solo 1 réplica en desarrollo
```

**Lógica por entorno:**
- **dev**: 1 réplica (recursos limitados, pruebas rápidas)
- **staging**: 2 réplicas (simula producción)
- **prod**: 3 réplicas (alta disponibilidad)

>En este punto se hizo commit y push al repo

---

## 🚀 Fase 4: Aplicaciones ArgoCD

### 4.1 Definición de Aplicación para Desarrollo

#### `argocd-app-dev.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp-dev
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/ignrui/k8s-gitops-demo
    targetRevision: main
    path: overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true       # Borra recursos eliminados del Git
      selfHeal: true    # Revierte cambios manuales
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
```

**Características clave:**
- **Sincronización automática**: Detecta cambios en Git y los aplica automáticamente
- **prune: true**: Si borramos un archivo de Git, ArgoCD borra el recurso del clúster
- **selfHeal: true**: Si modificamos algo manualmente en Kubernetes, ArgoCD lo revierte
- **CreateNamespace=true**: Crea el namespace automáticamente si no existe

### 4.2 Aplicación para Producción

La aplicación de producción (`argocd-app-prod.yaml`) difiere en:
- **Sincronización manual**: Requiere aprobación explícita para cambios
- **path: overlays/prod**: Usa configuración específica de producción
- **namespace: prod**: Despliega en entorno aislado

**¿Por qué manual en producción?**
- Mayor control sobre cambios críticos
- Permite revisiones y validaciones antes del despliegue
- Reduce riesgo de interrupciones de servicio

---

## ⚙️ Fase 5: Despliegue y Operación


## 🔄 Fase 5: Experimentar GitOps en Acción

### 5.1 Escalado de réplicas en Dev y sincronización automática

**Acción:** Se edita `overlays/dev/replica-patch.yaml` para escalar a 2 réplicas y se hace commit/push a GitHub.

**Comandos:**
```bash
cat > overlays/dev/replica-patch.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 2  # Scale up dev environment
EOF

git add overlays/dev/replica-patch.yaml
git commit -m "Scale dev environment to 2 replicas"
git push origin main
```

**Explicación:** Al modificar el número de réplicas y subir el cambio a Git (y crear el pull request con merge a main), ArgoCD sincroniza automáticamente el clúster. De manera automática, se crea un segundo pod en el namespace `dev` que podemos observar en la web de argoCD o en bash.

**Verificación:**
```bash
kubectl get pods -n dev

#salida : 
NAME                          READY   STATUS    RESTARTS      AGE
dev-webapp-7c9c8c8499-67cv4   1/1     Running   0             59s
dev-webapp-7c9c8c8499-xfr62   1/1     Running   1 (18m ago)   4d22h
```
Deberías ver dos pods corriendo para `dev-webapp`.


---

### 5.2 Prueba de auto-reparación GitOps

**Acción:** Se escala manualmente el deployment a 0 réplicas (fuera de Git) y se observa cómo ArgoCD restaura el estado declarado en Git.

**Comandos:**
```bash
kubectl scale deployment dev-webapp -n dev --replicas=0
kubectl get pods -n dev -w

#Salida
NAME                          READY   STATUS        RESTARTS      AGE
dev-webapp-7c9c8c8499-67cv4   1/1     Terminating   0             2m13s
dev-webapp-7c9c8c8499-xfr62   1/1     Terminating   1 (19m ago)   4d22h
dev-webapp-7c9c8c8499-ww67f   0/1     Pending       0             0s
dev-webapp-7c9c8c8499-ww67f   0/1     Pending       0             0s
dev-webapp-7c9c8c8499-btz5z   0/1     Pending       0             0s
dev-webapp-7c9c8c8499-btz5z   0/1     Pending       0             0s
dev-webapp-7c9c8c8499-ww67f   0/1     ContainerCreating   0             0s
dev-webapp-7c9c8c8499-btz5z   0/1     ContainerCreating   0             0s
dev-webapp-7c9c8c8499-xfr62   0/1     Completed           1 (19m ago)   4d22h
dev-webapp-7c9c8c8499-67cv4   0/1     Completed           0             2m13s
dev-webapp-7c9c8c8499-67cv4   0/1     Completed           0             2m14s
dev-webapp-7c9c8c8499-67cv4   0/1     Completed           0             2m14s
dev-webapp-7c9c8c8499-xfr62   0/1     Completed           1 (19m ago)   4d22h
dev-webapp-7c9c8c8499-xfr62   0/1     Completed           1 (19m ago)   4d22h
dev-webapp-7c9c8c8499-ww67f   0/1     Running             0             3s
dev-webapp-7c9c8c8499-btz5z   0/1     Running             0             5s
dev-webapp-7c9c8c8499-ww67f   1/1     Running             0             9s
dev-webapp-7c9c8c8499-btz5z   1/1     Running             0             11s
```

**Explicación:** Al modificar el estado del clúster manualmente, ArgoCD detecta la desviación y restaura el número de réplicas a 2, cumpliendo el principio de GitOps: Git es la única fuente de verdad.

---

### 5.3 Agregar un ConfigMap y parametrización por entorno

**Acción:** Se crea `base/configmap.yaml` con configuración genérica, se actualiza `base/kustomization.yaml` para incluir el ConfigMap y se modifica `base/deployment.yaml` para consumir el ConfigMap como variables de entorno.

**Comandos:**
```bash
cat > base/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config  # nombre del mapa para usar en deployment.yaml
data:
  APP_ENV: "base"
  LOG_LEVEL: "info"
  FEATURE_FLAG: "disabled"
EOF

cat > base/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
commonLabels:
  app: webapp
  managed-by: argocd
EOF

cat > base/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginxdemos/hello:latest
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: webapp-config  # Referencia al mapa creado
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
EOF
```

**Explicación:** El ConfigMap permite parametrizar la aplicación según el entorno. Se añade como recurso base y se referencia en el deployment para inyectar variables de entorno.

---

### 5.4 Configuración específica para Dev


**Acción:** Se crea `overlays/dev/configmap-patch.yaml` con valores específicos para dev y se actualiza `overlays/dev/kustomization.yaml` para aplicar el parche.

**Comandos:**
```bash
cat > overlays/dev/configmap-patch.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config  # emplea el kind y el name para saber donde a
data:
  APP_ENV: "development"
  LOG_LEVEL: "debug"
  FEATURE_FLAG: "enabled"
EOF

cat > overlays/dev/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: dev
namePrefix: dev-
bases:
  - ../../base
patchesStrategicMerge:
  - replica-patch.yaml
  - configmap-patch.yaml
commonLabels:
  environment: dev
EOF

git add .
git commit -m "Add ConfigMap with environment-specific configuration"
git push origin main
```

**Verificación:**
```bash
kubectl get configmaps -n dev
kubectl describe configmap dev-webapp-config -n dev
```
Deberiamos ver el ConfigMap con los valores específicos para dev.

---
## Parte 6: Promoción de Cambios
### 6.1 Promoción a Staging

Después de validar los cambios en dev, promovimos el ConfigMap a staging siguiendo los pasos de la guía:

**Comandos ejecutados:**
```bash
cp overlays/dev/configmap-patch.yaml overlays/staging/configmap-patch.yaml
cat > overlays/staging/configmap-patch.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  APP_ENV: "staging"
  LOG_LEVEL: "info"
  FEATURE_FLAG: "enabled"
EOF

cat > overlays/staging/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: staging
namePrefix: staging-
bases:
  - ../../base
patchesStrategicMerge:
  - replica-patch.yaml
  - configmap-patch.yaml
commonLabels:
  environment: staging
EOF

git checkout -b promote-to-staging
git add overlays/staging/
git commit -m "Promote ConfigMap changes to staging"
git push origin promote-to-staging
```

**Explicación:**
Se copia y adapta el parche de configuración para staging, se actualiza el kustomization, se crea una rama de promoción y se suben los cambios. En GitHub se crea y fusiona el Pull Request.

**Sincronización y descubrimiento del error:**
Tras hacer `git pull` en local y ejecutar:
```bash
kubectl get pods -n staging -w
kubectl get configmap -n staging
```
no aparecían recursos en staging. Descubrimos que faltaba aplicar el manifiesto de la aplicación ArgoCD para staging (`argocd-app-staging.yaml`).

**Corrección:**
Ejecutamos:
```bash
kubectl apply -f argocd-app-staging.yaml
```
Esto permitió que ArgoCD gestionara el entorno staging y sincronizara los recursos correctamente.

**Lección aprendida:**
Siempre hay que asegurarse de que los manifiestos de las aplicaciones ArgoCD estén aplicados antes de promover y verificar cambios en cada entorno.

A partir de aquí, continuamos con la validación y despliegue en staging según la guía.


### 6.2 Promoción a Producción — Primera fase (preparación y push de la rama)

**Acciones ejecutadas:**

1) Copiar el parche de `staging` a `prod` y ajustar los valores para producción:

```bash
cp overlays/staging/configmap-patch.yaml overlays/prod/configmap-patch.yaml
cat > overlays/prod/configmap-patch.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "warn"
  FEATURE_FLAG: "enabled"
EOF 
```

2) Actualizar el `kustomization` de `prod` para referenciar la base y los parches adecuados:

```bash
cat > overlays/prod/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: prod
namePrefix: prod-

bases:
  - ../../base

patchesStrategicMerge:
  - replica-patch.yaml
  - configmap-patch.yaml

commonLabels:
  environment: prod
EOF
```

3) Crear la rama de promoción, commitear y subirla al remoto:

```bash
git checkout -b promote-to-prod
git add overlays/prod/
git commit -m "Promote ConfigMap to production"
git push origin promote-to-prod
```

**Salida observada:**

- Rama creada localmente: `promote-to-prod`.
- Commit aplicado (2 archivos modificados / 1 creado).
- Push completado al remoto. GitHub ofreció la URL para crear el Pull Request:

  https://github.com/ignrui/k8s-gitops-demo/pull/new/promote-to-prod

**Explicación:**

En esta primera fase los cambios de configuración para `prod` se preparan y se colocan en una rama de promoción. No se realiza ninguna sincronización automática con el clúster en este paso: la rama solo contiene la propuesta de cambio que debe revisarse mediante un Pull Request y, una vez aprobada y fusionada a `main`, se procederá a la sincronización manual en ArgoCD (según la política de producción definida).

Siguiente acción: crear y revisar el Pull Request en GitHub; tras la fusión a `main` ejecutar la sincronización manual en ArgoCD para desplegar en producción.

### 6.2 Promoción a Producción — Sincronización manual y resultado

**Acciones ejecutadas:**

```bash
git checkout main
git pull origin main

# Production won't auto-sync, must approve manually
argocd app sync webapp-prod

# Verify
kubectl get configmap -n prod
kubectl describe configmap prod-webapp-config -n prod
```

**Salida observada:**

- `git checkout` y `git pull` confirmaron que `main` está actualizado.
- `argocd app sync webapp-prod` devolvió un error de autenticación por token expirado (respuesta JSON):

  {"level":"fatal","msg":"rpc error: code = Unauthenticated desc = invalid session: token has invalid claims: token is expired","time":"2025-11-05T20:13:51+01:00"}

- Comprobación con `kubectl` mostró que existe un `ConfigMap` en `prod` pero con los valores anteriores (no los valores de `production` recién preparados):

  NAME                 DATA   AGE
  kube-root-ca.crt     1      7d9h
  prod-webapp-config   3      2d10h

  Data
  ====
  FEATURE_FLAG:
  ----
  disabled
  LOG_LEVEL:
  ----
  info
  APP_ENV:
  ----
  base

**Explicación:**

El intento de sincronización manual con `argocd` falló por una sesión/credencial caducada en el cliente (`argocd` CLI). Por tanto, ArgoCD no recibió la orden de sincronizar desde la CLI y el clúster sigue mostrando el `ConfigMap` anterior (valores base). El push de la rama `promote-to-prod` ya existía en remoto, pero hasta que la rama no se fusione en `main` y se ejecute una sincronización manual (o la aplicación sea actualizada desde la UI/CLI con sesión válida), los cambios no se aplicarán en el clúster de producción.

**Acción recomendada inmediata:**

- Renovar la sesión de `argocd` (login) o usar la UI para realizar la sincronización manual, y volver a ejecutar `argocd app sync webapp-prod`. Tras una sincronización correcta, verificar con `kubectl describe configmap prod-webapp-config -n prod` que los valores `APP_ENV`, `LOG_LEVEL` y `FEATURE_FLAG` muestran los valores de `production`.

### Reintento: renovación de sesión y sincronización (salida real)

**Comandos ejecutados (sesión renovada con contraseña manual):**

```bash
argocd login localhost:8080 --username admin --password ******** --insecure
argocd app sync webapp-prod
kubectl describe configmap prod-webapp-config -n prod
```

**Salida observada:**

- Login: `'admin:login' logged in successfully` y `Context 'localhost:8080' updated`.
- `argocd app sync webapp-prod` devolvió un resumen de operación indicando que la aplicación `argocd/webapp-prod` fue sincronizada con éxito contra `main` (Sync Status: `Synced to main (b1cdbf7)`), con `Phase: Succeeded` y `Message: successfully synced (all tasks run)`.
- Detalle de recursos sincronizados (ejemplos):

  TIMESTAMP                  GROUP        KIND   NAMESPACE                  NAME    STATUS   HEALTH
  2025-11-05T20:18:15+01:00          ConfigMap        prod    prod-webapp-config    Synced
  2025-11-05T20:18:15+01:00            Service        prod   prod-webapp-service    Synced  Healthy
  2025-11-05T20:18:15+01:00   apps  Deployment        prod           prod-webapp    Synced  Healthy

- `kubectl describe configmap prod-webapp-config -n prod` mostró que el `ConfigMap` en `prod` aún contiene los valores anteriores (APP_ENV: `base`, LOG_LEVEL: `info`, FEATURE_FLAG: `disabled`).

**Interpretación y conclusión:**

- El login y la sincronización con ArgoCD fueron exitosos en la segunda tentativa: la operación de sincronización se ejecutó y ArgoCD informó que la aplicación quedó `Synced` y `Healthy` (operación finalizada correctamente).
- Comprobamos el estado de las aplicaciones con la CLI de ArgoCD (`argocd app list` y `argocd app get`) y mediante acceso en el navegador. Las tres aplicaciones (`dev`, `staging` y `prod`) aparecen con estado "Synced" y salud "Healthy". En `prod` la sincronización se ejecutó contra la revisión `b4a0099` y ArgoCD reportó el `ConfigMap` como `configured` mientras que el `Deployment` y el `Service` quedaron `unchanged` (no se detectaron diferencias que ArgoCD aplicara). Tras la verificación en el clúster, el `ConfigMap` en `prod` seguía mostrando los valores previos, por lo que investigamos el historial de Git y la definición declarativa para identificar por qué los cambios de configuración no se materializaron en los datos del `ConfigMap`.






## 🔚 Punto 10: Limpieza y desmontaje del entorno

Al finalizar las pruebas se procedió a limpiar el entorno local para dejar el sistema en un estado limpio. Los comandos ejecutados y la salida observada fueron los siguientes:

Comandos ejecutados:
```bash
pkill -f "port-forward"
# Delete ArgoCD applications
kubectl delete application webapp-dev -n argocd
kubectl delete application webapp-staging -n argocd
kubectl delete application webapp-prod -n argocd

# Delete namespaces
kubectl delete namespace dev staging prod

# Optional: Delete ArgoCD entirely
kubectl delete namespace argocd

# Stop Minikube
minikube stop

# Optional: Delete Minikube cluster (removes all data)
minikube delete
```

Salida observada (extracto):
```
application.argoproj.io "webapp-dev" deleted
[1]   Terminated              kubectl port-forward svc/dev-webapp-service 8081:80 -n dev > /tmp/portfwd-dev.log 2>&1
[2]-  Terminated              kubectl port-forward svc/staging-webapp-service 8082:80 -n staging > /tmp/portfwd-staging.log 2>&1
[3]+  Terminated              kubectl port-forward svc/prod-webapp-service 8083:80 -n prod > /tmp/portfwd-prod.log 2>&1
application.argoproj.io "webapp-staging" deleted
application.argoproj.io "webapp-prod" deleted
namespace "dev" deleted
namespace "staging" deleted
namespace "prod" deleted
namespace "argocd" deleted
✋  Stopping node "minikube"  ...
🛑  Powering off "minikube" via SSH ...
🛑  1 node stopped.
🔥  Deleting "minikube" in docker ...
🔥  Deleting container "minikube" ...
🔥  Removing /home/ctag/.minikube/machines/minikube ...
💀  Removed all traces of the "minikube" cluster.
```

Verificación y notas:
- Se confirmaron la terminación de los `port-forward` (mensajes de proceso terminado) y la eliminación de las Applications de ArgoCD.
- Los namespaces `dev`, `staging`, `prod` y `argocd` fueron eliminados correctamente.
- `minikube stop` y `minikube delete` detuvieron y eliminaron el clúster local, incluyendo los contenedores asociados.
- Esta limpieza elimina todos los recursos creados durante el laboratorio; si necesita conservar artefactos o logs, hay que exportarlos antes de ejecutar estos comandos.

Recomendación final: antes de ejecutar la eliminación del clúster en entornos de producción o con datos persistentes, confirmar copias de seguridad y exportar recursos críticos (manifiestos, volúmenes, imágenes) para evitar pérdidas de información.
