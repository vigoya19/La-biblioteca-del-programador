# Capítulo 12: Server-Side Rendering (SSR), SSG e Hidratación

En los inicios del desarrollo de aplicaciones web de una sola página (SPA), el navegador del usuario descargaba un archivo HTML prácticamente vacío (con una sola etiqueta contenedora como `<app-root></app-root>`) acompañado de pesados archivos JavaScript. El navegador debía procesar y ejecutar todo el código JS antes de poder pintar e interactuar con la interfaz visual. Esto es lo que conocemos como **Renderizado del Lado del Cliente (Client-Side Rendering - CSR)**.

Aunque excelente para aplicaciones internas cerradas, el enfoque CSR tradicional presenta dos problemas graves:
1. **Pobre Indexación SEO**: Los motores de búsqueda de internet (como Google, Bing o Yahoo) leen un archivo HTML en blanco antes de ejecutar JavaScript. Esto dificulta enormemente la indexación de catálogos y artículos de blogs, sepultando tu sitio en las últimas posiciones de búsqueda.
2. **Largos Tiempos de Carga Inicial**: En smartphones de gama media o baja conectados a redes móviles lentas, el usuario experimenta una "pantalla blanca" incómoda de varios segundos antes de ver el primer píxel del sitio web en pantalla.

Para solucionar esto de raíz a nivel empresarial, Angular incorpora soporte de primer nivel para **Server-Side Rendering (SSR)**, **Static Site Generation (SSG)** y el motor de **hidratación completa libre de parpadeos**.

En este capítulo, aprenderemos cómo configurar e implementar SSR y SSG en Angular moderno, comprenderemos el funcionamiento revolucionario de la hidratación completa de cliente y dominaremos las directrices de seguridad indispensables para interactuar con APIs del navegador de forma segura en entornos de servidor.

---

## 12.1 ¿Qué es SSR y cómo revoluciona el SEO?

El **Renderizado del Lado del Servidor (Server-Side Rendering - SSR)** consiste en ejecutar la aplicación Angular en un servidor web intermedio (típicamente impulsado por Node.js) ante cada petición física del usuario en el navegador.

El servidor Node.js compila el árbol de componentes, ejecuta la lógica necesaria para obtener los datos desde bases de datos remotas y genera un **documento HTML físico 100% completo y renderizado con todos los textos y datos reales**. 

El servidor envía este HTML estructurado de inmediato al cliente.

```
Enfoque CSR (Client-Side Rendering):
Navegador solicita URL ──► Servidor retorna HTML vacío ──► Descarga JS ──► Renderiza en cliente (Pantalla blanca inicial)

Enfoque SSR (Server-Side Rendering):
Navegador solicita URL ──► Servidor Node.js compila Angular ──► Retorna HTML pre-renderizado con datos ──► Pintado instantáneo (SEO perfecto)
```

### Métricas de Web Vitals Beneficiadas por SSR:
* **First Contentful Paint (FCP)** y **Largest Contentful Paint (LCP)**: Se reducen drásticamente debido a que el primer frame visual se dibuja en milisegundos en la pantalla del usuario.
* **Search Engine Optimization (SEO)**: Los indexadores de internet leen el contenido completo de manera instantánea, indexando imágenes, headers, palabras clave y metadatos dinámicos sin ningún tipo de fricción técnica.

---

## 12.2 Configuración e Inicialización de SSR con Angular CLI

En el Angular moderno, activar e instalar el soporte para Server-Side Rendering es ridículamente sencillo y se automatiza directamente utilizando comandos del Angular CLI.

### 1. Activar SSR en un Proyecto Existente
Abrimos la terminal en el directorio raíz de la aplicación e introducimos el siguiente comando oficial de andamiaje:

```bash
ng add @angular/ssr
```

