# Capítulo 11: Vistas Diferibles (@defer) en Profundidad

La optimización del tamaño del paquete inicial de descarga (initial bundle size) es uno de los mayores caballos de batalla en el desarrollo frontend moderno. Tradicionalmente, la única manera de aplicar la carga diferida (Lazy Loading) de componentes en Angular era asociándolos a configuraciones del enrutador. Si un componente pesado (como un editor de texto enriquecido, un mapa interactivo o un bloque de comentarios) vivía en la misma pantalla inicial, terminaba empaquetado en el bundle principal de descarga, ralentizando la velocidad de carga para el usuario.

Angular moderno (v17+) ha revolucionado esto incorporando las **Vistas Diferibles (Deferred Views - `@defer`)**.

`@defer` es un bloque sintáctico integrado en el compilador de Angular que permite aplicar la **carga diferida de componentes de manera puramente declarativa y directa en la plantilla**. Esto significa que puedes decidir exactamente *bajo qué condiciones* de interacción o visibilidad debe descargarse y renderizarse una porción del HTML, logrando una reducción masiva del bundle inicial sin configuraciones de rutas complejas.

En este capítulo, dominaremos el funcionamiento interno de `@defer`, exploraremos en detalle todos sus disparadores (triggers) integrados, aprenderemos a diseñar transiciones limpias utilizando bloques auxiliares de experiencia de usuario y analizaremos estrategias de precarga inteligente para aplicaciones de producción.

---

## 11.1 El Concepto y Funcionamiento Interno de `@defer`

Cuando envuelves un componente en un bloque `@defer`, el compilador de Angular (Ivy) analiza la plantilla y realiza una reestructuración profunda del código durante el proceso de compilación:

1. **División Física del Código (Code Splitting)**: Extrae de forma automática el componente y todas sus dependencias (estilos, subcomponentes, directivas) del bundle principal y los empaqueta en un **archivo JavaScript independiente (chunk perezoso)**.
2. **Generación de Importaciones Dinámicas**: Reemplaza el renderizado inmediato del componente por una promesa interna de importación dinámica (idéntica a un `import()`).
3. **Gestión de Estados**: Renderiza inicialmente un bloque marcador de posición de bajo peso (placeholder) y dispara la descarga física del archivo JavaScript únicamente cuando se cumple la condición o el disparador configurado.

```
          ┌────────────────────────────────────────────────────────┐
          │             Plantilla Principal (Bundle Inicial)       │
          │   [ Header Component ]                                 │
          │                                                        │
          │   @defer (on viewport) {                               │
          │       [ Componente Mapa Pesado (Carga Perezosa) ] ─────┼─────┐ (Chunk separado)
          │   } @placeholder {                                     │     │
          │       <div class="map-skeleton">Cargando...</div>      │     │
          │   }                                                    │     │
          └────────────────────────────────────────────────────────┘     │
                                                                         ▼
                                                     Descarga y compila en background
                                                     únicamente cuando entra en el viewport
```

### Requisito Fundamental: Componentes Standalone
Para que un componente pueda ser cargado de manera diferida mediante `@defer`, **debe ser obligatoriamente un Componente Standalone**. Si intentas usar `@defer` sobre un componente clásico declarado en un `NgModule`, el compilador no podrá realizar la división de paquetes y el componente se cargará de manera inmediata y tradicional.

---

## 11.2 Anatomía Completa y Bloques Auxiliares de `@defer`

Un bloque `@defer` completo se compone de cuatro secciones lógicas diseñadas para garantizar una experiencia de usuario (UX) impecable y libre de parpadeos visuales:

```html
@defer (on viewport) {
  <!-- 1. El Bloque Principal (Deferred Content) -->
  <!-- Aquí se ubican los componentes pesados que se descargarán de forma perezosa -->
  <app-grafico-analitico [datos]="reporte()"></app-grafico-analitico>
} @loading (after 100ms; minimum 500ms) {
  <!-- 2. El Bloque de Carga (Loading State) -->
  <!-- Se renderiza activamente MIENTRAS se está descargando el archivo JavaScript de red -->
  <div class="spinner">Descargando recursos analíticos...</div>
} @placeholder (minimum 300ms) {
  <!-- 3. El Bloque Marcador (Placeholder State) -->
  <!-- Se renderiza INICIALMENTE de forma instantánea. Debe ser ultra ligero -->
  <div class="esqueleto-tarjeta">Espacio reservado para gráfico corporativo</div>
} @error {
  <!-- 4. El Bloque de Error (Error State) -->
  <!-- Se renderiza únicamente si la descarga física de red falla (ej. pérdida de conexión) -->
  <div class="error-alerta">
    <p>No se pudo cargar el gráfico debido a un problema de conexión de red.</p>
  </div>
}
```

