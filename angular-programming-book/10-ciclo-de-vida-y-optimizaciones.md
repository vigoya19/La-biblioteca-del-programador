# Capítulo 10: Ciclo de Vida y Optimización de Rendimiento

El verdadero valor de un ingeniero de software no se mide únicamente por su capacidad para implementar funcionalidades visuales que funcionen en su máquina de desarrollo local. En el entorno profesional, las aplicaciones deben ser rápidas, fluidas, consumir el mínimo de memoria y ofrecer una experiencia de usuario sobresaliente en todo tipo de dispositivos móviles de gama baja y conexiones con limitaciones de red.

Para lograr un rendimiento de nivel empresarial, es imprescindible dominar el comportamiento interno del framework. Esto implica comprender cómo y cuándo se renderizan nuestros componentes y cómo responde Angular ante las interacciones de los usuarios.

En este capítulo, profundizaremos en la mecánica del **motor de detección de cambios de Angular**. Analizaremos en detalle los **hooks del ciclo de vida**, dominaremos la estrategia de optimización de renderizado **`OnPush`** y nos adentraremos en el mayor hito de rendimiento de la era moderna: **Zoneless Angular (desarrollo libre de Zone.js)**. Finalmente, aprenderemos a perfilar aplicaciones mediante **Angular DevTools** y optimizaremos recursos críticos con **`NgOptimizedImage`**.

---

## 10.1 Los Hooks del Ciclo de Vida en Detalle

Como vimos conceptualmente en el Capítulo 2, Angular administra de forma automática la creación, actualización e invalidez de los componentes. A continuación, analizaremos técnicamente el comportamiento y el propósito exacto de cada uno de los ganchos del ciclo de vida (**Lifecycle Hooks**):

| Gancho | Tipo de Operación | Frecuencia de Ejecución | Propósito Principal en Producción |
|---|---|---|---|
| **`ngOnChanges`** | Reactivo | Ante cambios en `@Input()` | Capturar y responder a modificaciones en parámetros de entrada tradicionales. |
| **`ngOnInit`** | Inicialización | Una única vez | Realizar peticiones HTTP, inicializar el estado del componente e inyecciones iniciales de servicios. |
| **`ngDoCheck`** | Detección personalizada | Frecuente (ante cada ciclo) | Monitorear e interceptar cambios complejos que escapan de las comprobaciones automáticas de Angular. |
| **`ngAfterContentInit`** | Renderizado de Proyección | Una única vez | Se ejecuta después de proyectar contenido externo (`<ng-content>`) dentro de las plantillas del componente. |
| **`ngAfterContentChecked`**| Renderizado de Proyección | Frecuente (ante cada ciclo) | Verificar la validez del contenido externo proyectado. |
| **`ngAfterViewInit`** | Renderizado del DOM | Una única vez | Interactuar de forma segura con el DOM real del componente o inicializar librerías visuales de terceros (ej. Chart.js). |
| **`ngAfterViewChecked`** | Renderizado del DOM | Frecuente (ante cada ciclo) | Verificar la validez de la vista ya pintada en el navegador. |
| **`ngOnDestroy`** | Limpieza | Una única vez | **Prevenir fugas de memoria (memory leaks)** cancelando suscripciones de RxJS, eventos del DOM y temporizadores. |

---

## 10.2 Estrategias de Detección de Cambios: `Default` frente a `OnPush`

Por defecto, Angular utiliza el algoritmo de detección de cambios clásico (**`ChangeDetectionStrategy.Default`**). Bajo esta estrategia, cuando sucede cualquier evento asíncrono en el navegador (como una petición HTTP que responde, un click de botón o un simple `setTimeout`), el framework recorre ciegamente **todo el árbol de componentes de arriba a abajo**, comprobando cada expresión para verificar si el DOM requiere actualización.

En aplicaciones corporativas masivas, esto introduce un overhead inaceptable. Para optimizar esto, los ingenieros profesionales configuran la estrategia **`ChangeDetectionStrategy.OnPush`**.

```
Estrategia Default (Barrido completo ciego):
[Componente Raíz] ──► [Componente Hijo] ──► [Componente Nieto (Verifica todo ante cualquier click)]

Estrategia OnPush (Verificación inteligente y saltos de rama):
[Componente Raíz] ──► [Hijo OnPush (Sin cambios de Input)] ──► [Nieto OnPush (Saltado/No verificado)]
      │
      ▼
(Optimiza el procesador al evitar evaluar ramas estáticas enteras de componentes)
```

