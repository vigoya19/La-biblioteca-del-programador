# Capítulo 16: Desarrollo Web y APIs

Go brilla en el desarrollo de servicios web y APIs. La libreria estandar incluye `net/http`, un servidor HTTP listo para produccion, y `database/sql` para acceso a bases de datos. Este capitulo cubre como construir APIs REST completas usando principalmente la libreria estandar.

---

## 16.1 net/http (stdlib)

### Servidor HTTP basico

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
)

func main() {
    // Handler como funcion
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Bienvenido a la API")
    })

    // Handler con metodo especifico (Go 1.22+)
    http.HandleFunc("GET /api/saludo", func(w http.ResponseWriter, r *http.Request) {
        nombre := r.URL.Query().Get("nombre")
        if nombre == "" {
            nombre = "Mundo"
        }
        fmt.Fprintf(w, "Hola, %s!", nombre)
    })

    log.Println("Servidor en :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Struct como handler

```go
package main

import (
    "encoding/json"
    "net/http"
)

type UsuarioHandler struct {
    servicio *ServicioUsuario
}

func (h *UsuarioHandler) Listar(w http.ResponseWriter, r *http.Request) {
    usuarios, err := h.servicio.Listar(r.Context())
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    responderJSON(w, http.StatusOK, usuarios)
}

func (h *UsuarioHandler) Crear(w http.ResponseWriter, r *http.Request) {
    var entrada CrearUsuarioInput
    if err := json.NewDecoder(r.Body).Decode(&entrada); err != nil {
        http.Error(w, "JSON invalido", http.StatusBadRequest)
        return
    }

    usuario, err := h.servicio.Crear(r.Context(), entrada)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    responderJSON(w, http.StatusCreated, usuario)
}

func (h *UsuarioHandler) Obtener(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id") // Go 1.22+
    usuario, err := h.servicio.ObtenerPorID(r.Context(), id)
    if err != nil {
        http.Error(w, "Usuario no encontrado", http.StatusNotFound)
        return
    }
    responderJSON(w, http.StatusOK, usuario)
}

func responderJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}
```

### Configurar servidor con timeouts

```go
package main

import (
    "net/http"
    "time"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", handler)

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    srv.ListenAndServe()
}
```

---

## 16.2 Middlewares

Los middlewares son funciones que envuelven handlers para agregar comportamiento transversal:

```go
package main

import (
    "context"
    "log"
    "net/http"
    "time"
)

type Middleware func(http.Handler) http.Handler

// Encadenar middlewares
func Encadenar(h http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

// Middleware de logging
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        inicio := time.Now()

        // Envolver ResponseWriter para capturar el status code
        wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
        next.ServeHTTP(wrapped, r)

        log.Printf("%s %s %d %v",
            r.Method, r.URL.Path, wrapped.statusCode, time.Since(inicio))
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

// Middleware de recuperacion de panics
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("PANIC: %v", err)
                http.Error(w, "error interno del servidor", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

// Middleware de CORS
func CORSMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }

        next.ServeHTTP(w, r)
    })
}

// Middleware de autenticacion
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "no autorizado", http.StatusUnauthorized)
            return
        }

        // Validar token...
        usuarioID := "123" // Extraido del token

        // Agregar al context
        ctx := context.WithValue(r.Context(), "usuarioID", usuarioID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Middleware de timeout
func TimeoutMiddleware(timeout time.Duration) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, cancel := context.WithTimeout(r.Context(), timeout)
            defer cancel()
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/api/usuarios", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte(`{"usuarios": []}`))
    })

    handler := Encadenar(mux,
        RecoveryMiddleware,
        LoggingMiddleware,
        CORSMiddleware,
        TimeoutMiddleware(30*time.Second),
    )

    http.ListenAndServe(":8080", handler)
}
```

---

## 16.3 APIs REST

### Diseno de una API REST completa

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "strconv"
)

type Tarea struct {
    ID        int    `json:"id"`
    Titulo    string `json:"titulo"`
    Completada bool   `json:"completada"`
}

type TareaHandler struct {
    tareas  []Tarea
    nextID  int
}

func (h *TareaHandler) Listar(w http.ResponseWriter, r *http.Request) {
    responderJSON(w, http.StatusOK, h.tareas)
}