El CLI realizará de forma automática múltiples modificaciones estructurales en tu base de código:
* Creará un archivo de arranque específico para el servidor Node.js: `server.ts`.
* Creará el punto de entrada de arranque de servidor en TypeScript: `main.server.ts`.
* Configurará el archivo de configuración global `app.config.server.ts` para proveer los servicios del lado del servidor.
* Actualizará `angular.json` para dar soporte a compilaciones duales (para navegador y para servidor).

### 2. Probar la Aplicación en Servidor en Local
Para iniciar el servidor de desarrollo local con SSR activado, simplemente ejecutamos:

```bash
npm run dev
```

---

## 12.3 El Nuevo Motor de Hidratación Completa (Hydration)

Tradicionalmente, en versiones antiguas de Angular (Universal), el servidor enviaba el HTML estático renderizado al cliente. Sin embargo, en cuanto el bundle de JavaScript terminaba de descargarse en el navegador, Angular destruía por completo todo el DOM estático enviado por el servidor y lo volvía a crear de cero en el cliente para enganchar sus eventos interactivos. Esto provocaba un molesto **parpadeo visual (flickering)** que arruinaba la experiencia de usuario premium.

En Angular moderno, esto se soluciona mediante la **Hidratación Completa del Lado del Cliente (Full Client-Side Hydration)**.

### ¿Cómo funciona la Hidratación Completa?
Cuando activas la hidratación, Angular ya no es destructivo en el cliente:
1. El navegador dibuja el HTML estático perfecto del servidor de forma instantánea.
2. JavaScript se descarga en segundo plano.
3. Al arrancar, el compilador de Angular recorre el árbol del DOM estático existente, **"adopta" los nodos existentes** y engancha los escuchas de eventos y el motor reactivo de Signals de manera invisible y transparente.
4. **Cero Parpadeos Visuales**: La transición de estático a interactivo es 100% fluida y libre de parpadeos.

### Activación en `app.config.ts`
El motor de hidratación viene activado por defecto al andamiar SSR con el CLI moderno. Se configura en la provisión del cliente utilizando la función **`withClientHydration()`**:

```typescript
import { ApplicationConfig } from "@angular/core";
import { provideClientHydration } from "@angular/platform-browser";
import { provideRouter } from "@angular/router";
import { routes } from "./app.routes";

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    // Habilita la hidratación no destructiva de alto rendimiento
    provideClientHydration() 
  ]
};
```

---

## 12.4 Generación de Sitios Estáticos (SSG / Prerendering)

¿Qué pasa si nuestro sitio web es estático (por ejemplo, landing pages corporativas, términos de condiciones o páginas de blog) y no cambia para cada usuario individual? Hacer que un servidor Node.js ejecute y compile Angular en tiempo real ante cada petición de red es un gasto innecesario de infraestructura y computación.

Para estos escenarios, Angular proporciona **SSG (Static Site Generation)** o **Prerendering**.

Consiste en compilar y renderizar todas las páginas estáticas de la aplicación **exactamente una vez en tiempo de compilación (build time)**, generando archivos HTML físicos estáticos que se guardan en el directorio de salida. Puedes subirlos directamente a servicios de almacenamiento estático ultra rápidos y económicos como Firebase Hosting, Netlify, AWS S3 o Vercel.

### 1. Prerenderizar Rutas Estáticas
Durante el proceso de build de producción (`ng build`), el compilador de Angular analizará tus rutas estáticas y generará los archivos HTML físicos de forma automática.

```bash
# Compilar la aplicación y generar los prerenders estáticos
ng build
```

---

### 2. Prerenderizar Rutas Dinámicas con Parámetros
Si tienes rutas dinámicas (como `/productos/:id`), el compilador no sabe de antemano qué identificadores de productos existen en tu base de datos para generar sus respectivos archivos HTML físicos. 

Para resolver esto, podemos proveer un archivo de texto plano conteniendo la lista de rutas parametrizadas que queremos pre-renderizar estáticamente:

1. Creamos un archivo en la raíz del proyecto llamado `rutas-prerender.txt`:
```
/productos/102
/productos/105
/productos/109
/productos/abc-laptops
```

