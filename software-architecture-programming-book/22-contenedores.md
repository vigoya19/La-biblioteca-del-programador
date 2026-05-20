# Capítulo 22: Contenedores y Orquestación

> "Los contenedores no son virtualización ligera. Son procesos aislados con expectativas claras."

## 22.1 Docker: Más Allá de lo Básico

### Contenedor vs Máquina Virtual

```
VM:
┌──────────────────────────┐
│ App                      │
├──────────────────────────┤
│ Guest OS (kernel propio) │  ← Pesado, GBs
├──────────────────────────┤
│ Hypervisor               │
├──────────────────────────┤
│ Host OS                  │
└──────────────────────────┘

Contenedor:
┌──────────────────────────┐
│ App                      │
├──────────────────────────┤
│ Namespaces + Cgroups     │  ← Ligero, MBs
├──────────────────────────┤
│ Host OS (kernel compart) │
└──────────────────────────┘
```

**Un contenedor es un proceso aislado con su propio filesystem, red y límites de recursos.**

### Conceptos Clave
- **Imagen**: Build inmutable, capas stacked (Union FS).
- **Contenedor**: Instancia en ejecución de una imagen.
- **Registry**: Almacén de imágenes (Docker Hub, ECR, GCR).
- **Dockerfile**: Receta para construir la imagen.

### Buenas Prácticas de Dockerfile

```dockerfile
# ✅ BUENAS PRÁCTICAS
FROM node:20-alpine AS builder    # Imagen específica y ligera
WORKDIR /app
COPY package*.json ./             # Copiar deps primero (cacheo de capas)
RUN npm ci --only=production      # ci, no install (reproducible)
COPY . .                          # Luego el código
RUN npm run build

FROM node:20-alpine AS runtime    # Multi-stage build
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

USER node                         # No root en producción
EXPOSE 3000
HEALTHCHECK --interval=30s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/main.js"]
```

**Principios**:
1. **Multi-stage builds**: Separar build de runtime.
2. **Imágenes base mínimas**: Alpine, Distroless (sin shell, sin package manager).
3. **No root**: El contenedor no debe ejecutar como root.
4. **Una preocupación por contenedor**: Un proceso, un contenedor.
5. **Señales**: El proceso debe manejar SIGTERM para shutdown graceful.

## 22.2 Kubernetes (K8s)

> "Kubernetes es un sistema operativo distribuido. Trátalo como tal."

### Arquitectura Conceptual

```
┌──────────────────────────────────────────────────────┐
│ Control Plane (Master)                               │
│ ┌──────────┐ ┌──────────┐ ┌──────────────────────┐  │
│ │API Server│ │Scheduler │ │Controller Manager    │  │
│ └──────────┘ └──────────┘ │ etcd (almacén estado)│  │
│                            └──────────────────────┘  │
└──────────────────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Worker Node │  │ Worker Node │  │ Worker Node │
│             │  │             │  │             │
│ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │
│ │Pod Pod  │ │  │ │Pod Pod  │ │  │ │Pod Pod  │ │
│ │Pod      │ │  │ │Pod      │ │  │ │         │ │
│ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │
│ kubelet     │  │ kubelet     │  │ kubelet     │
│ kube-proxy  │  │ kube-proxy  │  │ kube-proxy  │
└─────────────┘  └─────────────┘  └─────────────┘
```

### Objetos Fundamentales

```yaml
# Pod: la unidad mínima (1+ contenedores que comparten network/storage)
apiVersion: v1
kind: Pod
metadata:
  name: mi-app
spec:
  containers:
    - name: app
      image: mi-app:1.0
      ports:
        - containerPort: 8080
      resources:
        requests: { memory: "128Mi", cpu: "250m" }
        limits:   { memory: "256Mi", cpu: "500m" }
      readinessProbe:
        httpGet: { path: /health, port: 8080 }
        initialDelaySeconds: 5
      livenessProbe:
        httpGet: { path: /health, port: 8080 }
        initialDelaySeconds: 15
```

### Abstracciones Clave de K8s

| Objeto | Propósito |
|--------|-----------|
| **Deployment** | Gestiona réplicas, rolling updates, rollbacks |
| **Service** | IP estable + DNS interno + load balancing |
| **Ingress** | Exposición HTTP(S) al exterior, routing por path/domain |
| **ConfigMap** | Configuración no sensible |
| **Secret** | Datos sensibles (codificados, no encriptados por defecto) |
| **StatefulSet** | Pods con identidad estable (BDs, Kafka) |
| **DaemonSet** | Un pod por nodo (logging, monitoreo) |
| **PersistentVolume** | Storage que sobrevive a pods |
| **HorizontalPodAutoscaler** | Escalar pods basado en CPU/memoria/métricas custom |
| **NetworkPolicy** | Firewall entre pods |
| **ServiceAccount** | Identidad para pods (con IRSA en AWS) |

### Resource Management

```
requests: Lo que el pod "reserva" (mínimo garantizado)
limits:   Lo que el pod "no puede exceder"

Si requests > capacidad del nodo → pod queda Pending
Si usage > limits → OOMKilled (CPU: throttled, no muere)
```

**Regla**: Siempre define requests y limits. Sin limits, un memory leak de un pod tumba el nodo entero.

## 22.3 Helm

Package manager para Kubernetes. Charts = templates + values.

```
┌─────────────────────────────────┐
│ Chart (mi-app/)                 │
│ ├── Chart.yaml (metadata)       │
│ ├── values.yaml (defaults)      │
│ ├── templates/                  │
│ │   ├── deployment.yaml         │
│ │   ├── service.yaml            │
│ │   └── ingress.yaml            │
│ └── charts/ (dependencias)      │
└─────────────────────────────────┘

helm install mi-app ./mi-app -f values-prod.yaml
```

## 22.4 Estrategia de Deploy en Kubernetes

| Estrategia | Cómo Funciona |
|-----------|--------------|
| **RollingUpdate** (default) | Reemplaza pods gradualmente. Sin downtime si está bien configurado. |
| **Recreate** | Elimina todos los pods viejos, crea los nuevos. Downtime. |
| **Blue/Green** | Despliega versión nueva paralela, cambia tráfico de golpe vía Service/Label. |
| **Canary** | Tráfico gradual a nueva versión. Con Ingress Controller (Nginx, Istio). |

### RollingUpdate fino:
```yaml
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1         # Pods extra durante update
      maxUnavailable: 0   # Ningún pod fuera de servicio
  minReadySeconds: 10     # Esperar a que el pod esté realmente listo
```

---

> **Reflexión del capítulo**: Kubernetes es increíblemente poderoso e increíblemente complejo. No lo adoptes porque está de moda. Adóptalo cuando la complejidad de gestionar contenedores manualmente supere la complejidad de aprender K8s. Y cuando lo hagas, invierte en el equipo: K8s mal configurado es más peligroso que no tener K8s.

---

← [Capítulo anterior](21-cloud.md) | [Inicio](README.md) | [Capítulo siguiente →](23-serverless.md)
