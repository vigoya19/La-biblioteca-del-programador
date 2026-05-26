# Capítulo 13: Ejercicios Prácticos Integradores

> "La teoría sin práctica es entretenimiento intelectual. La práctica sin teoría es superstición. Este capítulo une ambas."
> — Inspirado en el espíritu DevOps

Este capítulo es fundamentalmente diferente a los anteriores. No es una referencia teórica: es un **taller guiado** donde construirás, paso a paso y desde cero, una aplicación web completa desplegada en producción sobre Docker. Cada ejercicio incluye el **objetivo concreto**, las **instrucciones paso a paso**, y una **checklist de verificación** para que confirmes que tu implementación funciona antes de avanzar.

El proyecto que construirás —**TasksFlow**— es una aplicación de gestión de tareas con backend en Node.js, frontend en Nginx, base de datos PostgreSQL, caché Redis y un proxy reverso. Lo diseñarás con Dockerfile multi-stage, lo orquestarás con Compose para desarrollo y con Kubernetes para producción, le añadirás CI/CD con GitHub Actions, monitoreo con Prometheus + Grafana, y aplicarás hardening de seguridad.

Al finalizar este capítulo, habrás integrado todos los conocimientos del libro en una experiencia práctica real.

---

## Definición del Proyecto: TasksFlow

**TasksFlow** es un gestor de tareas (To-Do) tipo Kanban con las siguientes características técnicas:

- **Backend**: Node.js + Fastify + PostgreSQL
- **Frontend**: HTML/CSS/JS vanilla servido por Nginx
- **Caché**: Redis para sesiones y rate limiting
- **Proxy reverso**: Nginx como API Gateway
- **CI/CD**: GitHub Actions con build multi-arquitectura
- **Orquestación**: Docker Compose (dev) + Kubernetes (prod)
- **Monitoreo**: Prometheus + Grafana + cAdvisor

### Requisitos previos

Asegúrate de tener instalado antes de comenzar:

```bash
docker --version      # >= 24.0
docker compose version # >= 2.20
node --version         # >= 20 LTS
```

Crea el directorio raíz del proyecto:

```bash
mkdir -p ~/tasksflow
cd ~/tasksflow
```

---

## Ejercicio 1: Dockerfile Multi-Stage — Backend en Node.js

### Objetivo

Construir una imagen de producción para el backend de TasksFlow utilizando multi-stage builds, con una imagen final **alpine, non-root y menor de 150 MB**.

### Paso 1.1: Crear la estructura del backend

```bash
mkdir -p backend/src
cd backend
npm init -y
npm install fastify pg redis
```

Crea `backend/src/server.js`:

```javascript
const fastify = require("fastify")({ logger: true });
const { Pool } = require("pg");
const redis = require("redis");

// ── PostgreSQL ──
const pool = new Pool({
  host: process.env.DB_HOST || "localhost",
  port: parseInt(process.env.DB_PORT || "5432"),
  database: process.env.DB_NAME || "tasksflow",
  user: process.env.DB_USER || "tasksflow",
  password: process.env.DB_PASSWORD || "tasksflow"
});

// ── Redis ──
const redisClient = redis.createClient({
  url: `redis://${process.env.REDIS_HOST || "localhost"}:${process.env.REDIS_PORT || "6379"}`
});
redisClient.connect().catch(err => fastify.log.warn("Redis no disponible:", err.message));

// ── Health check ──
fastify.get("/health", async () => ({ status: "ok", uptime: process.uptime() }));

// ── CRUD de tareas ──
fastify.get("/api/tasks", async () => {
  const { rows } = await pool.query(
    "SELECT id, title, status, priority, created_at FROM tasks ORDER BY created_at DESC"
  );
  return rows;
});

fastify.post("/api/tasks", async (request) => {
  const { title, priority = "media" } = request.body;
  const { rows } = await pool.query(
    "INSERT INTO tasks (title, status, priority) VALUES ($1, 'pendiente', $2) RETURNING id, title, status, priority, created_at",
    [title, priority]
  );
  return rows[0];
});

fastify.patch("/api/tasks/:id", async (request) => {
  const { id } = request.params;
  const { status } = request.body;
  const { rows } = await pool.query(
    "UPDATE tasks SET status = $1 WHERE id = $2 RETURNING id, title, status, priority, created_at",
    [status, id]
  );
  if (!rows.length) return fastify.httpErrors.notFound("Tarea no encontrada");
  return rows[0];
});

fastify.delete("/api/tasks/:id", async (request) => {
  const { id } = request.params;
  await pool.query("DELETE FROM tasks WHERE id = $1", [id]);
  return { deleted: true };
});

// ── Arranque ──
const start = async () => {
  try {
    // Crear tabla si no existe
    await pool.query(`
      CREATE TABLE IF NOT EXISTS tasks (
        id SERIAL PRIMARY KEY,
        title VARCHAR(255) NOT NULL,
        status VARCHAR(20) DEFAULT 'pendiente',
        priority VARCHAR(10) DEFAULT 'media',
        created_at TIMESTAMPTZ DEFAULT NOW()
      )
    `);

    const port = parseInt(process.env.PORT || "3000");
    await fastify.listen({ port, host: "0.0.0.0" });
    fastify.log.info(`TasksFlow API corriendo en puerto ${port}`);
  } catch (err) {
    fastify.log.error(err);
    process.exit(1);
  }
};

start();
```

### Paso 1.2: Escribir el Dockerfile multi-stage

Crea `backend/Dockerfile`:

```dockerfile
# ── Stage 1: Dependencies ──
FROM node:20-alpine@sha256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0 AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

