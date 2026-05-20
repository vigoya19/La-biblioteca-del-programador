# Capítulo 19: Progressive Web Apps (PWAs)

> "Una PWA no es solo una web que se puede instalar. Es una aplicación que funciona offline, se actualiza silenciosamente y ofrece una experiencia nativa sin pasar por una tienda de aplicaciones."

Las **Progressive Web Apps** representan la convergencia entre las aplicaciones web y las aplicaciones nativas. Con una PWA, tus usuarios pueden instalar tu aplicación Angular en su pantalla de inicio, recibir notificaciones push, trabajar sin conexión a internet y disfrutar de tiempos de carga instantáneos gracias al cacheo inteligente del Service Worker.

Angular tiene soporte de primera clase para PWAs a través del paquete `@angular/pwa`, que configura automáticamente el Service Worker, genera el Web App Manifest y establece las estrategias de cacheo.

---

## 19.1 ¿Qué es una PWA y Por Qué Angular es Ideal?

Una PWA cumple tres criterios fundamentales:

1. **Confiable**: Se carga instantáneamente y funciona offline (o con conexión inestable).
2. **Rápida**: Responde de forma fluida a las interacciones del usuario.
3. **Instalable**: Se puede añadir a la pantalla de inicio como una app nativa.

### Tecnologías Base de una PWA

| Tecnología | Función |
|---|---|
| **Service Worker** | Proxy JavaScript entre la app y la red. Intercepta peticiones, cachea recursos y permite funcionalidad offline. |
| **Web App Manifest** | Archivo JSON que define nombre, iconos, colores y comportamiento de la app cuando se instala. |
| **HTTPS** | Requisito obligatorio. Los Service Workers solo funcionan sobre HTTPS (excepto `localhost`). |

### ¿Por Qué Angular?

- `@angular/service-worker` proporciona un Service Worker configurable por archivo JSON (sin escribir código de SW manual).
- `@angular/pwa` automatiza toda la configuración inicial con un schematic.
- La arquitectura de compilación de Angular genera hashes únicos por archivo, facilitando la invalidación de caché.

---

## 19.2 Configuración con `@angular/pwa`

### Instalación

```bash
ng add @angular/pwa
```

Este comando realiza automáticamente:
1. Instala `@angular/service-worker`
2. Genera `ngsw-config.json` (configuración del Service Worker)
3. Genera `manifest.webmanifest` con metadatos de la app
4. Genera iconos placeholder en múltiples resoluciones
5. Registra el Service Worker en `app.config.ts`

### Configuración del Service Worker: `ngsw-config.json`

```json
{
  "$schema": "./node_modules/@angular/service-worker/config/schema.json",
  "index": "/index.html",
  
  "assetGroups": [
    {
      "name": "app-shell",
      "installMode": "prefetch",
      "updateMode": "prefetch",
      "resources": {
        "files": [
          "/favicon.ico",
          "/index.html",
          "/manifest.webmanifest",
          "/*.css",
          "/*.js"
        ]
      }
    },
    {
      "name": "assets",
      "installMode": "lazy",
      "updateMode": "prefetch",
      "resources": {
        "files": [
          "/assets/**",
          "/*.(svg|cur|jpg|jpeg|png|apng|webp|avif|gif|otf|ttf|woff|woff2)"
        ]
      }
    }
  ],
  
  "dataGroups": [
    {
      "name": "api-productos",
      "urls": ["/api/productos/**"],
      "cacheConfig": {
        "strategy": "freshness",
        "maxSize": 100,
        "maxAge": "1h",
        "timeout": "5s"
      }
    },
    {
      "name": "api-categorias",
      "urls": ["/api/categorias"],
      "cacheConfig": {
        "strategy": "performance",
        "maxSize": 10,
        "maxAge": "1d"
      }
    }
  ]
}
```

---

## 19.3 Estrategias de Caché

### `assetGroups`: Recursos Estáticos

