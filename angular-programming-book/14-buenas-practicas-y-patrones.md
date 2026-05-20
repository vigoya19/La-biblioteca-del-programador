# Capítulo 14: Buenas Prácticas y Patrones de Diseño

El éxito a largo plazo de una aplicación web empresarial no depende de la rapidez con la que se escriba su primer prototipo, sino de la facilidad con la que un equipo de desarrolladores pueda seguir añadiendo funcionalidades, corrigiendo bugs y refactorizando el código sin introducir errores colaterales. A medida que las aplicaciones crecen, la consistencia, la claridad y el respeto a los principios de arquitectura de software se vuelven vitales.

En el ecosistema moderno de Angular, contar con un framework opinionado nos proporciona una excelente base de estandarización. Sin embargo, sigue siendo responsabilidad de los ingenieros aplicar patrones limpios para evitar que el código de negocio se mezcle de forma caótica con las plantillas visuales.

En este capítulo, aprenderemos a aplicar los **principios SOLID en el frontend**. Diseñaremos **estructuras de carpetas escalables y limpias** preparadas para equipos de cientos de desarrolladores, profundizaremos en el patrón de **Smart y Dumb Components**, exploraremos la integración de la **Arquitectura Hexagonal (DDD)** en Angular y definiremos una **lista de verificación (Code Review Checklist)** exhaustiva para tus revisiones de código de producción.

---

## 14.1 Principios SOLID Aplicados al Frontend

Los principios **SOLID** son cinco directrices fundamentales del diseño orientado a objetos que, adaptadas al desarrollo web con TypeScript y Angular, garantizan la máxima mantenibilidad del software:

### 1. Principio de Responsabilidad Única (Single Responsibility - S)
* *Concepto*: Un componente o servicio debe hacer exactamente una sola cosa.
* *En Angular*: Si un componente realiza llamadas directas a APIs HTTP, gestiona el estado de sesión, valida formularios complejos y además pinta la UI, está violando flagrantemente este principio. Delega las peticiones a **Servicios**, las validaciones a **Validadores puros** y la visualización visual pesada a **subcomponentes Dumb**.

### 2. Principio de Abierto/Cerrado (Open/Closed - O)
* *Concepto*: Las entidades de software deben estar abiertas a la extensión pero cerradas a la modificación.
* *En Angular*: En lugar de llenar un componente visual de múltiples condicionales `@if` internos para pintar variaciones (ej. botón rojo, botón azul, botón verde, botón con icono), extiende su comportamiento utilizando **Directivas de Atributo** o mapeando las propiedades mediante la API **`imports`** y el polimorfismo de componentes standalone.

### 3. Principio de Sustitución de Liskov (Liskov Substitution - L)
* *Concepto*: Las clases derivadas deben poder ser sustituidas por sus clases base sin alterar la corrección del programa.
* *En Angular*: Al diseñar implementaciones de inyección personalizadas (por ejemplo, si creas un `AuthMockService` para tests que hereda del `AuthService` real), asegúrate de respetar estrictamente las firmas de los métodos y los tipos de retorno de los observables y signals de la clase base.

### 4. Principio de Segregación de Interfaces (Interface Segregation - I)
* *Concepto*: Los clientes no deben estar obligados a depender de interfaces que no utilizan.
* *En Angular*: Al definir interfaces para tipar los inputs de tus componentes, no pases objetos de base de datos completos con 50 propiedades si el componente visual solo necesita pintar el `nombre` y la `foto`. Diseña interfaces ligeras y segregadas exclusivas para la vista.

```typescript
// EVITA ESTO (Acoplamiento excesivo):
export interface ComponenteInput {
  usuario: UsuarioCompletoBD; // Trae contraseñas, tokens, fechas de creación...
}

// MEJOR SOLUCIÓN (Interfaz segregada para presentación):
export interface TarjetaPerfilVM {
  nombre: string;
  avatarUrl: string;
}
```

### 5. Principio de Inversión de Dependencias (Dependency Inversion - D)
* *Concepto*: Depende de abstracciones, no de implementaciones concretas.
* *En Angular*: El sistema de Inyección de Dependencias nativo de Angular resuelve esto de forma brillante utilizando **`InjectionToken`** y clases abstractas. Permite que tus componentes dependan de un token abstracto de servicio, delegando al archivo `app.config.ts` la decisión de inyectar el servicio de producción real o un mock de testing.