# ── Stage 2: Production ──
FROM node:20-alpine@sha256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0 AS production

# Crear usuario no-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copiar dependencias desde stage 1
COPY --from=deps /app/node_modules ./node_modules
COPY package*.json ./
COPY src/ ./src/

# Configurar ownership
RUN chown -R appuser:appgroup /app

# Cambiar a usuario no-root
USER appuser

# Health check: verificar que la API responde
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

EXPOSE 3000
CMD ["node", "src/server.js"]
```

### Paso 1.3: Crear .dockerignore

Crea `backend/.dockerignore`:

```
node_modules
npm-debug.log
.env
.git
.gitignore
*.md
.vscode
```

### Paso 1.4: Construir y verificar la imagen

```bash
docker build -t tasksflow-api:1.0.0 ./backend

# Verificar tamaño
docker images tasksflow-api
# REPOSITORY      TAG       IMAGE ID       CREATED         SIZE
# tasksflow-api   1.0.0     abc123def456   2 seconds ago   145MB

# Verificar que corre como non-root
docker run --rm tasksflow-api:1.0.0 whoami
# appuser
```

### ✅ Verificación del Ejercicio 1

- [ ] La imagen se construye sin errores con `docker build`
- [ ] El tamaño de la imagen es menor a 200 MB (`docker images`)
- [ ] El contenedor ejecuta procesos como `appuser`, NO como `root`
- [ ] `docker run --rm tasksflow-api:1.0.0 node -e "console.log('OK')"` imprime OK
- [ ] El HEALTHCHECK está definido: `docker inspect tasksflow-api:1.0.0 | jq '.[0].Config.Healthcheck'`

### 🎯 Tarea para el lector

Elimina el HEALTHCHECK del Dockerfile, reconstruye la imagen, y ejecuta `docker run -d --name no-hc tasksflow-api:1.0.0`. Observa con `docker ps` que el status nunca muestra `(healthy)`. Vuelve a añadir el HEALTHCHECK y compara la diferencia.

---

## Ejercicio 2: Frontend + Nginx en Docker

### Objetivo

Construir un frontend HTML/CSS/JS vanilla empaquetado con Nginx en una imagen Alpine de menos de 20 MB, sirviendo como Single Page Application con proxy reverso hacia la API.

### Paso 2.1: Crear el frontend

```bash
mkdir -p ~/tasksflow/frontend
```

Crea `frontend/index.html`:

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>TasksFlow — Gestor de Tareas</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <div id="app">
    <header>
      <h1>📋 TasksFlow</h1>
      <p class="subtitle">Gestión de tareas con Docker</p>
    </header>

    <main>
      <form id="task-form">
        <input type="text" id="task-input" placeholder="¿Qué necesitas hacer?" required autofocus>
        <select id="task-priority">
          <option value="media">Prioridad media</option>
          <option value="alta">🔥 Alta</option>
          <option value="baja">🟢 Baja</option>
        </select>
        <button type="submit">Añadir Tarea</button>
      </form>

      <div id="stats">
        <span id="total-tasks">0 tareas</span>
        <span id="pending-tasks">0 pendientes</span>
        <span id="done-tasks">0 completadas</span>
      </div>

      <div id="task-list"></div>
    </main>

    <footer>
      <p>TasksFlow v1.0 — Desplegado con Docker | <span id="api-status">API: 🔴</span></p>
    </footer>
  </div>
  <script src="app.js"></script>
</body>
</html>
```

Crea `frontend/styles.css`:

```css
* { margin: 0; padding: 0; box-sizing: border-box; }
body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; background: #f0f2f5; color: #1a1a2e; }
#app { max-width: 800px; margin: 0 auto; padding: 2rem 1rem; }
header { text-align: center; margin-bottom: 2rem; }
header h1 { font-size: 2.5rem; color: #0f3460; }
.subtitle { color: #666; margin-top: 0.25rem; }

#task-form { display: flex; gap: 0.5rem; margin-bottom: 1.5rem; background: white; padding: 1rem; border-radius: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
#task-form input { flex: 1; padding: 0.75rem 1rem; border: 2px solid #e0e0e0; border-radius: 8px; font-size: 1rem; transition: border-color 0.2s; }
#task-form input:focus { outline: none; border-color: #0f3460; }
#task-form select { padding: 0.75rem; border: 2px solid #e0e0e0; border-radius: 8px; font-size: 0.9rem; }
#task-form button { padding: 0.75rem 1.5rem; background: #0f3460; color: white; border: none; border-radius: 8px; font-size: 1rem; font-weight: 600; cursor: pointer; transition: background 0.2s; }
#task-form button:hover { background: #16213e; }

#stats { display: flex; gap: 1rem; margin-bottom: 1rem; }
#stats span { background: white; padding: 0.5rem 1rem; border-radius: 8px; font-size: 0.85rem; font-weight: 600; color: #666; box-shadow: 0 1px 4px rgba(0,0,0,0.05); }

.task-card { background: white; padding: 1rem 1.25rem; margin-bottom: 0.5rem; border-radius: 10px; display: flex; align-items: center; gap: 1rem; box-shadow: 0 1px 4px rgba(0,0,0,0.05); transition: transform 0.15s; }
.task-card:hover { transform: translateY(-1px); }
.task-card.completed { opacity: 0.6; }
.task-card.completed .task-title { text-decoration: line-through; }
.task-status { width: 24px; height: 24px; border-radius: 50%; border: 2px solid #ccc; cursor: pointer; display: flex; align-items: center; justify-content: center; transition: all 0.2s; flex-shrink: 0; }
.task-status:hover { border-color: #0f3460; }
.task-card.completed .task-status { background: #4caf50; border-color: #4caf50; color: white; }
.task-title { flex: 1; font-size: 0.95rem; }
.task-priority { font-size: 0.75rem; padding: 0.15rem 0.5rem; border-radius: 4px; font-weight: 600; }
.task-priority.alta { background: #ffe0e0; color: #c62828; }
.task-priority.media { background: #fff3e0; color: #e65100; }
.task-priority.baja { background: #e8f5e9; color: #2e7d32; }
.task-delete { background: none; border: none; color: #ccc; cursor: pointer; font-size: 1.2rem; padding: 0.25rem; transition: color 0.2s; }
.task-delete:hover { color: #c62828; }

.empty-state { text-align: center; padding: 3rem 1rem; color: #999; }
.empty-state .icon { font-size: 3rem; margin-bottom: 0.5rem; }

footer { text-align: center; margin-top: 3rem; color: #999; font-size: 0.8rem; }
#api-status { font-weight: 600; }

@media (max-width: 600px) {
  #task-form { flex-direction: column; }
  #stats { flex-direction: column; }
}
```

