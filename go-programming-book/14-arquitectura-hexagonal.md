# Capítulo 14: Arquitectura Hexagonal en Go

La arquitectura hexagonal (o de Puertos y Adaptadores) propuesta por Alistair Cockburn, es una forma de estructurar aplicaciones para que el dominio sea independiente de la infraestructura. En Go, gracias a las interfaces implicitamente implementadas, esta arquitectura encaja de forma natural.

> [!NOTE]
> Para comprender en profundidad los fundamentos teóricos detrás de estos patrones, te recomendamos leer el [Capítulo 6: Domain-Driven Design (DDD)](../libro-arquitectura-software/06-ddd.md) y el [Capítulo 7: Arquitectura Hexagonal y Clean Architecture](../libro-arquitectura-software/07-hexagonal-clean.md) del libro de **Arquitectura de Software** en tu workspace.

---

## 14.1 Principios de la Arquitectura Hexagonal

### Concepto central

La idea es simple: **el negocio en el centro, la infraestructura afuera**.

```
         ┌──────────────────────┐
         │    ADAPTADORES       │
         │  ┌──────────────┐    │
         │  │  APLICACION  │    │
         │  │ ┌──────────┐ │    │
   HTTP ─┼──┼─┤ DOMINIO  ├─┼────┼── PostgreSQL
         │  │ └──────────┘ │    │
         │  └──────────────┘    │
         │    ADAPTADORES       │
         └──────────────────────┘
```

**Regla fundamental**: el dominio no conoce nada del exterior. La infraestructura conoce al dominio, nunca al reves.

### Beneficios

- **Testeable**: el dominio se prueba sin BD, HTTP, ni dependencias externas.
- **Reemplazable**: cambiar de PostgreSQL a MongoDB solo requiere un nuevo adaptador.
- **Mantenible**: cada capa tiene una responsabilidad clara.
- **Flexible**: agregar nuevos adaptadores (gRPC, CLI, RabbitMQ) sin tocar el dominio.

---

## 14.2 Puertos y Adaptadores

### Puertos (Interfaces)

Los puertos son interfaces que definen como se comunica el dominio con el exterior:

```go
// Puerto de entrada (driven port): define lo que ofrece el dominio
type ServicioUsuario interface {
    Registrar(email, contrasena string) (*Usuario, error)
    Autenticar(email, contrasena string) (string, error) // Retorna token
    ObtenerPerfil(id string) (*Usuario, error)
}

// Puerto de salida (driving port): define lo que el dominio necesita del exterior
type RepositorioUsuario interface {
    Guardar(ctx context.Context, u *Usuario) error
    BuscarPorID(ctx context.Context, id string) (*Usuario, error)
    BuscarPorEmail(ctx context.Context, email string) (*Usuario, error)
    Actualizar(ctx context.Context, u *Usuario) error
    Eliminar(ctx context.Context, id string) error
}

type ServicioEmail interface {
    EnviarBienvenida(ctx context.Context, email, nombre string) error
    EnviarRecuperacion(ctx context.Context, email, token string) error
}
```

### Adaptadores (Implementaciones)

Los adaptadores implementan los puertos:

```go
// Adaptador de entrada: HTTP Handler
type UsuarioHandler struct {
    servicio ServicioUsuario
}

// Adaptador de salida: PostgreSQL
type RepositorioPostgres struct {
    db *sql.DB
}

// Adaptador de salida: SMTP
type ServicioEmailSMTP struct {
    host string
    port int
}
```

---

## 14.3 Dominio, Aplicacion e Infraestructura

### Estructura del proyecto

```
mi-proyecto/
├── cmd/
│   └── servidor/
│       └── main.go              # Ensambla todo
├── internal/
│   ├── dominio/                  # Reglas de negocio puras
│   │   ├── usuario.go           # Entidad Usuario
│   │   ├── errores.go           # Errores de dominio
│   │   └── usuario_test.go
│   ├── aplicacion/               # Casos de uso
│   │   ├── servicio_usuario.go  # Logica de aplicacion
│   │   ├── puertos.go           # Interfaces (puertos)
│   │   └── servicio_usuario_test.go
│   └── infraestructura/          # Implementaciones concretas
│       ├── postgres/
│       │   └── repositorio_usuario.go
│       ├── http/
│       │   ├── handler_usuario.go
│       │   └── middleware.go
│       └── email/
│           └── servicio_email.go
├── go.mod
└── go.sum
```

---

## 14.4 Implementacion Paso a Paso

### Paso 1: Definir el dominio