---

## 14.2 Estructura de Carpetas Limpia y Mantenible (DDD y Feature-driven)

Para proyectos empresariales de gran escala, las estructuras de carpetas planas o desorganizadas provocan pérdidas de tiempo masivas buscando archivos. Se recomienda aplicar una **Arquitectura Guiada por Características (Feature-driven Layout)** combinada con **Domain-Driven Design (DDD)**.

```
src/
├── app/
│   ├── core/                        # Núcleo de la aplicación (Singleton global)
│   │   ├── guards/                  # Guards funcionales de seguridad
│   │   ├── interceptors/            # Interceptores HTTP de red
│   │   ├── services/                # Servicios singleton globales (Auth, API Client)
│   │   └── tokens/                  # Tokens de inyección de configuración
│   ├── shared/                      # Elementos compartidos reutilizables
│   │   ├── components/              # Dumb components comunes (botones, modales)
│   │   ├── directives/              # Directivas personalizadas de UI
│   │   └── pipes/                   # Pipes reutilizables de formateo de datos
│   ├── features/                    # Características de negocio (Feature-driven)
│   │   ├── auth/                    # Dominio de Autenticación
│   │   │   ├── components/          # Dumb components de login (formulario)
│   │   │   ├── services/            # Estado local del módulo
│   │   │   ├── login.component.ts   # Smart component (Página de Login)
│   │   │   └── auth.routes.ts       # Subrutas perezosas de autenticación
│   │   └── dashboard/               # Dominio de Panel de Control
│   ├── app.config.ts                # Configuración global standalone
│   ├── app.routes.ts                # Árbol de enrutamiento principal
│   └── app.component.ts             # Componente raíz
```

---

## 14.3 El Patrón Smart Components vs. Dumb Components

La separación avanzada de responsabilidades visuales y lógicas es la mejor herramienta para mantener el código testeable y reutilizable. Dividimos nuestros componentes en dos roles claramente definidos:

### 1. Smart Components (Componentes Inteligentes)
* **Responsabilidad**: Orquestar la lógica de negocio y controlar el flujo de datos.
* **Características**:
  * Inyectan servicios de negocio y realizan peticiones HTTP remotos.
  * Se conectan a Stores globales de estado (ej. NgRx Signals Store).
  * Escuchan los eventos emitidos por los componentes Dumb en su plantilla y reaccionan.
  * Suelen ser las páginas finales enlazadas al enrutador (ej. `login.component.ts`, `catalogo.component.ts`).

### 2. Dumb Components (Componentes Presentacionales / Tontos)
* **Responsabilidad**: Renderizar la interfaz visual y gestionar la interacción directa del usuario en pantalla.
* **Características**:
  * **Cero inyección de servicios de negocio**. No saben de dónde vienen los datos.
  * Reciben datos estrictamente a través de Inputs basados en Signals (`input()`).
  * Notifican interacciones exclusivamente a través de Outputs (`output()`).
  * Son altamente reutilizables en múltiples pantallas.
  * Suelen ser bloques visuales y layouts comunes (ej. `tarjeta-producto.component.ts`, `tabla-datos.component.ts`).

```typescript
// EJEMPLO DUMB COMPONENT (Puro, reutilizable y agnóstico a servicios):
@Component({
  selector: "ui-tarjeta-precio",
  standalone: true,
  template: `
    <div class="border p-4 rounded shadow">
      <h3>{{ titulo() }}</h3>
      <p class="text-green-600 font-bold">{{ valor() }} €</p>
      <button (click)="emitirClick()" class="btn">Comprar ahora</button>
    </div>
  `
})
export class TarjetaPrecioComponent {
  readonly titulo = input.required<string>();
  readonly valor = input.required<number>();
  readonly comprar = output<void>(); // Emite el evento hacia el padre

  emitirClick(): void {
    this.comprar.emit();
  }
}
```

---

## 14.4 Integración de Arquitectura Hexagonal y DDD en Angular

Para aplicaciones de gran escala que forman parte de sistemas core de la empresa, es sumamente recomendable aislar por completo la lógica de negocio pura del framework de Angular. Esto se logra implementando **Arquitectura Hexagonal**.