func (h *TareaHandler) Crear(w http.ResponseWriter, r *http.Request) {
    var t Tarea
    if err := json.NewDecoder(r.Body).Decode(&t); err != nil {
        http.Error(w, "JSON invalido", http.StatusBadRequest)
        return
    }

    if t.Titulo == "" {
        http.Error(w, "titulo es requerido", http.StatusBadRequest)
        return
    }

    h.nextID++
    t.ID = h.nextID
    t.Completada = false
    h.tareas = append(h.tareas, t)

    responderJSON(w, http.StatusCreated, t)
}

func (h *TareaHandler) Obtener(w http.ResponseWriter, r *http.Request) {
    id, err := strconv.Atoi(r.PathValue("id"))
    if err != nil {
        http.Error(w, "ID invalido", http.StatusBadRequest)
        return
    }

    for _, t := range h.tareas {
        if t.ID == id {
            responderJSON(w, http.StatusOK, t)
            return
        }
    }
    http.Error(w, "tarea no encontrada", http.StatusNotFound)
}

func (h *TareaHandler) Actualizar(w http.ResponseWriter, r *http.Request) {
    id, err := strconv.Atoi(r.PathValue("id"))
    if err != nil {
        http.Error(w, "ID invalido", http.StatusBadRequest)
        return
    }

    var entrada Tarea
    if err := json.NewDecoder(r.Body).Decode(&entrada); err != nil {
        http.Error(w, "JSON invalido", http.StatusBadRequest)
        return
    }

    for i, t := range h.tareas {
        if t.ID == id {
            if entrada.Titulo != "" {
                h.tareas[i].Titulo = entrada.Titulo
            }
            h.tareas[i].Completada = entrada.Completada
            responderJSON(w, http.StatusOK, h.tareas[i])
            return
        }
    }
    http.Error(w, "tarea no encontrada", http.StatusNotFound)
}

func (h *TareaHandler) Eliminar(w http.ResponseWriter, r *http.Request) {
    id, err := strconv.Atoi(r.PathValue("id"))
    if err != nil {
        http.Error(w, "ID invalido", http.StatusBadRequest)
        return
    }

    for i, t := range h.tareas {
        if t.ID == id {
            h.tareas = append(h.tareas[:i], h.tareas[i+1:]...)
            w.WriteHeader(http.StatusNoContent)
            return
        }
    }
    http.Error(w, "tarea no encontrada", http.StatusNotFound)
}

func responderJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    if data != nil {
        json.NewEncoder(w).Encode(data)
    }
}