### ¿Cómo funciona `OnPush`?

Cuando configuras un componente con `changeDetection: ChangeDetectionStrategy.OnPush`, le indicas a Angular que **debe ignorar y saltarse por completo el ciclo de detección de cambios de este componente y de todos sus hijos**, a menos que suceda uno de los siguientes eventos específicos:

1. **Nueva Referencia en un Input**: Uno de los parámetros de entrada del componente recibe un nuevo valor que cambia de referencia física en memoria (`===`). *Por ello es crucial utilizar inmutabilidad*.
2. **Emisión de Eventos en la Plantilla**: Se dispara un evento directo desde el HTML del propio componente (ej. `(click)="hacerAlgo()"`).
3. **Señales Reactivas Modificadas**: Un Signal leído directamente en la plantilla del componente sufre modificaciones (`set()` o `update()`).
4. **Marcado Manual Explicito**: El componente solicita de forma imperativa su verificación llamando al método `markForCheck()` del servicio inyectado `ChangeDetectorRef`.

### Configuración Práctica de `OnPush`
```typescript
import { Component, ChangeDetectionStrategy, input } from "@angular/core";

@Component({
  selector: "ui-tarjeta-premium",
  standalone: true,
  template: `<div class="card"><h3>{{ titulo() }}</h3></div>`,
  // Activación mandatoria de OnPush en componentes de producción
  changeDetection: ChangeDetectionStrategy.OnPush 
})
export class TarjetaPremiumComponent {
  readonly titulo = input.required<string>();
}
```

---

## 10.3 Zoneless Angular: El Mayor Hito de Rendimiento (v18/19+)

Históricamente, la detección de cambios automática de Angular ha estado impulsada por **`zone.js`**. Esta librería modifica las APIs nativas del navegador (como `addEventListener`, `fetch`, `setTimeout`) para enterarse de cuándo sucede cualquier tarea asíncrona y forzar el barrido global.

A partir de Angular v18/19, el framework ha introducido el soporte oficial y experimental para **Zoneless Angular (Detección de Cambios Libre de Zonas)**.

### ¿Por qué Zoneless es el futuro?
* **Reducción del Tamaño del Bundle**: Al remover `zone.js`, tu aplicación reduce instantáneamente cerca de **15kB a 30kB gzip** del bundle inicial descargado por los usuarios.
* **Rendimiento Excepcional en Rendering**: Angular ya no realiza barridos globales automáticos ciegos. La detección de cambios es gatillada granularmente por Signals y marcados finos de eventos, eliminando repintadas fantasmas.
* **Depuración de Errores Simplificada**: Los rastros de pila de excepciones (stack traces) en la consola del navegador son limpios y legibles, sin las cientos de líneas internas incomprensibles generadas por Zone.js.

### Cómo Configurar una Aplicación 100% Zoneless
Para activar Zoneless en tu aplicación standalone, debemos remover la detección de cambios por zonas y proveer el detector experimental en tu archivo de configuración global `app.config.ts`:

```typescript
import { ApplicationConfig, provideExperimentalZonelessChangeDetection } from "@angular/core";
import { provideRouter } from "@angular/router";
import { routes } from "./app.routes";

export const appConfig: ApplicationConfig = {
  providers: [
    // 1. Activamos la detección de cambios de alto rendimiento Zoneless nativa
    provideExperimentalZonelessChangeDetection(),
    
    provideRouter(routes)
  ]
};
```

Adicionalmente, debes retirar la importación de `zone.js` en tu archivo `angular.json` o en tu script de inicio `polyfills` para que el navegador no lo descargue en memoria. ¡Tu aplicación ahora vuela libre!

---

## 10.4 Diagnóstico y Perfilado con Angular DevTools

Para identificar qué componentes están sufriendo renderizados lentos o bucles innecesarios de actualización, Google provee la extensión oficial para navegadores **Angular DevTools**.

```
┌─────────────────────────────────────────────────────────────────┐
│                       Angular DevTools Profiler                 │
│                                                                 │
│  [■ Record]  [|| Pause]  [↻ Clear]                             │
│                                                                 │
│  Componentes Evaluados en Tiempo Real:                          │
│  ███████████████████████ 14ms  -> app-dashboard (Lento / Default)│
│  ░░░ 1.2ms                 -> ui-tarjeta-premium (Rápido / OnPush)│
└─────────────────────────────────────────────────────────────────┘
```