```go
// internal/dominio/usuario.go
package dominio

import (
    "errors"
    "regexp"
    "strings"
)

type Usuario struct {
    ID    string
    Email string
    Nombre string
}

var (
    ErrEmailInvalido   = errors.New("el email no tiene un formato valido")
    ErrNombreVacio     = errors.New("el nombre no puede estar vacio")
    ErrEmailDuplicado  = errors.New("el email ya esta registrado")
    ErrUsuarioNoExiste = errors.New("el usuario no existe")
)

func NuevoUsuario(email, nombre string) (*Usuario, error) {
    email = strings.TrimSpace(strings.ToLower(email))
    nombre = strings.TrimSpace(nombre)

    if !validarEmail(email) {
        return nil, ErrEmailInvalido
    }
    if nombre == "" {
        return nil, ErrNombreVacio
    }

    return &Usuario{
        ID:     generarID(),
        Email:  email,
        Nombre: nombre,
    }, nil
}

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

func validarEmail(email string) bool {
    return emailRegex.MatchString(email)
}

func generarID() string {
    return "id-" + strings.ToLower(email[:strings.Index(email, "@")])
}
```

### Paso 2: Definir puertos (interfaces)

```go
// internal/aplicacion/puertos.go
package aplicacion

import (
    "context"
    "mi-proyecto/internal/dominio"
)

// Puertos de entrada (lo que ofrece la aplicacion)
type ServicioUsuario interface {
    Registrar(ctx context.Context, email, nombre string) (*dominio.Usuario, error)
    ObtenerPerfil(ctx context.Context, id string) (*dominio.Usuario, error)
}

// Puertos de salida (lo que necesita la aplicacion del exterior)
type RepositorioUsuario interface {
    Guardar(ctx context.Context, u *dominio.Usuario) error
    BuscarPorID(ctx context.Context, id string) (*dominio.Usuario, error)
    BuscarPorEmail(ctx context.Context, email string) (*dominio.Usuario, error)
}

type ServicioNotificacion interface {
    EnviarBienvenida(ctx context.Context, email, nombre string) error
}
```

### Paso 3: Implementar la capa de aplicacion

```go
// internal/aplicacion/servicio_usuario.go
package aplicacion

import (
    "context"
    "fmt"

    "mi-proyecto/internal/dominio"
)

type servicioUsuarioImpl struct {
    repo    RepositorioUsuario
    notif   ServicioNotificacion
}

func NuevoServicioUsuario(repo RepositorioUsuario, notif ServicioNotificacion) ServicioUsuario {
    return &servicioUsuarioImpl{
        repo:  repo,
        notif: notif,
    }
}

func (s *servicioUsuarioImpl) Registrar(ctx context.Context, email, nombre string) (*dominio.Usuario, error) {
    // 1. Crear entidad de dominio (validaciones en dominio)
    usuario, err := dominio.NuevoUsuario(email, nombre)
    if err != nil {
        return nil, fmt.Errorf("datos invalidos: %w", err)
    }

    // 2. Verificar unicidad (regla de aplicacion)
    existente, err := s.repo.BuscarPorEmail(ctx, email)
    if err != nil && err != dominio.ErrUsuarioNoExiste {
        return nil, fmt.Errorf("error al verificar email: %w", err)
    }
    if existente != nil {
        return nil, dominio.ErrEmailDuplicado
    }

    // 3. Persistir
    if err := s.repo.Guardar(ctx, usuario); err != nil {
        return nil, fmt.Errorf("error al guardar usuario: %w", err)
    }

    // 4. Notificar (fire and forget)
    go func() {
        if err := s.notif.EnviarBienvenida(context.Background(), usuario.Email, usuario.Nombre); err != nil {
            // Loggear, no fallar el registro
        }
    }()

    return usuario, nil
}

func (s *servicioUsuarioImpl) ObtenerPerfil(ctx context.Context, id string) (*dominio.Usuario, error) {
    usuario, err := s.repo.BuscarPorID(ctx, id)
    if err != nil {
        return nil, err
    }
    return usuario, nil
}
```

### Paso 4: Implementar adaptadores de infraestructura