### Parámetros Avanzados de los Bloques Auxiliares:

* **`after` (en `@loading`)**: Evita parpadeos bruscos de spinners en conexiones de alta velocidad. Indica al bloque que espere un tiempo determinado (ej. `100ms`) antes de mostrar la animación de carga. Si el componente se descarga antes de ese tiempo, el spinner jamás se pinta en pantalla.
* **`minimum` (en `@loading` y `@placeholder`)**: Garantiza la estabilidad visual. Obliga al bloque visual a mostrarse durante al menos un periodo de tiempo (ej. `500ms`), evitando transiciones demasiado rápidas y parpadeos molestos que puedan frustrar al usuario.

---

## 11.3 Los Disparadores Incorporados (Triggers) de `@defer`

Angular proporciona una potente colección de disparadores predefinidos que cubren prácticamente cualquier escenario interactivo imaginable de diseño web premium:

### 1. `on idle` (Por Defecto)
El disparador estándar por defecto de Angular. Inicia la descarga del componente en segundo plano tan pronto como el hilo principal del navegador del usuario se encuentra libre (utilizando bajo el capó la API nativa `requestIdleCallback`).

```html
<!-- Se descarga en segundo plano cuando el navegador está ocioso -->
@defer (on idle) {
  <app-comentarios-foro></app-comentarios-foro>
}
```

---

### 2. `on immediate`
Dispara la descarga del paquete del componente de forma asíncrona e inmediata, tan pronto como la plantilla principal termina de procesar su renderizado inicial en pantalla.

```html
@defer (on immediate) {
  <app-modal-promocion></app-modal-promocion>
}
```

---

### 3. `on timer(duracion)`
Gatilla la descarga física del componente después de transcurrido un periodo de tiempo específico en milisegundos o segundos.

```html
<!-- Se descarga y renderiza exactamente 3 segundos después de cargar la web -->
@defer (on timer(3s)) {
  <app-newsletter-popup></app-newsletter-popup>
}
```

---

### 4. `on viewport`
Uno de los disparadores más espectaculares de rendimiento. Utiliza la API nativa `IntersectionObserver` para gatillar la descarga del componente **únicamente cuando el bloque marcador de posición (placeholder) entra físicamente en la pantalla visible del usuario** al hacer scroll.

Es mandatorio para pie de páginas pesados, mapas integrados en medio de la web o galerías de fotos extensas.

```html
<div class="espaciador-largo">Hacer scroll para visualizar el mapa</div>

<!-- El mapa se descargará únicamente cuando el usuario baje haciendo scroll y el placeholder sea visible -->
@defer (on viewport) {
  <app-mapa-google-premium></app-mapa-google-premium>
} @placeholder {
  <div class="esqueleto-mapa">Cargando mapa interactivo...</div>
}
```

---

### 5. `on interaction`
Dispara la descarga del componente perezoso cuando el usuario hace clic o interactúa físicamente con el marcador de posición (placeholder) o con un elemento de referencia externo.

```html
<!-- El editor se descarga únicamente cuando el usuario hace clic en el área reservada -->
@defer (on interaction) {
  <app-editor-texto-enriquecido></app-editor-texto-enriquecido>
} @placeholder {
  <div class="cursor-pointer p-4 border rounded">Haga clic aquí para empezar a escribir...</div>
}
```

---

### 6. `on hover`
Gatilla la descarga de los recursos cuando el puntero del ratón del usuario se sitúa por encima del marcador de posición o elemento seleccionado, anticipando con precisión que es muy probable que termine interactuando con él.