func main() {
    handler := &TareaHandler{}

    mux := http.NewServeMux()
    mux.HandleFunc("GET /api/tareas", handler.Listar)
    mux.HandleFunc("POST /api/tareas", handler.Crear)
    mux.HandleFunc("GET /api/tareas/{id}", handler.Obtener)
    mux.HandleFunc("PUT /api/tareas/{id}", handler.Actualizar)
    mux.HandleFunc("DELETE /api/tareas/{id}", handler.Eliminar)

    log.Println("API REST en :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### Versionado de API

```go
// Estrategia 1: version en la URL
mux.HandleFunc("GET /api/v1/usuarios", handlerV1.Listar)
mux.HandleFunc("GET /api/v2/usuarios", handlerV2.Listar)

// Estrategia 2: header de version
func VersionMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        version := r.Header.Get("API-Version")
        ctx := context.WithValue(r.Context(), "apiVersion", version)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### Paginacion

```go
type Paginacion struct {
    Pagina    int `json:"pagina"`
    PorPagina int `json:"por_pagina"`
    Total     int `json:"total"`
}

type RespuestaPaginada struct {
    Datos      interface{} `json:"datos"`
    Paginacion Paginacion  `json:"paginacion"`
}

func (h *TareaHandler) ListarPaginado(w http.ResponseWriter, r *http.Request) {
    pagina, _ := strconv.Atoi(r.URL.Query().Get("pagina"))
    porPagina, _ := strconv.Atoi(r.URL.Query().Get("por_pagina"))

    if pagina < 1 {
        pagina = 1
    }
    if porPagina < 1 || porPagina > 100 {
        porPagina = 10
    }

    inicio := (pagina - 1) * porPagina
    fin := inicio + porPagina
    if inicio > len(h.tareas) {
        inicio = len(h.tareas)
    }
    if fin > len(h.tareas) {
        fin = len(h.tareas)
    }

    responderJSON(w, http.StatusOK, RespuestaPaginada{
        Datos: h.tareas[inicio:fin],
        Paginacion: Paginacion{
            Pagina:    pagina,
            PorPagina: porPagina,
            Total:     len(h.tareas),
        },
    })
}
```

---

## 16.4 Bases de Datos (database/sql)

### Conexion y operaciones basicas

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "time"

    _ "github.com/lib/pq" // Driver PostgreSQL
)

type Usuario struct {
    ID        int
    Nombre    string
    Email     string
    CreadoEn  time.Time
}

type RepositorioUsuario struct {
    db *sql.DB
}

func NuevoRepositorioUsuario(dsn string) (*RepositorioUsuario, error) {
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, fmt.Errorf("error al abrir BD: %w", err)
    }

    // Configurar pool de conexiones
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(10)
    db.SetConnMaxLifetime(5 * time.Minute)

    // Verificar conexion
    if err := db.PingContext(context.Background()); err != nil {
        return nil, fmt.Errorf("error al conectar BD: %w", err)
    }

    return &RepositorioUsuario{db: db}, nil
}

func (r *RepositorioUsuario) Crear(ctx context.Context, u *Usuario) error {
    query := `
        INSERT INTO usuarios (nombre, email)
        VALUES ($1, $2)
        RETURNING id, creado_en`

    return r.db.QueryRowContext(ctx, query, u.Nombre, u.Email).
        Scan(&u.ID, &u.CreadoEn)
}

func (r *RepositorioUsuario) ObtenerPorID(ctx context.Context, id int) (*Usuario, error) {
    query := `SELECT id, nombre, email, creado_en FROM usuarios WHERE id = $1`

    u := &Usuario{}
    err := r.db.QueryRowContext(ctx, query, id).
        Scan(&u.ID, &u.Nombre, &u.Email, &u.CreadoEn)

    if err == sql.ErrNoRows {
        return nil, fmt.Errorf("usuario %d no encontrado", id)
    }
    if err != nil {
        return nil, fmt.Errorf("error al buscar usuario: %w", err)
    }
    return u, nil
}

func (r *RepositorioUsuario) Listar(ctx context.Context) ([]Usuario, error) {
    query := `SELECT id, nombre, email, creado_en FROM usuarios ORDER BY id`

    rows, err := r.db.QueryContext(ctx, query)
    if err != nil {
        return nil, fmt.Errorf("error al listar usuarios: %w", err)
    }
    defer rows.Close()

    var usuarios []Usuario
    for rows.Next() {
        var u Usuario
        if err := rows.Scan(&u.ID, &u.Nombre, &u.Email, &u.CreadoEn); err != nil {
            return nil, fmt.Errorf("error al escanear usuario: %w", err)
        }
        usuarios = append(usuarios, u)
    }

    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("error al iterar filas: %w", err)
    }

    return usuarios, nil
}

func (r *RepositorioUsuario) Actualizar(ctx context.Context, u *Usuario) error {
    query := `UPDATE usuarios SET nombre = $1, email = $2 WHERE id = $3`

    resultado, err := r.db.ExecContext(ctx, query, u.Nombre, u.Email, u.ID)
    if err != nil {
        return fmt.Errorf("error al actualizar usuario: %w", err)
    }

    filas, _ := resultado.RowsAffected()
    if filas == 0 {
        return fmt.Errorf("usuario %d no encontrado", u.ID)
    }
    return nil
}

func (r *RepositorioUsuario) Eliminar(ctx context.Context, id int) error {
    query := `DELETE FROM usuarios WHERE id = $1`

    resultado, err := r.db.ExecContext(ctx, query, id)
    if err != nil {
        return fmt.Errorf("error al eliminar usuario: %w", err)
    }

    filas, _ := resultado.RowsAffected()
    if filas == 0 {
        return fmt.Errorf("usuario %d no encontrado", id)
    }
    return nil
}
```

### Transacciones

```go
func (r *RepositorioUsuario) TransferirPuntos(ctx context.Context, deID, aID, puntos int) error {
    tx, err := r.db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("error al iniciar transaccion: %w", err)
    }
    defer tx.Rollback() // Rollback si no se hace commit

    // Descontar puntos
    resultado, err := tx.ExecContext(ctx,
        "UPDATE usuarios SET puntos = puntos - $1 WHERE id = $2 AND puntos >= $1",
        puntos, deID)
    if err != nil {
        return fmt.Errorf("error al descontar: %w", err)
    }
    filas, _ := resultado.RowsAffected()
    if filas == 0 {
        return fmt.Errorf("puntos insuficientes o usuario no encontrado")
    }

    // Acreditar puntos
    _, err = tx.ExecContext(ctx,
        "UPDATE usuarios SET puntos = puntos + $1 WHERE id = $2",
        puntos, aID)
    if err != nil {
        return fmt.Errorf("error al acreditar: %w", err)
    }

    return tx.Commit()
}
```

### Prepared statements

```go
func (r *RepositorioUsuario) CrearVarios(ctx context.Context, usuarios []Usuario) error {
    stmt, err := r.db.PrepareContext(ctx,
        "INSERT INTO usuarios (nombre, email) VALUES ($1, $2)")
    if err != nil {
        return fmt.Errorf("error al preparar statement: %w", err)
    }
    defer stmt.Close()

    for _, u := range usuarios {
        if _, err := stmt.ExecContext(ctx, u.Nombre, u.Email); err != nil {
            return fmt.Errorf("error al insertar usuario: %w", err)
        }
    }
    return nil
}
```

---

## 16.5 Migraciones

### Usando golang-migrate

```bash
# Instalar CLI
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Crear migracion
migrate create -ext sql -dir migrations -seq crear_tabla_usuarios

# Ejecutar migraciones
migrate -database "postgres://localhost/mi_db?sslmode=disable" -path migrations up

# Revertir
migrate -database "postgres://localhost/mi_db?sslmode=disable" -path migrations down

# Forzar version (si hay inconsistencia)
migrate -database "postgres://localhost/mi_db?sslmode=disable" -path migrations force 1
```

### Archivos de migracion

```sql
-- migrations/000001_crear_tabla_usuarios.up.sql
CREATE TABLE IF NOT EXISTS usuarios (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    puntos INTEGER DEFAULT 0,
    creado_en TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    actualizado_en TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_usuarios_email ON usuarios(email);
```

```sql
-- migrations/000001_crear_tabla_usuarios.down.sql
DROP TABLE IF EXISTS usuarios;
```

### Ejecutar migraciones desde codigo

```go
package main

import (
    "log"

    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func ejecutarMigraciones(databaseURL string) error {
    m, err := migrate.New(
        "file://migrations",
        databaseURL,
    )
    if err != nil {
        return err
    }
    defer m.Close()

    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return err
    }

    log.Println("Migraciones ejecutadas correctamente")
    return nil
}
```

### Embeber migraciones en el binario

```go
package main

import (
    "embed"

    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    "github.com/golang-migrate/migrate/v4/source/iofs"
)

//go:embed migrations/*.sql
var migracionesFS embed.FS

func ejecutarMigracionesEmbebidas(databaseURL string) error {
    source, err := iofs.New(migracionesFS, "migrations")
    if err != nil {
        return err
    }

    m, err := migrate.NewWithSourceInstance("iofs", source, databaseURL)
    if err != nil {
        return err
    }
    defer m.Close()

    return m.Up()
}
```

---

## Resumen del Capítulo

- `net/http` es el servidor HTTP de la stdlib, listo para produccion con timeouts adecuados.
- Los middlewares envuelven handlers para logging, CORS, auth, recovery y timeouts.
- Go 1.22+ permite definir metodos HTTP en las rutas (`GET /api/usuarios`).
- Las APIs REST se construyen con handlers por recurso y `json.NewEncoder/Decoder`.
- `database/sql` proporciona acceso a bases de datos con pool de conexiones y prepared statements.
- Las transacciones garantizan atomicidad con `BeginTx`, `Commit` y `Rollback`.
- Las migraciones (`golang-migrate`) gestionan el esquema de BD de forma controlada y versionada.
- Las migraciones pueden embeberse en el binario con `embed` y `iofs`.

En el siguiente y ultimo capitulo encontraras ejercicios practicos para consolidar todo lo aprendido.