```go
// internal/infraestructura/postgres/repositorio_usuario.go
package postgres

import (
    "context"
    "database/sql"
    "errors"
    "fmt"

    "mi-proyecto/internal/dominio"
)

type RepositorioUsuario struct {
    db *sql.DB
}

func NuevoRepositorioUsuario(db *sql.DB) *RepositorioUsuario {
    return &RepositorioUsuario{db: db}
}

func (r *RepositorioUsuario) Guardar(ctx context.Context, u *dominio.Usuario) error {
    query := `INSERT INTO usuarios (id, email, nombre) VALUES ($1, $2, $3)`
    _, err := r.db.ExecContext(ctx, query, u.ID, u.Email, u.Nombre)
    if err != nil {
        return fmt.Errorf("error al insertar usuario: %w", err)
    }
    return nil
}

func (r *RepositorioUsuario) BuscarPorID(ctx context.Context, id string) (*dominio.Usuario, error) {
    query := `SELECT id, email, nombre FROM usuarios WHERE id = $1`
    u := &dominio.Usuario{}
    err := r.db.QueryRowContext(ctx, query, id).Scan(&u.ID, &u.Email, &u.Nombre)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, dominio.ErrUsuarioNoExiste
    }
    if err != nil {
        return nil, fmt.Errorf("error al buscar usuario: %w", err)
    }
    return u, nil
}

func (r *RepositorioUsuario) BuscarPorEmail(ctx context.Context, email string) (*dominio.Usuario, error) {
    query := `SELECT id, email, nombre FROM usuarios WHERE email = $1`
    u := &dominio.Usuario{}
    err := r.db.QueryRowContext(ctx, query, email).Scan(&u.ID, &u.Email, &u.Nombre)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, dominio.ErrUsuarioNoExiste
    }
    if err != nil {
        return nil, fmt.Errorf("error al buscar usuario por email: %w", err)
    }
    return u, nil
}
```

```go
// internal/infraestructura/http/handler_usuario.go
package http

import (
    "encoding/json"
    "errors"
    "net/http"

    "mi-proyecto/internal/aplicacion"
    "mi-proyecto/internal/dominio"
)

type UsuarioHandler struct {
    servicio aplicacion.ServicioUsuario
}

func NuevoUsuarioHandler(s aplicacion.ServicioUsuario) *UsuarioHandler {
    return &UsuarioHandler{servicio: s}
}

type RegistrarRequest struct {
    Email  string `json:"email"`
    Nombre string `json:"nombre"`
}

func (h *UsuarioHandler) Registrar(w http.ResponseWriter, r *http.Request) {
    var req RegistrarRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "cuerpo invalido", http.StatusBadRequest)
        return
    }

    usuario, err := h.servicio.Registrar(r.Context(), req.Email, req.Nombre)
    if err != nil {
        switch {
        case errors.Is(err, dominio.ErrEmailInvalido),
             errors.Is(err, dominio.ErrNombreVacio):
            http.Error(w, err.Error(), http.StatusBadRequest)
        case errors.Is(err, dominio.ErrEmailDuplicado):
            http.Error(w, err.Error(), http.StatusConflict)
        default:
            http.Error(w, "error interno", http.StatusInternalServerError)
        }
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(usuario)
}

func (h *UsuarioHandler) ObtenerPerfil(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")

    usuario, err := h.servicio.ObtenerPerfil(r.Context(), id)
    if err != nil {
        if errors.Is(err, dominio.ErrUsuarioNoExiste) {
            http.Error(w, "usuario no encontrado", http.StatusNotFound)
            return
        }
        http.Error(w, "error interno", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(usuario)
}
```

### Paso 5: Ensamblar en main

```go
// cmd/servidor/main.go
package main

import (
    "database/sql"
    "log"
    "net/http"
    "os"

    _ "github.com/lib/pq"

    "mi-proyecto/internal/aplicacion"
    httpAdapter "mi-proyecto/internal/infraestructura/http"
    emailAdapter "mi-proyecto/internal/infraestructura/email"
    "mi-proyecto/internal/infraestructura/postgres"
)

func main() {
    // Infraestructura
    db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Adaptadores de salida
    repoUsuario := postgres.NuevoRepositorioUsuario(db)
    servicioEmail := emailAdapter.NuevoServicioEmail("smtp.ejemplo.com", 587)

    // Capa de aplicacion
    servicioUsuario := aplicacion.NuevoServicioUsuario(repoUsuario, servicioEmail)

    // Adaptadores de entrada
    usuarioHandler := httpAdapter.NuevoUsuarioHandler(servicioUsuario)

    // Configurar servidor HTTP
    mux := http.NewServeMux()
    mux.HandleFunc("POST /usuarios", usuarioHandler.Registrar)
    mux.HandleFunc("GET /usuarios/{id}", usuarioHandler.ObtenerPerfil)

    log.Println("Servidor iniciado en :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

---

## 14.5 Testing en Arquitectura Hexagonal

### Test del dominio (sin dependencias)

```go
// internal/dominio/usuario_test.go
package dominio

import "testing"