| Modo | Comportamiento | Uso |
|---|---|---|
| `prefetch` (installMode) | Descarga y cachea TODOS los recursos inmediatamente al instalar el SW | App shell (HTML, CSS, JS principales) |
| `lazy` (installMode) | Solo cachea recursos cuando el usuario los solicita por primera vez | Imágenes, fuentes, assets grandes |

### `dataGroups`: Datos Dinámicos (APIs)

| Estrategia | Comportamiento | Uso |
|---|---|---|
| **`performance`** (Cache-first) | Sirve desde caché primero. Solo consulta la red si no hay caché o ha expirado (`maxAge`). | Datos que cambian poco: categorías, configuraciones, traducciones |
| **`freshness`** (Network-first) | Consulta la red primero. Si la red falla o tarda más de `timeout`, sirve desde caché. | Datos que cambian frecuentemente: productos, precios, inventario |

```
Estrategia "performance" (Cache-first):
  Usuario → ¿En caché? → SÍ → Devolver caché (instantáneo)
                        → NO → Consultar red → Cachear → Devolver

Estrategia "freshness" (Network-first):
  Usuario → Consultar red → ¿Respuesta en < timeout? → SÍ → Devolver y cachear
                                                       → NO → Devolver caché
```

---

## 19.4 Actualizaciones de la Aplicación con `SwUpdate`

Cuando despliegas una nueva versión de tu aplicación, el Service Worker la detecta automáticamente en segundo plano. El servicio `SwUpdate` permite controlar cómo y cuándo se aplica la actualización.

```typescript
import { Component, inject, OnInit } from "@angular/core";
import { SwUpdate, VersionReadyEvent } from "@angular/service-worker";
import { filter } from "rxjs";

@Component({
  selector: "app-root",
  standalone: true,
  template: `
    <div class="app-container">
      @if (nuevaVersionDisponible) {
        <div class="fixed bottom-4 right-4 bg-blue-600 text-white p-4 rounded-lg shadow-xl z-50 
                    flex items-center gap-3 max-w-sm animate-slide-up">
          <span>🔄</span>
          <div>
            <p class="font-bold text-sm">Nueva versión disponible</p>
            <p class="text-xs opacity-80">Actualiza para obtener las últimas mejoras.</p>
          </div>
          <button (click)="actualizarApp()" 
                  class="px-3 py-1 bg-white text-blue-600 rounded text-sm font-bold hover:bg-blue-50">
            Actualizar
          </button>
        </div>
      }
      <router-outlet />
    </div>
  `
})
export class AppComponent implements OnInit {
  private readonly swUpdate = inject(SwUpdate);
  nuevaVersionDisponible = false;

  ngOnInit(): void {
    if (this.swUpdate.isEnabled) {
      // Escuchar cuando hay una nueva versión lista para activar
      this.swUpdate.versionUpdates.pipe(
        filter((event): event is VersionReadyEvent => event.type === "VERSION_READY")
      ).subscribe((event) => {
        console.log(`Versión actual: ${event.currentVersion.hash}`);
        console.log(`Nueva versión: ${event.latestVersion.hash}`);
        this.nuevaVersionDisponible = true;
      });

      // Comprobar actualizaciones cada 30 minutos
      setInterval(() => {
        this.swUpdate.checkForUpdate().then(found => {
          if (found) console.log("Nueva versión detectada");
        });
      }, 30 * 60 * 1000);
    }
  }

  actualizarApp(): void {
    // Recargar la página para activar la nueva versión del Service Worker
    document.location.reload();
  }
}
```

### Manejo de Versiones Rotas

Si una versión desplegada tiene un error crítico que impide la carga, Angular lo detecta automáticamente:

```typescript
this.swUpdate.unrecoverable.subscribe(event => {
  console.error("Versión irrecuperable:", event.reason);
  
  // Forzar recarga limpia sin caché
  if (confirm("La aplicación necesita recargarse para funcionar correctamente. ¿Recargar ahora?")) {
    document.location.reload();
  }
});
```