```html
<!-- Se descarga físicamente cuando pasas el ratón por encima del placeholder -->
@defer (on hover) {
  <app-menu-desplegable-detallado></app-menu-desplegable-detallado>
} @placeholder {
  <button class="btn-menu">Ver Opciones Expandidas</button>
}
```

---

## 11.4 Disparadores Lógicos Personalizados (`when`)

Si las interacciones nativas no son suficientes, `@defer` te permite crear disparadores condicionales lógicos avanzados utilizando la palabra clave **`when`** y pasándole cualquier expresión booleana o **Signal reactivo**.

A diferencia de los disparadores `on`, el bloque `@defer (when expresion)` cargará el componente perezoso cuando la condición se evalúe a `true`. Sin embargo, **nunca volverá a desmontar el componente si la condición regresa a `false`**. Es una transición permanente de renderizado.

```typescript
import { Component, signal } from "@angular/core";
import { FormularioEdicionComponent } from "./formulario-edicion.component";

@Component({
  selector: "app-perfil",
  standalone: true,
  imports: [FormularioEdicionComponent],
  template: `
    <button (click)="activarEdicion()" class="btn">Editar Perfil</button>

    <!-- Se carga y renderiza cuando el Signal 'habilitarEdicion' cambia a true -->
    @defer (when habilitarEdicion()) {
      <app-formulario-edicion></app-formulario-edicion>
    } @placeholder {
      <p>Modo de edición inactivo de momento.</p>
    }
  `
})
export class PerfilComponent {
  readonly habilitarEdicion = signal<boolean>(false);

  activarEdicion() {
    this.habilitarEdicion.set(true);
  }
}
```

---

## 11.5 Estrategia de Precarga Inteligente (`prefetch`)

¿Qué pasa si queremos que un componente analítico pesado se cargue de inmediato en segundo plano para que esté disponible instantáneamente cuando el usuario interactúe con él? `@defer` te otorga el control absoluto de esto permitiéndote **separar la condición de descarga (prefetch) de la condición de renderizado físico (render)**.

Para lograr esto, combinamos `@defer` con la instrucción **`prefetch`**:

```html
<!-- 
  PRECARGA (prefetch): Se descarga físicamente por red cuando el ratón del usuario 
                       pasa por encima (hover) del botón de opciones.
  RENDERIZADO: Se monta visualmente en el DOM únicamente cuando el usuario 
               hace clic físico (interaction) en dicho botón.
-->
@defer (on interaction; prefetch on hover) {
  <app-grafico-ventas></app-grafico-ventas>
} @placeholder {
  <button class="btn-analisis">Haga clic para expandir Estadísticas</button>
}
```

Esta técnica de precarga inteligente proporciona una experiencia visual instantánea de **cero latencia de renderizado (zero-latency feel)**, wow-eando al usuario al ofrecerle paneles dinámicos sin tiempos de espera.

---

## Resumen del Capítulo

* Las **Vistas Diferibles (`@defer`)** son una potente característica del compilador de plantillas de Angular moderno que permite realizar **Code-Splitting declarativo** directo en el HTML.
* El uso de `@defer` está restringido estrictamente a **Componentes Standalone** para permitir que las herramientas de empaquetado (build pipeline) aíslen correctamente los chunks dinámicos.
* Un bloque `@defer` robusto en producción debe ir acompañado de los bloques auxiliares **`@placeholder`**, **`@loading`** y **`@error`** para garantizar la consistencia visual y la estabilidad ante fallos de conexión.
* Parámetros de sincronización como **`after`** y **`minimum`** son vitales para evitar parpadeos molestos de spinners en conexiones de internet rápidas.
* Los disparadores integrados (`viewport`, `interaction`, `hover`, `timer`, `idle`) resuelven de forma estandarizada escenarios interactivos complejos sin necesidad de código imperativo.
* Combinar **`prefetch on`** con disparadores de renderizado permite anticipar y pre-descargar recursos pesados en red, ofreciendo interfaces dinámicas instantáneas de alto rendimiento empresarial.

En el próximo capítulo, aprenderemos cómo mejorar la indexación SEO de nuestros sitios de gran escala dominando el **Server-Side Rendering (SSR), SSG e Hidratación** en Angular moderno.