func TestNuevoUsuario_EmailValido(t *testing.T) {
    u, err := NuevoUsuario("andres@ejemplo.com", "Andres")
    if err != nil {
        t.Fatalf("error inesperado: %v", err)
    }
    if u.Email != "andres@ejemplo.com" {
        t.Errorf("email = %q, esperado %q", u.Email, "andres@ejemplo.com")
    }
}

func TestNuevoUsuario_EmailInvalido(t *testing.T) {
    casos := []string{"", "sinarroba", "@sinusuario", "sin@dominio"}
    for _, email := range casos {
        _, err := NuevoUsuario(email, "Nombre")
        if err != ErrEmailInvalido {
            t.Errorf("email %q: esperado ErrEmailInvalido, obtenido %v", email, err)
        }
    }
}
```

### Test de aplicacion (con mocks)

```go
// internal/aplicacion/servicio_usuario_test.go
package aplicacion

import (
    "context"
    "testing"

    "mi-proyecto/internal/dominio"
)

// Mock del repositorio
type repoMock struct {
    usuarios map[string]*dominio.Usuario
}

func (m *repoMock) Guardar(ctx context.Context, u *dominio.Usuario) error {
    m.usuarios[u.Email] = u
    return nil
}

func (m *repoMock) BuscarPorID(ctx context.Context, id string) (*dominio.Usuario, error) {
    for _, u := range m.usuarios {
        if u.ID == id {
            return u, nil
        }
    }
    return nil, dominio.ErrUsuarioNoExiste
}

func (m *repoMock) BuscarPorEmail(ctx context.Context, email string) (*dominio.Usuario, error) {
    u, ok := m.usuarios[email]
    if !ok {
        return nil, dominio.ErrUsuarioNoExiste
    }
    return u, nil
}

// Mock de notificaciones
type notifMock struct {
    enviados []string
}

func (m *notifMock) EnviarBienvenida(ctx context.Context, email, nombre string) error {
    m.enviados = append(m.enviados, email)
    return nil
}

func TestRegistrar_Exitoso(t *testing.T) {
    repo := &repoMock{usuarios: make(map[string]*dominio.Usuario)}
    notif := &notifMock{}
    servicio := NuevoServicioUsuario(repo, notif)

    u, err := servicio.Registrar(context.Background(), "andres@ejemplo.com", "Andres")
    if err != nil {
        t.Fatalf("error inesperado: %v", err)
    }
    if u.Email != "andres@ejemplo.com" {
        t.Errorf("email = %q", u.Email)
    }
}

func TestRegistrar_EmailDuplicado(t *testing.T) {
    repo := &repoMock{usuarios: make(map[string]*dominio.Usuario)}
    notif := &notifMock{}
    servicio := NuevoServicioUsuario(repo, notif)

    // Primer registro
    servicio.Registrar(context.Background(), "andres@ejemplo.com", "Andres")

    // Segundo registro con el mismo email
    _, err := servicio.Registrar(context.Background(), "andres@ejemplo.com", "Otro")
    if err != dominio.ErrEmailDuplicado {
        t.Errorf("esperado ErrEmailDuplicado, obtenido %v", err)
    }
}
```

---

## 14.6 Proyecto Completo de Ejemplo

La estructura final del proyecto:

```
proyecto-hexagonal/
├── cmd/
│   └── servidor/
│       └── main.go
├── internal/
│   ├── dominio/
│   │   ├── usuario.go
│   │   ├── errores.go
│   │   └── usuario_test.go
│   ├── aplicacion/
│   │   ├── puertos.go
│   │   ├── servicio_usuario.go
│   │   └── servicio_usuario_test.go
│   └── infraestructura/
│       ├── postgres/
│       │   ├── repositorio_usuario.go
│       │   └── repositorio_usuario_integration_test.go
│       ├── http/
│       │   ├── handler_usuario.go
│       │   └── handler_usuario_test.go
│       └── email/
│           └── servicio_email.go
├── go.mod
├── go.sum
└── Makefile
```

---

## Resumen del Capítulo

- La arquitectura hexagonal aísla el dominio de la infraestructura mediante puertos (interfaces) y adaptadores (implementaciones).
- El dominio contiene las reglas de negocio puras, sin dependencias externas.
- La capa de aplicacion orquesta los casos de uso usando los puertos definidos.
- La infraestructura implementa los adaptadores concretos (HTTP, PostgreSQL, SMTP).
- Las dependencias apuntan hacia adentro: infraestructura -> aplicacion -> dominio.
- El testing es trivial: el dominio se prueba sin mocks, la aplicacion con mocks de puertos.
- Los adaptadores de infraestructura se prueban con integracion real (BD de prueba, etc.).
- `main.go` ensambla todas las piezas (composicion raiz).

En el siguiente capítulo exploraremos temas avanzados de Go.