### Cómo Utilizar el Profiler para Encontrar Cuellos de Botella:
1. Instala la extensión **Angular DevTools** desde la Chrome Web Store.
2. Abre tu aplicación en el servidor local de desarrollo (`ng serve`) y abre las Chrome DevTools (`F12`).
3. Ve a la pestaña **Angular** y selecciona la sub-pestaña **Profiler**.
4. Haz click en el botón **Record (Grabar)** e interactúa con tu aplicación (haz scroll, clics en filtros, etc.).
5. Detén la grabación. La herramienta te presentará un diagrama de barras de colores indicando el tiempo exacto que le tomó a cada componente procesar la detección de cambios.
   * **Barras Rojas/Naranjas (> 10ms)**: Componentes lentos que requieren refactorización inmediata (casi siempre solucionable aplicando `OnPush` o reduciendo llamadas de métodos costosos dentro de la plantilla HTML).
   * **Barras Verdes (< 2ms)**: Componentes optimizados y eficientes.

---

## 10.5 Optimización Web Vital: `NgOptimizedImage`

En el desarrollo frontend profesional, la carga rápida de imágenes y assets visuales es un factor determinante para mejorar las métricas de posicionamiento SEO de Google llamadas **Core Web Vitals** (específicamente la métrica de **Largest Contentful Paint - LCP** y la de **Cumulative Layout Shift - CLS**).

Angular moderno proporciona la directiva integrada **`NgOptimizedImage`** para automatizar todas las optimizaciones complejas de carga de imágenes en producción.

### Ventajas de usar `NgOptimizedImage`:
* **Carga Perezosa Automática (Lazy Loading)**: Las imágenes se descargan únicamente cuando están a punto de aparecer en la pantalla del usuario (viewport).
* **Priorización de Imágenes Críticas**: Permite marcar imágenes críticas (como el banner inicial de la web) con la bandera `priority` para que se descarguen de inmediato, reduciendo drásticamente el LCP.
* **Previene Desplazamientos de Diseño (CLS)**: Obliga a declarar las dimensiones (`width` y `height`), reservando el espacio exacto en el layout y evitando saltos bruscos en la pantalla mientras las imágenes cargan.
* **Optimización de Calidad y Formatos**: Genera de forma automatizada las etiquetas `srcset` para servir imágenes responsive adaptadas a la resolución de pantalla del usuario.

### Implementación Práctica:

Importamos la directiva en nuestro componente:

```typescript
import { Component } from "@angular/core";
import { NgOptimizedImage } from "@angular/common"; // Importación obligatoria

@Component({
  selector: "app-banner-producto",
  standalone: true,
  imports: [NgOptimizedImage],
  template: `
    <div class="banner-container">
      <!-- 1. Imagen Crítica de Cabecera (Con prioridad y LCP optimizado) -->
      <img ngSrc="assets/banners/heros-computador.webp" 
           width="1200" 
           height="400" 
           priority 
           alt="Última tecnología de computadores de escritorio" 
           class="rounded shadow">

      <!-- 2. Imagen Secundaria (Con Lazy Loading automático implícito) -->
      <img ngSrc="assets/productos/mouse-gamer.png" 
           width="300" 
           height="300" 
           alt="Mouse gamer ultra rápido con luces RGB">
    </div>
  `
})
export class BannerProductoComponent {}
```

---

## Resumen del Capítulo

* Comprender el **Ciclo de Vida** de los componentes permite decidir con exactitud técnica dónde inyectar lógica de negocio (`ngOnInit`), dónde manipular el DOM físico (`ngAfterViewInit`) y dónde prevenir memory leaks (`ngOnDestroy`).
* La estrategia **`OnPush`** es el estándar mandatorio de alto rendimiento en producción, reduciendo el overhead de detección al ignorar ramas completas estáticas del árbol de componentes.
* **Zoneless Angular** representa una revolución total al desacoplarse de `zone.js` mediante la función `provideExperimentalZonelessChangeDetection()`, logrando bundles ultra ligeros y renders extremadamente rápidos guiados por Signals.
* La herramienta **Angular DevTools** nos permite perfilar, capturar e identificar cuellos de botella visuales en tiempo real a través de métricas legibles de duración.
* La directiva **`NgOptimizedImage`** automatiza la carga de assets visuales protegiendo las Web Vitals críticas como LCP y CLS mediante priorizaciones avanzadas.

En el próximo capítulo, aprenderemos a optimizar el renderizado visual de carga diferida dominando la sintaxis declarativa más avanzada de Angular: **Vistas Diferibles (`@defer`) en Profundidad**.