2. Registramos el archivo de rutas dinámicas en tu configuración de `angular.json` o ejecutamos directamente el script de prerenderizado:
```bash
# Ejecutar prerendering pasándole los parámetros dinámicos
ng run mi-app:prerender --routes-file rutas-prerender.txt
```

---

## 12.5 Consideraciones de Seguridad en SSR: APIs del Navegador

Uno de los errores más comunes cometidos por los desarrolladores que implementan SSR por primera vez es intentar acceder directamente a objetos globales y APIs exclusivas del navegador web, tales como **`window`**, **`document`**, **`localStorage`** o **`sessionStorage`**.

### El Peligro del Entorno Servidor
Cuando tu aplicación Angular se compila e inicializa en el servidor Node.js durante una petición SSR, **estos objetos del navegador no existen físicamente en la memoria de Node**. Intentar invocar un `window.innerWidth` o un `localStorage.getItem()` provocará que el servidor Node.js colapse inmediatamente con un error de ejecución de excepción: `ReferenceError: window is not defined`, devolviendo un código de error de red al cliente y rompiendo tu SEO.

### Cómo Escribir Código Seguro para Servidor
Para evitar fallos de renderizado en servidor, debemos verificar mediante inyección si el componente se está ejecutando en el entorno físico del **Navegador** o del **Servidor** antes de invocar APIs exclusivas:

```typescript
import { Component, OnInit, inject, PLATFORM_ID } from "@angular/core";
import { isPlatformBrowser, isPlatformServer } from "@angular/common";

@Component({
  selector: "ui-tarjeta-analytics",
  standalone: true,
  template: `<p>Visualizando Sección Interactiva</p>`
})
export class TarjetaAnalyticsComponent implements OnInit {
  // 1. Inyectamos el identificador único de plataforma
  private readonly platformId = inject(PLATFORM_ID);

  ngOnInit(): void {
    // 2. Verificamos de forma segura si nos encontramos físicamente en el navegador
    if (isPlatformBrowser(this.platformId)) {
      console.log("Código seguro: Ejecutándose en el cliente.");
      
      // Es 100% seguro acceder a objetos globales del navegador aquí
      const anchoPantalla = window.innerWidth;
      localStorage.setItem("ancho", anchoPantalla.toString());
    }

    // 3. Verificamos si estamos ejecutándonos en el servidor Node.js
    if (isPlatformServer(this.platformId)) {
      console.log("Código seguro: Ejecutándose en el servidor Node.js de la empresa.");
      // Evita lógica visual del navegador aquí. Lógica puramente analítica.
    }
  }
}
```

---

## Resumen del Capítulo

* **Server-Side Rendering (SSR)** compila y renderiza tus vistas en un servidor Node.js intermedio, sirviendo HTML 100% estructurado ideal para maximizar el posicionamiento SEO y el FCP de Core Web Vitals.
* La **Hidratación Completa (Client-side Hydration)** es un mecanismo no destructivo de acoplamiento de eventos que adopta el DOM estático del servidor en el cliente de forma transparente y sin parpadeos visuales (flickering).
* **Static Site Generation (SSG)** o Prerendering pre-compila tus rutas en archivos HTML físicos en tiempo de build, eliminando costos innecesarios de infraestructura de servidor para contenido estático.
* Es fundamental utilizar **`isPlatformBrowser`** y **`isPlatformServer`** junto con **`PLATFORM_ID`** para blindar la aplicación contra fallos catastróficos de ejecución al consumir APIs exclusivas del navegador.

En el próximo capítulo, aprenderemos cómo asegurar la calidad de nuestro software y prevenir regresiones de código dominando el **Testing Unitario y de Integración con Vitest** y **Playwright**.

---

← [Capítulo anterior](11-vistas-diferibles.md) | [Inicio](README.md) | [Capítulo siguiente →](13-testing-en-angular.md)