Crea `frontend/app.js`:

```javascript
const API_BASE = window.location.hostname === "localhost"
  ? "http://localhost:3000"
  : "/api";

const apiStatus = document.getElementById("api-status");
const taskForm = document.getElementById("task-form");
const taskInput = document.getElementById("task-input");
const taskPriority = document.getElementById("task-priority");
const taskList = document.getElementById("task-list");
const totalTasks = document.getElementById("total-tasks");
const pendingTasks = document.getElementById("pending-tasks");
const doneTasks = document.getElementById("done-tasks");

async function checkHealth() {
  try {
    const res = await fetch(`${API_BASE.replace("/api", "")}/health`);
    if (res.ok) {
      apiStatus.textContent = "API: 🟢";
      apiStatus.style.color = "#4caf50";
      return true;
    }
  } catch {}
  apiStatus.textContent = "API: 🔴";
  apiStatus.style.color = "#c62828";
  return false;
}

async function loadTasks() {
  try {
    const res = await fetch(`${API_BASE}/tasks`);
    const tasks = await res.json();
    renderTasks(tasks);
    updateStats(tasks);
  } catch {
    taskList.innerHTML = '<div class="empty-state"><div class="icon">⚠️</div><p>Error al cargar tareas. ¿Está corriendo la API?</p></div>';
  }
}

function renderTasks(tasks) {
  if (!tasks.length) {
    taskList.innerHTML = '<div class="empty-state"><div class="icon">📝</div><p>No hay tareas. ¡Crea la primera!</p></div>';
    return;
  }

  taskList.innerHTML = tasks.map(task => `
    <div class="task-card ${task.status === "completada" ? "completed" : ""}">
      <div class="task-status" onclick="toggleTask('${task.id}', '${task.status}')">
        ${task.status === "completada" ? "✓" : ""}
      </div>
      <span class="task-title">${escapeHtml(task.title)}</span>
      <span class="task-priority ${task.priority}">${task.priority}</span>
      <button class="task-delete" onclick="deleteTask('${task.id}')" title="Eliminar">✕</button>
    </div>
  `).join("");
}

function updateStats(tasks) {
  totalTasks.textContent = `${tasks.length} tarea${tasks.length !== 1 ? "s" : ""}`;
  pendingTasks.textContent = `${tasks.filter(t => t.status !== "completada").length} pendientes`;
  doneTasks.textContent = `${tasks.filter(t => t.status === "completada").length} completadas`;
}

function escapeHtml(text) {
  const div = document.createElement("div");
  div.textContent = text;
  return div.innerHTML;
}

taskForm.addEventListener("submit", async (e) => {
  e.preventDefault();
  const title = taskInput.value.trim();
  const priority = taskPriority.value;
  if (!title) return;

  await fetch(`${API_BASE}/tasks`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ title, priority })
  });

  taskInput.value = "";
  taskInput.focus();
  loadTasks();
});

window.toggleTask = async (id, currentStatus) => {
  const newStatus = currentStatus === "completada" ? "pendiente" : "completada";
  await fetch(`${API_BASE}/tasks/${id}`, {
    method: "PATCH",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ status: newStatus })
  });
  loadTasks();
};

window.deleteTask = async (id) => {
  await fetch(`${API_BASE}/tasks/${id}`, { method: "DELETE" });
  loadTasks();
};

// Inicializar
checkHealth();
loadTasks();
setInterval(checkHealth, 30000);
```

### Paso 2.2: Crear configuración de Nginx

Crea `frontend/nginx.conf`:

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Gzip
    gzip on;
    gzip_types text/css application/javascript text/html;
    gzip_min_length 1000;

    # Seguridad básica
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "no-referrer" always;

    # Proxy reverso hacia la API
    location /api/ {
        proxy_pass http://api:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Health check
    location /health {
        proxy_pass http://api:3000/health;
        proxy_http_version 1.1;
    }

    # SPA: todas las rutas no encontradas van a index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # No loguear favicon
    location = /favicon.ico {
        access_log off;
        log_not_found off;
    }
}
```

### Paso 2.3: Crear el Dockerfile del frontend

Crea `frontend/Dockerfile`:

```dockerfile
FROM nginx:1.27-alpine@sha256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0