---

## 19.5 Push Notifications con `SwPush`

Las notificaciones push permiten re-engagement del usuario incluso cuando la aplicación no está abierta:

```typescript
import { Component, inject } from "@angular/core";
import { SwPush } from "@angular/service-worker";
import { HttpClient } from "@angular/common/http";

@Component({
  selector: "app-notificaciones",
  standalone: true,
  template: `
    <button (click)="suscribirse()" class="px-4 py-2 bg-purple-600 text-white rounded">
      🔔 Activar notificaciones
    </button>
  `
})
export class NotificacionesComponent {
  private readonly swPush = inject(SwPush);
  private readonly http = inject(HttpClient);

  // Clave pública VAPID (generar con: npx web-push generate-vapid-keys)
  private readonly VAPID_PUBLIC_KEY = "BLn...tu-clave-publica-aqui";

  async suscribirse(): Promise<void> {
    try {
      const subscription = await this.swPush.requestSubscription({
        serverPublicKey: this.VAPID_PUBLIC_KEY
      });
      
      console.log("Suscripción creada:", JSON.stringify(subscription));
      
      // Enviar la suscripción al backend para que pueda enviar pushes
      this.http.post("/api/push-subscriptions", subscription).subscribe({
        next: () => console.log("Suscripción registrada en el servidor"),
        error: (err) => console.error("Error registrando suscripción:", err)
      });
    } catch (err) {
      console.error("Error al suscribirse a push notifications:", err);
    }
  }
}
```

### Manejar Notificaciones Recibidas

```typescript
// En app.component.ts
this.swPush.messages.subscribe((message: any) => {
  console.log("Push notification recibida:", message);
});

// Manejar clic en la notificación
this.swPush.notificationClicks.subscribe(({ action, notification }) => {
  console.log("Notificación clicada:", notification.title);
  if (notification.data?.url) {
    window.open(notification.data.url, "_self");
  }
});
```

---

## 19.6 Soporte Offline

### Página de Fallback Offline

Configura una página que se muestre cuando el usuario está completamente offline y la ruta solicitada no está en caché:

```json
// ngsw-config.json
{
  "navigationUrls": [
    "/**",
    "!/__/**"
  ],
  "navigationRequestStrategy": "freshness"
}
```

### Detección del Estado de Conexión

```typescript
import { Injectable, signal, effect } from "@angular/core";

@Injectable({ providedIn: "root" })
export class ConexionService {
  readonly estaOnline = signal(navigator.onLine);

  constructor() {
    window.addEventListener("online", () => this.estaOnline.set(true));
    window.addEventListener("offline", () => this.estaOnline.set(false));
    
    effect(() => {
      if (!this.estaOnline()) {
        console.log("[OFFLINE] La aplicación está funcionando sin conexión");
      } else {
        console.log("[ONLINE] Conexión restaurada");
      }
    });
  }
}
```

```html
<!-- Banner de offline en el AppComponent -->
@if (!conexion.estaOnline()) {
  <div class="bg-amber-500 text-white text-center py-2 text-sm font-semibold">
    ⚠️ Estás sin conexión. Algunos datos pueden no estar actualizados.
  </div>
}
```

---

## 19.7 Instalación de la PWA

### Evento `beforeinstallprompt`

Los navegadores Chromium disparan este evento cuando la PWA cumple los criterios de instalabilidad. Puedes capturarlo para mostrar tu propia UI de instalación personalizada:

```typescript
import { Injectable, signal } from "@angular/core";

@Injectable({ providedIn: "root" })
export class PwaInstalacionService {
  private deferredPrompt: any = null;
  readonly puedeInstalar = signal(false);

  constructor() {
    window.addEventListener("beforeinstallprompt", (event: Event) => {
      event.preventDefault(); // Prevenir el prompt automático del navegador
      this.deferredPrompt = event;
      this.puedeInstalar.set(true);
    });

    window.addEventListener("appinstalled", () => {
      this.puedeInstalar.set(false);
      console.log("PWA instalada exitosamente");
    });
  }

  async instalar(): Promise<void> {
    if (!this.deferredPrompt) return;
    
    this.deferredPrompt.prompt();
    const resultado = await this.deferredPrompt.userChoice;
    console.log("Resultado de instalación:", resultado.outcome);
    
    this.deferredPrompt = null;
    this.puedeInstalar.set(false);
  }
}
```