En este enfoque, Angular se reduce a un simple "detalle de infraestructura" (el adaptador primario de presentación). La lógica pura, las reglas de negocio y los modelos de dominio viven en una capa interna agnóstica que utiliza TypeScript puro, facilitando portar el código a otros frameworks o entornos Node.js si fuera necesario en el futuro.

Puedes consultar una explicación teórica sumamente detallada de esta arquitectura en nuestro libro de [Arquitectura de Software: Capas de Hexagonal y Clean](../libro-arquitectura-software/07-hexagonal-clean.md).

```
┌───────────────────────────────────────────────────────────────┐
│              Infraestructura (Angular Adapter)                │
│    - ListaProductosComponent                                  │
│    - AngularHttpClient                                        │
└──────────────────────────────┬────────────────────────────────┘
                               ▼ (Inyección de Interfaces)
┌───────────────────────────────────────────────────────────────┐
│               Dominio y Casos de Uso (Hexágono Puro)          │
│    - Caso de Uso: ObtenerCatalogo                             │
│    - Puerto: RepositorioProductos (Interface)                 │
│    - Entidad de Dominio: Producto (Modelo Puro)               │
└───────────────────────────────────────────────────────────────┘
```

---

## 14.5 Code Review Checklist para Angular Moderno

Para tus revisiones de código de producción, utiliza esta lista de verificación exhaustiva para garantizar la máxima calidad técnica:

### 1. Arquitectura y Estructura
* `[ ]` ¿El componente es Standalone? (Mandatorio en Angular moderno).
* `[ ]` ¿Se ha configurado la estrategia de detección de cambios `ChangeDetectionStrategy.OnPush`?
* `[ ]` ¿Las carpetas respetan la división Feature-driven?

### 2. Reactividad y Signals
* `[ ]` ¿Se utilizan Signal-based Inputs (`input()`) en lugar del clásico decorador `@Input()`?
* `[ ]` ¿Los estados derivados dinámicos utilizan `computed()` en lugar de getters costosos en la plantilla HTML?
* `[ ]` ¿Se ha evitado escribir en otros Signals dentro de bloques de efectos (`effect()`)?

### 3. Asincronismo y Conectividad
* `[ ]` ¿Los Observables HTTP de componentes se cancelan automáticamente en la destrucción utilizando `takeUntilDestroyed`?
* `[ ]` ¿Las interceptaciones de red o inyección de tokens JWT se realizan de forma invisible mediante **Interceptores HTTP Funcionales**?
* `[ ]` ¿Se han evitado suscripciones anidadas en red (callback hell) utilizando operadores de orquestación como `switchMap`?

### 4. Plantillas y Control Flow
* `[ ]` ¿Se utiliza el nuevo **Control Flow sintáctico** (`@if`, `@for`, `@switch`) integrado en el compilador?
* `[ ]` ¿El bloque `@for` cuenta de forma obligatoria con la instrucción **`track`** configurada con un identificador único?
* `[ ]` ¿Las imágenes críticas de cabecera utilizan la directiva **`NgOptimizedImage`** configurada con la bandera `priority`?

---

## Resumen del Capítulo

* La aplicación de los **principios SOLID** en frontend reduce drásticamente la deuda técnica al delimitar las responsabilidades de clases y componentes.
* La organización de archivos **Feature-driven** estructurada en dominios de negocio asegura la escalabilidad y legibilidad de proyectos corporativos extensos.
* El patrón **Smart y Dumb Components** separa con éxito la orquestación de negocio (Smart) del renderizado agnóstico e interactivo visual (Dumb).
* La **Arquitectura Hexagonal** blinda las reglas de negocio al aislarlas de Angular, tratándolo como un adaptador de interfaz reemplazable de infraestructura.
* Una **lista de verificación estricta de Code Review** unifica los estándares de calidad técnica del equipo, acelerando los pipelines de CI/CD para producción.

En el próximo capítulo, consolidaremos de forma práctica todo el conocimiento adquirido en este libro construyendo paso a paso un **Proyecto Integrador de Alto Rendimiento**.

---

← [Capítulo anterior](13-testing-en-angular.md) | [Inicio](README.md) | [Capítulo siguiente →](15-ejercicios-practicos.md)