# Eliminar contenido default
RUN rm -rf /usr/share/nginx/html/*

# Copiar assets del frontend
COPY index.html styles.css app.js /usr/share/nginx/html/

# Copiar configuración de Nginx
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Non-root
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    chown -R nginx:nginx /var/cache/nginx && \
    chown -R nginx:nginx /var/log/nginx && \
    chown -R nginx:nginx /etc/nginx/conf.d && \
    touch /var/run/nginx.pid && \
    chown -R nginx:nginx /var/run/nginx.pid

USER nginx

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:80/ || exit 1

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Paso 2.4: Construir y verificar

```bash
docker build -t tasksflow-frontend:1.0.0 ./frontend

# Verificar tamaño
docker images tasksflow-frontend
# REPOSITORY            TAG       SIZE
# tasksflow-frontend    1.0.0     18MB

# Probar que sirve
docker run -d --name frontend-test -p 8080:80 tasksflow-frontend:1.0.0
curl -s http://localhost:8080/ | head -5
docker rm -f frontend-test
```

### ✅ Verificación del Ejercicio 2

- [ ] La imagen pesa menos de 25 MB (`docker images tasksflow-frontend`)
- [ ] Nginx sirve el HTML correctamente en `http://localhost:8080`
- [ ] El contenedor corre como usuario `nginx`, no `root`: `docker run --rm tasksflow-frontend:1.0.0 whoami`
- [ ] Los headers de seguridad están presentes: `curl -I http://localhost:8080/ | grep -i x-`

### 🎯 Tarea para el lector

Añade una página 404 personalizada (`404.html`) a la imagen y configura Nginx para que la sirva con `error_page 404 /404.html;`. Verifica que navegar a una ruta inexistente muestra tu página personalizada.

---

## Ejercicio 3: Docker Compose — Stack de Desarrollo Completo

### Objetivo

Orquestar el stack completo (frontend, backend, PostgreSQL, Redis) con Docker Compose para desarrollo, usando bind mounts para hot reload y health checks para orden de arranque.

### Paso 3.1: Script de inicialización de BD

Crea `~/tasksflow/db/init.sql`:

```sql
CREATE TABLE IF NOT EXISTS tasks (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    status VARCHAR(20) DEFAULT 'pendiente',
    priority VARCHAR(10) DEFAULT 'media',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Datos de ejemplo
INSERT INTO tasks (title, status, priority) VALUES
    ('Configurar Docker Compose', 'completada', 'alta'),
    ('Crear el frontend de TasksFlow', 'completada', 'alta'),
    ('Añadir tests al backend', 'pendiente', 'media'),
    ('Configurar CI/CD con GitHub Actions', 'pendiente', 'alta'),
    ('Escribir documentación de la API', 'pendiente', 'baja')
ON CONFLICT DO NOTHING;
```

### Paso 3.2: Crear docker-compose.yml

Crea `~/tasksflow/docker-compose.yml`:

```yaml
services:
  # ── PostgreSQL ──
  db:
    image: postgres:16-alpine
    container_name: tasksflow-db
    environment:
      POSTGRES_USER: tasksflow
      POSTGRES_PASSWORD: tasksflow
      POSTGRES_DB: tasksflow
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    networks:
      - tasksflow-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U tasksflow -d tasksflow"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
    restart: unless-stopped

  # ── Redis ──
  cache:
    image: redis:7-alpine
    container_name: tasksflow-cache
    ports:
      - "6379:6379"
    volumes:
      - redisdata:/data
    networks:
      - tasksflow-net
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped

  # ── Backend API ──
  api:
    build:
      context: ./backend
      dockerfile: Dockerfile
      target: deps  # Usar stage de dependencias para desarrollo (más rápido)
    container_name: tasksflow-api
    environment:
      PORT: 3000
      DB_HOST: db
      DB_PORT: 5432
      DB_NAME: tasksflow
      DB_USER: tasksflow
      DB_PASSWORD: tasksflow
      REDIS_HOST: cache
      REDIS_PORT: 6379
    ports:
      - "3000:3000"
    volumes:
      - ./backend/src:/app/src  # Hot reload: editar código y reiniciar
    command: ["node", "--watch", "src/server.js"]  # --watch disponible en Node 22+
    networks:
      - tasksflow-net
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    restart: unless-stopped

  # ── Frontend Nginx ──
  web:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: tasksflow-web
    ports:
      - "8080:80"
    volumes:
      - ./frontend/index.html:/usr/share/nginx/html/index.html
      - ./frontend/styles.css:/usr/share/nginx/html/styles.css
      - ./frontend/app.js:/usr/share/nginx/html/app.js
      - ./frontend/nginx.conf:/etc/nginx/conf.d/default.conf
    networks:
      - tasksflow-net
    depends_on:
      - api
    restart: unless-stopped

volumes:
  pgdata:
  redisdata:

networks:
  tasksflow-net:
    driver: bridge
```

### Paso 3.3: Levantar el stack

```bash
docker compose up -d

# Verificar que todos los servicios están healthy
docker compose ps
# NAME              STATUS
# tasksflow-db      Up (healthy)
# tasksflow-cache   Up (healthy)
# tasksflow-api     Up
# tasksflow-web     Up
```

### Paso 3.4: Probar la aplicación

```bash
# 1. Verificar API
curl -s http://localhost:3000/health | jq
# { "status": "ok", "uptime": 12.5 }

# 2. Crear una tarea
curl -s -X POST http://localhost:3000/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Aprender Docker Compose","priority":"alta"}' | jq

# 3. Listar tareas
curl -s http://localhost:3000/api/tasks | jq '.[].title'
# "Configurar Docker Compose"
# "Crear el frontend de TasksFlow"
# ...

# 4. Abrir el frontend
open http://localhost:8080   # macOS
# xdg-open http://localhost:8080  # Linux
```

### Paso 3.5: Demostrar hot reload

Modifica `frontend/styles.css`, cambia el color de fondo de `body` de `#f0f2f5` a `#e8f5e9`. Recarga el navegador. El cambio debe ser instantáneo porque el archivo está bind-mounted.

### ✅ Verificación del Ejercicio 3

- [ ] `docker compose up -d` levanta los 4 servicios sin errores
- [ ] `docker compose ps` muestra todos los servicios como `Up`
- [ ] `curl http://localhost:3000/health` devuelve `{"status":"ok"}`
- [ ] `curl http://localhost:3000/api/tasks` lista las tareas de ejemplo
- [ ] El frontend en `http://localhost:8080` se conecta a la API y muestra tareas
- [ ] Crear una tarea desde el formulario web la añade a la lista instantáneamente
- [ ] Marcar una tarea como completada la tacha en la UI

### 🎯 Tarea para el lector

Añade un contenedor **Adminer** al `docker-compose.yml` para gestionar PostgreSQL visualmente. Debe exponerse en el puerto 8081 y conectarse automáticamente a la base de datos `db`. Pista: usa la imagen `adminer:latest` y la variable de entorno `ADMINER_DEFAULT_SERVER: db`.

---

## Ejercicio 4: Seguridad — Hardening del Contenedor

### Objetivo

Aplicar hardening al contenedor del backend: eliminar capabilities innecesarias, montar sistema de archivos como read-only, limitar recursos, y añadir un perfil seccomp.

### Paso 4.1: Crear perfil seccomp personalizado

Crea `~/tasksflow/security/seccomp-profile.json`:

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_AARCH64"],
  "syscalls": [
    { "names": ["accept", "accept4", "bind", "close", "connect", "epoll_ctl", "epoll_pwait", "epoll_wait", "eventfd2", "exit", "exit_group", "fchmod", "fchown", "fstat", "futex", "getcwd", "getdents64", "getegid", "geteuid", "getgid", "getpid", "getppid", "getrandom", "gettid", "getuid", "ioctl", "lseek", "madvise", "mmap", "mprotect", "munmap", "nanosleep", "newfstatat", "openat", "prctl", "pread64", "pwrite64", "read", "readlink", "recvfrom", "recvmsg", "rt_sigaction", "rt_sigprocmask", "rt_sigreturn", "sched_getaffinity", "sched_yield", "sendmsg", "sendto", "set_robust_list", "set_tid_address", "setsockopt", "sigaltstack", "socket", "tgkill", "uname", "write", "writev"], "action": "SCMP_ACT_ALLOW" }
  ]
}
```

### Paso 4.2: Ejecutar el backend con hardening

Primero, para desarrollo puro eliminamos el hot reload (que necesita escritura):

```bash
# Detener el stack
docker compose down

# Ejecutar backend con hardening
docker run -d \
  --name tasksflow-api-secure \
  --network tasksflow_tasksflow-net \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt=no-new-privileges:true \
  --security-opt=seccomp=./security/seccomp-profile.json \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64M \
  --memory=256m \
  --memory-swap=256m \
  --cpus=1.0 \
  --pids-limit=50 \
  -e PORT=3000 \
  -e DB_HOST=db \
  -e DB_NAME=tasksflow \
  -e DB_USER=tasksflow \
  -e DB_PASSWORD=tasksflow \
  -p 3000:3000 \
  tasksflow-api:1.0.0
```

### Paso 4.3: Verificar las restricciones

```bash
# 1. Verificar capabilities: debe estar casi vacío
docker exec tasksflow-api-secure cat /proc/1/status | grep -i cap
# CapInh: 0000000000000000
# CapPrm: 0000000000000400  ← solo NET_BIND_SERVICE (bit 10)
# CapEff: 0000000000000400
# CapBnd: 0000000000000400

# 2. Intentar escribir en / (debe fallar)
docker exec tasksflow-api-secure touch /test.txt
# touch: cannot touch '/test.txt': Read-only file system

# 3. Intentar montar algo (debe fallar por no-new-privileges + cap-drop)
docker exec tasksflow-api-secure mount
# mount: permission denied (are you root?)

# 4. Verificar límites de memoria
docker stats --no-stream tasksflow-api-secure
# MEM USAGE / LIMIT
# 45MiB / 256MiB

# 5. Verificar que el health check responde
curl -s http://localhost:3000/health
```

### Paso 4.4: Escaneo de vulnerabilidades

```bash
# Escanear con Docker Scout (necesario: docker login)
docker scout quickview tasksflow-api:1.0.0

# Escanear con Trivy (alternativa open source)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image tasksflow-api:1.0.0 --severity CRITICAL,HIGH

# Inspeccionar capas con dive
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive tasksflow-api:1.0.0
```

### ✅ Verificación del Ejercicio 4

- [ ] El contenedor arranca exitosamente con todas las restricciones aplicadas
- [ ] Las capabilities del proceso son mínimas (solo NET_BIND_SERVICE)
- [ ] El sistema de archivos raíz es read-only
- [ ] No se puede escalar a root ni montar sistemas de archivos
- [ ] La API responde correctamente en `/health`
- [ ] El escaneo de vulnerabilidades no muestra CRITICALs

### 🎯 Tarea para el lector

Actualiza el `docker-compose.yml` para que el servicio `api` en producción use las mismas restricciones de seguridad usando la clave `security_opt`, `cap_drop`, `cap_add`, `read_only`, `tmpfs` y `deploy.resources.limits`. Crea un override file `docker-compose.prod.yml`.

---

## Ejercicio 5: CI/CD con GitHub Actions

### Objetivo

Configurar un pipeline de CI/CD que ejecute tests, construya la imagen multi-arquitectura (amd64 + arm64), la publique en GitHub Container Registry (GHCR), y despliegue en un VPS.

### Paso 5.1: Añadir tests al backend

Crea `backend/src/server.test.js`:

```javascript
const { describe, it, before, after } = require("node:test");
const assert = require("node:assert/strict");

describe("TasksFlow API — Tests de Integración", () => {
  let baseUrl;
  let createdTaskId;

  before(async () => {
    baseUrl = `http://localhost:${process.env.PORT || 3000}`;
  });

  it("GET /health debe devolver status ok", async () => {
    const res = await fetch(`${baseUrl}/health`);
    const data = await res.json();
    assert.equal(data.status, "ok");
    assert.ok(typeof data.uptime === "number");
  });

  it("POST /api/tasks debe crear una tarea", async () => {
    const res = await fetch(`${baseUrl}/api/tasks`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ title: "Tarea de prueba CI", priority: "alta" })
    });
    const data = await res.json();
    assert.equal(data.title, "Tarea de prueba CI");
    assert.equal(data.status, "pendiente");
    assert.ok(data.id);
    createdTaskId = data.id;
  });

  it("GET /api/tasks debe listar tareas", async () => {
    const res = await fetch(`${baseUrl}/api/tasks`);
    const tasks = await res.json();
    assert.ok(Array.isArray(tasks));
    assert.ok(tasks.length > 0);
  });

  it("PATCH /api/tasks/:id debe actualizar estado", async () => {
    const res = await fetch(`${baseUrl}/api/tasks/${createdTaskId}`, {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ status: "completada" })
    });
    const data = await res.json();
    assert.equal(data.status, "completada");
  });

  it("DELETE /api/tasks/:id debe eliminar", async () => {
    const res = await fetch(`${baseUrl}/api/tasks/${createdTaskId}`, {
      method: "DELETE"
    });
    const data = await res.json();
    assert.equal(data.deleted, true);
  });
});
```

Añade al `package.json` del backend:

```json
{
  "scripts": {
    "test": "node --test src/server.test.js",
    "start": "node src/server.js"
  }
}
```

Prueba localmente:

```bash
cd ~/tasksflow/backend
docker compose -f ../docker-compose.yml up -d db cache  # solo servicios de apoyo
sleep 5
npm test
```

### Paso 5.2: Crear el workflow de GitHub Actions

Crea `~/tasksflow/.github/workflows/ci-cd.yml`:

```yaml
name: CI/CD — TasksFlow

on:
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME_API: ${{ github.repository }}/tasksflow-api
  IMAGE_NAME_WEB: ${{ github.repository }}/tasksflow-web

jobs:
  # ── Tests ──
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: tasksflow
          POSTGRES_PASSWORD: tasksflow
          POSTGRES_DB: tasksflow
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: "npm"
          cache-dependency-path: backend/package-lock.json

      - name: Install dependencies
        run: npm ci
        working-directory: backend

      - name: Run tests
        run: npm test
        working-directory: backend
        env:
          PORT: 3000
          DB_HOST: localhost
          DB_NAME: tasksflow
          DB_USER: tasksflow
          DB_PASSWORD: tasksflow
          REDIS_HOST: localhost

  # ── Build & Push ──
  build-push:
    needs: test
    if: github.event_name != 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    strategy:
      matrix:
        service: [api, web]

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ matrix.service == 'api' && env.IMAGE_NAME_API || env.IMAGE_NAME_WEB }}
          tags: |
            type=ref,event=branch
            type=ref,event=tag
            type=sha,prefix=,format=short
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push (${{ matrix.service }})
        uses: docker/build-push-action@v6
        with:
          context: ./${{ matrix.service == 'api' && 'backend' || 'frontend' }}
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ── Deploy (VPS con Docker Compose) ──
  deploy:
    needs: build-push
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to VPS via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /opt/tasksflow
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
            docker compose -f docker-compose.prod.yml pull
            docker compose -f docker-compose.prod.yml up -d --remove-orphans
            docker image prune -af
```

### Paso 5.3: Configurar secrets en GitHub

En tu repositorio de GitHub, ve a **Settings → Secrets and variables → Actions** y añade:

- `VPS_HOST`: IP de tu servidor
- `VPS_USER`: usuario SSH (ej. `deploy`)
- `VPS_SSH_KEY`: clave privada SSH

### ✅ Verificación del Ejercicio 5

- [ ] El workflow de CI/CD existe en `.github/workflows/ci-cd.yml`
- [ ] Al hacer push a `main`, el job `test` se ejecuta y pasa
- [ ] Los tests de integración del backend pasan todos
- [ ] El job `build-push` construye imágenes multi-arquitectura (amd64 + arm64)
- [ ] Las imágenes se publican en `ghcr.io/<usuario>/tasksflow-api` y `tasksflow-web`
- [ ] Al crear un tag `v1.0.0`, se dispara el deploy automático

### 🎯 Tarea para el lector

Añade un paso de **linting** al workflow usando ESLint. Crea una configuración `.eslintrc.json` en `backend/` y un paso `Run linter` antes de `Run tests`. Si el linting falla, el pipeline debe detenerse.

---

## Ejercicio 6: Kubernetes — Despliegue en Producción

### Objetivo

Desplegar TasksFlow en un clúster local de Kubernetes (Minikube o Kind), creando Deployments, Services, ConfigMaps, Secrets y PersistentVolumeClaims.

### Paso 6.1: Crear clúster local

```bash
# Opción A: Minikube
minikube start --cpus=4 --memory=4096 --driver=docker

# Opción B: Kind
kind create cluster --name tasksflow --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
      - containerPort: 30081
        hostPort: 30081
EOF
```

### Paso 6.2: Crear manifiestos de Kubernetes

Crea el directorio `~/tasksflow/k8s/` y los siguientes archivos:

`k8s/namespace.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tasksflow
```

`k8s/configmap.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: tasksflow
data:
  DB_HOST: "postgres-svc"
  DB_PORT: "5432"
  DB_NAME: "tasksflow"
  REDIS_HOST: "redis-svc"
  REDIS_PORT: "6379"
  PORT: "3000"
```

`k8s/secrets.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: tasksflow
type: Opaque
stringData:
  DB_USER: tasksflow
  DB_PASSWORD: tasksflow
  POSTGRES_PASSWORD: tasksflow
```

`k8s/postgres.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: tasksflow
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: tasksflow
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: api-config
                  key: DB_NAME
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "tasksflow", "-d", "tasksflow"]
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "tasksflow", "-d", "tasksflow"]
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc

---
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: tasksflow
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
  type: ClusterIP
```

`k8s/redis.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: tasksflow
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
          livenessProbe:
            exec:
              command: ["redis-cli", "ping"]
            initialDelaySeconds: 5
            periodSeconds: 10

---
apiVersion: v1
kind: Service
metadata:
  name: redis-svc
  namespace: tasksflow
spec:
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
  type: ClusterIP
```

`k8s/api.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: tasksflow
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: ghcr.io/<tu-usuario>/tasksflow-api:latest
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: api-config
          env:
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_PASSWORD
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10

---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: tasksflow
spec:
  selector:
    app: api
  ports:
    - port: 3000
      targetPort: 3000
  type: ClusterIP
```

`k8s/web.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: tasksflow
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: ghcr.io/<tu-usuario>/tasksflow-web:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "16Mi"
              cpu: "50m"
            limits:
              memory: "32Mi"
              cpu: "100m"
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 15

---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: tasksflow
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
  type: NodePort
```

### Paso 6.3: Desplegar en Kubernetes

```bash
# Aplicar todos los manifiestos
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secrets.yaml
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/redis.yaml
kubectl apply -f k8s/api.yaml
kubectl apply -f k8s/web.yaml

# Verificar despliegue
kubectl get all -n tasksflow
# NAME                            READY   STATUS    RESTARTS   AGE
# pod/api-7d9f8c6b5-abcde         1/1     Running   0          30s
# pod/api-7d9f8c6b5-fghij         1/1     Running   0          30s
# pod/api-7d9f8c6b5-klmno         1/1     Running   0          30s
# pod/postgres-5c8b9d7f6-xxxxx    1/1     Running   0          45s
# pod/redis-84f6c9d8b-yyyyy       1/1     Running   0          40s
# pod/web-6d9f8c7b5-zzzzz         1/1     Running   0          25s

# Ver logs de la API
kubectl logs -n tasksflow -l app=api --tail=20

# Acceder al frontend (Minikube)
minikube service web-svc -n tasksflow

# Escalar la API a 5 réplicas
kubectl scale deployment api -n tasksflow --replicas=5

# Ejecutar migración de BD
kubectl exec -n tasksflow deployment/api -- node -e "
  const { Pool } = require('pg');
  const pool = new Pool({
    host: process.env.DB_HOST,
    database: process.env.DB_NAME,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD
  });
  pool.query('CREATE TABLE IF NOT EXISTS tasks (id SERIAL PRIMARY KEY, title VARCHAR(255), status VARCHAR(20), priority VARCHAR(10), created_at TIMESTAMPTZ DEFAULT NOW())').then(() => { console.log('Migración OK'); pool.end(); });
"
```

### ✅ Verificación del Ejercicio 6

- [ ] `kubectl get pods -n tasksflow` muestra todos los pods `Running`
- [ ] `kubectl logs -n tasksflow -l app=api` no muestra errores
- [ ] La API responde: `kubectl port-forward -n tasksflow svc/api-svc 3000:3000` y luego `curl localhost:3000/health`
- [ ] El frontend es accesible vía NodePort o port-forward
- [ ] Al escalar la API con `--replicas=5`, Kubernetes distribuye los pods

### 🎯 Tarea para el lector

Añade un **Ingress** (usando NGINX Ingress Controller) para exponer tanto el frontend como la API bajo un solo dominio: `tasksflow.local` → frontend, `tasksflow.local/api` → API. Habilita TLS con un certificado autofirmado.

---

## Ejercicio 7: Monitoreo con Prometheus + Grafana + cAdvisor

### Objetivo

Desplegar un stack de monitoreo con Prometheus, Grafana y cAdvisor usando Docker Compose, creando dashboards para TasksFlow.

### Paso 7.1: Crear configuración de Prometheus

Crea `~/tasksflow/monitoring/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: "tasksflow-api"
    metrics_path: /metrics
    static_configs:
      - targets: ["api:3000"]
```

### Paso 7.2: Añadir métricas al backend

Instala `prom-client` en el backend:

```bash
cd ~/tasksflow/backend
npm install prom-client
```

Añade al inicio de `backend/src/server.js`:

```javascript
const promClient = require("prom-client");
const register = new promClient.Registry();
promClient.collectDefaultMetrics({ register });

const httpRequestsTotal = new promClient.Counter({
  name: "http_requests_total",
  help: "Total HTTP requests",
  labelNames: ["method", "route", "status"],
  registers: [register]
});

// Middleware de métricas
fastify.addHook("onRequest", async (request) => {
  request.startTime = Date.now();
});

fastify.addHook("onResponse", async (request, reply) => {
  const duration = (Date.now() - request.startTime) / 1000;
  httpRequestsTotal.inc({
    method: request.method,
    route: request.routeOptions.url || request.url,
    status: reply.statusCode
  });
});

// Endpoint de métricas
fastify.get("/metrics", async () => {
  return register.metrics();
});
```

Reconstruye la imagen:

```bash
docker compose build api
```

### Paso 7.3: Crear docker-compose para monitoreo

Crea `~/tasksflow/monitoring/docker-compose.monitoring.yml`:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=15d"
    ports:
      - "9090:9090"
    networks:
      - tasksflow_tasksflow-net
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-piechart-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    ports:
      - "3001:3000"
    networks:
      - tasksflow_tasksflow-net
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    devices:
      - /dev/kmsg
    ports:
      - "8082:8080"
    networks:
      - tasksflow_tasksflow-net
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"
    ports:
      - "9100:9100"
    networks:
      - tasksflow_tasksflow-net
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:

networks:
  tasksflow_tasksflow-net:
    external: true
```

### Paso 7.4: Levantar el stack de monitoreo

```bash
# Asegurarse de que TasksFlow está corriendo
docker compose up -d

# Levantar monitoreo
docker compose -f monitoring/docker-compose.monitoring.yml up -d

# Verificar
docker compose -f monitoring/docker-compose.monitoring.yml ps
# NAME             STATUS
# prometheus       Up
# grafana          Up
# cadvisor         Up
# node-exporter    Up

# Acceder a Prometheus
open http://localhost:9090    # Probar query: rate(http_requests_total[5m])

# Acceder a Grafana
open http://localhost:3001    # usuario: admin, contraseña: admin
# Configurar datasource: http://prometheus:9090
# Importar dashboard Docker: ID 179 (Docker monitoring)
```

### ✅ Verificación del Ejercicio 7

- [ ] Prometheus está accesible en `http://localhost:9090` con targets UP
- [ ] Grafana está accesible en `http://localhost:3001`
- [ ] cAdvisor muestra métricas de contenedores en `http://localhost:8082`
- [ ] La métrica `http_requests_total` aparece en Prometheus tras hacer peticiones a la API
- [ ] Puedes crear un gráfico en Grafana de `rate(http_requests_total[5m])` por endpoint

### 🎯 Tarea para el lector

Crea un dashboard de Grafana específico para TasksFlow que muestre: (1) Requests por segundo por endpoint, (2) Latencia p99, (3) Número de tareas creadas vs completadas en las últimas 24h, (4) Uso de CPU y memoria de cada contenedor del stack.

---

## Retos Adicionales para el Lector

Si has completado todos los ejercicios, aquí tienes retos avanzados:

1. **🔥 Zero-Downtime Deploy en Kubernetes**: Implementa una estrategia de rolling update con `maxSurge` y `maxUnavailable`, y demuestra que la API sigue respondiendo durante el despliegue usando `while true; do curl -s localhost:3000/health; sleep 1; done`.

2. **🔐 Autenticación JWT**: Añade autenticación al backend con `fastify-jwt`. Protege las rutas `/api/tasks/*` para que requieran un token válido. Actualiza el frontend con un formulario de login.

3. **📦 Multi-Arquitectura Real**: Configura un builder remoto en una Raspberry Pi (o una VM ARM64) y construye la imagen para `linux/arm64` de forma nativa (no emulada con QEMU).

4. **🚨 AlertManager**: Configura AlertManager con Prometheus para enviar alertas a Discord/Slack cuando: la API tenga más de 5% de errores 5xx, el uso de CPU supere el 80%, o un contenedor lleve más de 1 minuto sin healthy.

5. **🌐 Traefik como Ingress**: Sustituye Nginx por Traefik como proxy reverso + ingress controller en Kubernetes, con obtención automática de certificados Let's Encrypt.

6. **🧪 Chaos Engineering**: Usa `chaos-mesh` o `pumba` para matar aleatoriamente containers y verificar que el clúster se auto-recupera.

7. **📊 ELK Stack**: Añade centralización de logs con Elasticsearch + Logstash + Kibana. Configura los contenedores para enviar logs estructurados (JSON) vía driver `gelf` o `fluentd`.

---

## Resumen del Capítulo

En este capítulo has construido, paso a paso, una aplicación completa desplegada en producción:

- **Ejercicio 1**: Dockerfile multi-stage para Node.js con non-root, HEALTHCHECK y tamaño optimizado.
- **Ejercicio 2**: Frontend vanilla empaquetado con Nginx Alpine en menos de 20 MB, con proxy reverso.
- **Ejercicio 3**: Docker Compose con 4 servicios, health checks, dependencias condicionales, bind mounts para desarrollo.
- **Ejercicio 4**: Hardening de seguridad: capabilities, seccomp, read-only FS, resource limits, escaneo de vulnerabilidades con Trivy y dive.
- **Ejercicio 5**: CI/CD con GitHub Actions: tests, build multi-arquitectura (amd64 + arm64), push a GHCR, deploy automático.
- **Ejercicio 6**: Kubernetes: Deployments, Services, ConfigMaps, Secrets, PVCs, probes, y escalado.
- **Ejercicio 7**: Monitoreo con Prometheus + Grafana + cAdvisor, métricas personalizadas desde el backend.

Cada ejercicio ha reforzado conceptos de los capítulos anteriores en un contexto práctico real. Los retos adicionales te permiten seguir explorando áreas avanzadas del ecosistema Docker.

> *"Un profesional no es quien nunca falla, sino quien ha fallado tantas veces en entornos controlados que ya sabe exactamente qué hacer cuando falla en producción."*

---

← [Capítulo anterior](capitulo-12-produccion.md) | [Inicio](../README.md) | [Capítulo siguiente →](../apendices/apendice-a-quick-reference.md)