```html
<!-- Botón de instalación personalizado -->
@if (pwaService.puedeInstalar()) {
  <button (click)="pwaService.instalar()" class="flex items-center gap-2 px-4 py-2 bg-green-600 text-white rounded-lg">
    <span>📲</span>
    <span>Instalar TechStore</span>
  </button>
}
```

---

## 19.8 Web App Manifest

```json
// manifest.webmanifest
{
  "name": "TechStore - Tu Tienda de Tecnología",
  "short_name": "TechStore",
  "description": "Encuentra los mejores productos de tecnología al mejor precio",
  "start_url": "/catalogo",
  "display": "standalone",
  "orientation": "portrait-primary",
  "background_color": "#ffffff",
  "theme_color": "#2563eb",
  "categories": ["shopping", "technology"],
  "icons": [
    { "src": "assets/icons/icon-72x72.png", "sizes": "72x72", "type": "image/png" },
    { "src": "assets/icons/icon-96x96.png", "sizes": "96x96", "type": "image/png" },
    { "src": "assets/icons/icon-128x128.png", "sizes": "128x128", "type": "image/png" },
    { "src": "assets/icons/icon-144x144.png", "sizes": "144x144", "type": "image/png" },
    { "src": "assets/icons/icon-192x192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "assets/icons/icon-512x512.png", "sizes": "512x512", "type": "image/png", "purpose": "any maskable" }
  ],
  "screenshots": [
    { "src": "assets/screenshots/catalogo.png", "sizes": "1280x720", "type": "image/png", "label": "Catálogo de productos" }
  ]
}
```

---

## 19.9 Auditoría PWA con Lighthouse

```bash
# Generar reporte de Lighthouse para PWA
npx lighthouse https://tu-app.com --view --preset=desktop --output-path=./lighthouse-report.html

# O desde Chrome DevTools:
# 1. Abrir DevTools (F12)
# 2. Pestaña "Lighthouse"
# 3. Seleccionar "Progressive Web App"
# 4. Click "Analyze page load"
```

### Criterios de Instalabilidad (Chromium)

| Criterio | Requisito |
|---|---|
| HTTPS | La página se sirve sobre HTTPS |
| Service Worker | Registrado con un handler `fetch` |
| Manifest | Incluye `name`/`short_name`, `start_url`, icono 192px, icono 512px, `display: standalone/fullscreen` |
| No instalada | La app no está ya instalada |

---

## Resumen del Capítulo

* Una **PWA** combina las ventajas de la web (URLs, SEO, sin tienda) con las de las apps nativas (instalación, offline, push notifications).
* **`@angular/pwa`** automatiza la configuración completa: Service Worker, manifest e iconos.
* Las estrategias de caché **`performance`** (cache-first) y **`freshness`** (network-first) se aplican según la naturaleza de cada dato.
* **`SwUpdate`** permite detectar nuevas versiones desplegadas y mostrar al usuario una UI de actualización personalizada.
* **`SwPush`** habilita notificaciones push con VAPID keys para re-engagement del usuario.
* La detección de estado online/offline con `navigator.onLine` y eventos permite mostrar banners informativos y ajustar el comportamiento de la app.
* El evento **`beforeinstallprompt`** permite crear UIs de instalación personalizadas en lugar de depender del prompt nativo del navegador.
* **Lighthouse** audita los criterios PWA y proporciona un score con recomendaciones específicas.

---

← [Capítulo anterior](18-i18n-y-a11y.md) | [Inicio](README.md)
