# Capítulo 8: Comunicación Asíncrona y HttpClient

Las aplicaciones web modernas rara vez viven en aislamiento. Para ser verdaderamente útiles, deben interactuar constantemente con servidores externos, consumiendo APIs REST, enviando credenciales de acceso, almacenando registros en bases de datos y descargando archivos multimedia de forma asíncrona.

Angular proporciona un cliente HTTP robusto, seguro y totalmente reactivo llamado **`HttpClient`**. Este cliente no trabaja con Promesas tradicionales de JavaScript; en su lugar, está profundamente integrado con **RxJS (Observables)**, lo que nos otorga un control absoluto sobre la manipulación de flujos de datos asíncronos en tiempo real.

En este capítulo, aprenderemos a configurar el servicio `HttpClient` en aplicaciones modernas independientes utilizando la función **`provideHttpClient()`**. Aprenderemos a estructurar operaciones CRUD con tipado fuerte, dominaremos los operadores de RxJS más críticos para la optimización de peticiones y diseñaremos **Interceptores HTTP Funcionales** para inyectar tokens de seguridad (JWT) y gestionar errores globales de red de forma elegante.

---

## 8.1 Configuración Moderna de `HttpClient` (`provideHttpClient`)

En la arquitectura standalone de Angular v17+, ya no se utiliza el clásico `HttpClientModule` para importar el soporte de peticiones en red. En su lugar, el soporte de red se registra globalmente en `app.config.ts` utilizando la función **`provideHttpClient()`**.

### Registro del Proveedor en `app.config.ts`

Adicionalmente, podemos pasar configuraciones avanzadas como el soporte de interceptores o llamadas con credenciales a través de sub-proveedores:

```typescript
import { ApplicationConfig } from "@angular/core";
import { provideHttpClient, withInterceptors } from "@angular/common/http";
import { tokenInterceptor } from "./core/interceptors/token.interceptor";
import { errorInterceptor } from "./core/interceptors/error.interceptor";

export const appConfig: ApplicationConfig = {
  providers: [
    // Registro del cliente HTTP con soporte para interceptores funcionales
    provideHttpClient(
      withInterceptors([
        tokenInterceptor,
        errorInterceptor
      ])
    )
  ]
};
```

---

## 8.2 Consumo de APIs REST (Operaciones CRUD)

Una vez configurado el proveedor global, podemos inyectar `HttpClient` en nuestros servicios mediante la función `inject()` y realizar peticiones asíncronas con tipado estricto para garantizar la robustez del código.

Definimos primero una interfaz para tipar el recurso de nuestra API corporativa:

```typescript
export interface Producto {
  id: string;
  nombre: string;
  precio: number;
  categoria: string;
}
```

### Implementación de un Servicio CRUD Completo:

```typescript
import { Injectable, inject } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { Observable } from "rxjs";
import { Producto } from "../models/producto.interface";

@Injectable({
  providedIn: "root"
})
export class ProductoService {
  // Inyección del cliente HTTP funcional
  private readonly http = inject(HttpClient);
  private readonly apiUrl = "https://api.empresa.com/v1/productos";

  // 1. GET (Obtener todos los productos)
  obtenerTodos(): Observable<Producto[]> {
    // La petición HTTP está fuertemente tipada de forma genérica <Producto[]>
    return this.http.get<Producto[]>(this.apiUrl);
  }

  // 2. GET BY ID (Obtener un único registro)
  obtenerPorId(id: string): Observable<Producto> {
    return this.http.get<Producto>(`${this.apiUrl}/${id}`);
  }

  // 3. POST (Crear un registro)
  crear(producto: Omit<Producto, "id">): Observable<Producto> {
    return this.http.post<Producto>(this.apiUrl, producto);
  }

  // 4. PUT (Actualizar registro completo)
  actualizar(id: string, producto: Producto): Observable<Producto> {
    return this.http.put<Producto>(`${this.apiUrl}/${id}`, producto);
  }

  // 5. DELETE (Eliminar registro de la base de datos)
  eliminar(id: string): Observable<void> {
    return this.http.delete<void>(`${`${this.apiUrl}/${id}`}`);
  }
}
```

---

## 8.3 Operadores Esenciales de RxJS para Peticiones HTTP

Dado que `HttpClient` retorna Observables de RxJS, podemos aplicar **operadores de tubería (`pipe()`)** para realizar transformaciones asíncronas, optimizar el rendimiento de red y gestionar errores de forma fina.

### 1. `switchMap` (Evitar "Race Conditions" en Búsquedas)
Cancela inmediatamente la petición HTTP previa que esté en curso si el usuario realiza una nueva acción. Es mandatorio para motores de búsqueda interactiva o autocompletado en tiempo real.

```typescript
// Cancela la petición anterior si el término de búsqueda cambia
buscarProductos(termino$: Observable<string>): Observable<Producto[]> {
  return termino$.pipe(
    switchMap((termino) => this.http.get<Producto[]>(`${this.apiUrl}?q=${termino}`))
  );
}
```

---

### 2. `shareReplay` (Cachear Peticiones HTTP Repetidas)
Evita realizar peticiones idénticas repetidas a la base de datos compartiendo y transmitiendo la última respuesta exitosa almacenada en caché a todos los componentes que se suscriban al observable.

```typescript
import { shareReplay } from "rxjs/operators";

@Injectable({ providedIn: 'root' })
export class CategoriaService {
  private readonly http = inject(HttpClient);

  // La petición se ejecutará una única vez física por sesión
  readonly categorias$ = this.http.get<string[]>("/api/categorias").pipe(
    shareReplay(1) // Almacena y comparte la última emisión exitosa
  );
}
```

---

### 3. `retry` y `catchError` (Reintentos y Manejo de Errores)
* `retry`: Si la conexión de red falla momentáneamente, reintenta automáticamente la petición HTTP el número de veces indicado antes de reportar un fallo definitivo.
* `catchError`: Intercepta cualquier excepción o código de estado de error (4xx/5xx) de HTTP, permitiendo manejarlo localmente y retornar un flujo seguro o lanzar una excepción formateada.

```typescript
import { catchError, retry } from "rxjs/operators";
import { of, throwError } from "rxjs";

obtenerDatosRespaldo(): Observable<Producto[]> {
  return this.http.get<Producto[]>(this.apiUrl).pipe(
    retry(2), // Reintenta 2 veces la petición ante caídas temporales de red
    catchError((error) => {
      console.error("Excepción detectada en llamada HTTP:", error);
      // Retornamos un flujo seguro con datos por defecto para que la app no explote
      return of([{ id: "0", nombre: "Producto de Respaldo", precio: 0, categoria: "Default" }]);
    })
  );
}
```

---

## 8.4 Interceptores HTTP Funcionales

Un **Interceptor** es un middleware de red que se sitúa entre tu aplicación Angular y el servidor externo. Permite inspeccionar, clonar y modificar todas las solicitudes salientes o las respuestas entrantes de manera global antes de que lleguen a su destino.

En el Angular moderno, los interceptores basados en clases están deprecados y se reemplazan por **Interceptores Funcionales** ultra eficientes.

### 1. Interceptor de Autenticación JWT (`tokenInterceptor`)

Este interceptor inyecta de forma automática el encabezado `Authorization: Bearer <token>` en todas las solicitudes que van hacia servidores seguros si el usuario cuenta con una sesión iniciada.

```typescript
import { HttpInterceptorFn } from "@angular/common/http";
import { inject } from "@angular/core";
import { AuthService } from "../services/auth.service";

export const tokenInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = localStorage.getItem("authToken");

  console.log(`[HTTP INTERCEPTOR]: Procesando solicitud hacia: ${req.url}`);

  // Si tenemos un token guardado, clonamos la petición inyectando el header de seguridad
  if (token) {
    const solicitudClonada = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });
    
    // Continuamos la cadena de interceptores con la petición modificada
    return next(solicitudClonada);
  }

  // Si no hay token, la petición continúa su flujo original sin modificaciones
  return next(req);
};
```

---

### 2. Interceptor Global de Errores (`errorInterceptor`)

Este interceptor captura de forma centralizada cualquier error de red (por ejemplo, token expirado con código `401 Unauthorized` o servidor caído `500 Internal Server Error`) y ejecuta acciones correctivas globales.

```typescript
import { HttpInterceptorFn, HttpErrorResponse } from "@angular/common/http";
import { inject } from "@angular/core";
import { Router } from "@angular/router";
import { catchError } from "rxjs/operators";
import { throwError } from "rxjs";

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      // Capturamos el código de estado HTTP
      switch (error.status) {
        case 401:
          console.error("[ERROR 401]: Sesión no autorizada o expirada. Redireccionando...");
          localStorage.removeItem("authToken");
          router.navigate(["/login"]);
          break;
        case 403:
          console.error("[ERROR 403]: Acceso prohibido a este recurso.");
          router.navigate(["/sin-permiso"]);
          break;
        case 500:
          console.error("[ERROR 500]: Fallo grave en el servidor backend.");
          break;
        default:
          console.error(`[HTTP EXCEPTION]: Error código ${error.status}: ${error.message}`);
      }

      // Propagamos el error para que el servicio o componente que llamó también lo conozca
      return throwError(() => new Error(error.message));
    })
  );
};
```

---

## 8.5 Gestión Robusta de Errores y Cancelación de Peticiones

En aplicaciones empresariales de gran escala, dejar suscripciones HTTP abiertas puede provocar fugas de memoria silenciosas. Aunque las peticiones HTTP (`GET`, `POST`) de Angular emiten una única vez y se completan automáticamente (lo que previene fugas en la mayoría de los casos), existen escenarios donde el usuario navega fuera de una pantalla *antes* de que la petición de datos del servidor haya finalizado.

Podemos utilizar la función `takeUntilDestroyed` (del módulo `@angular/core/rxjs-interop`) para cancelar automáticamente peticiones HTTP activas si el componente se destruye del DOM:

```typescript
import { Component, OnInit, inject } from "@angular/core";
import { takeUntilDestroyed } from "@angular/core/rxjs-interop";
import { ProductoService } from "./producto.service";
import { DestroyRef } from "@angular/core";

@Component({
  selector: "app-perfil-empresa",
  standalone: true,
  template: `<p>Visualizando Datos Financieros...</p>`
})
export class PerfilEmpresaComponent implements OnInit {
  private readonly prodService = inject(ProductoService);
  // Servicio que encapsula la referencia de destrucción del componente
  private readonly destroyRef = inject(DestroyRef); 

  ngOnInit(): void {
    this.prodService.obtenerTodos().pipe(
      // Cancela físicamente la llamada HTTP en red si el usuario sale de esta pantalla
      // antes de que el servidor remoto haya respondido.
      takeUntilDestroyed(this.destroyRef) 
    ).subscribe((datos) => {
      console.log("Datos del servidor recibidos:", datos);
    });
  }
}
```

---

## Resumen del Capítulo

* El servicio reactivo **`HttpClient`** se inicializa en aplicaciones modernas mediante **`provideHttpClient()`** en `app.config.ts`.
* La API está integrada con **RxJS**, retornando Observables síncronos optimizados que simplifican el tipado estricto de llamadas de red.
* Operadores críticos como **`switchMap`** cancelan flujos redundantes para evitar colisiones de red, mientras que **`shareReplay`** optimiza la memoria a través de cachés inteligentes.
* Los **Interceptores HTTP Funcionales** representan el middleware moderno de red, permitiendo inyectar tokens JWT Bearer de forma invisible y centralizar la gestión de errores globales de red.
* La API **`takeUntilDestroyed`** proporciona una estrategia de protección de red al cancelar peticiones en vuelo ante la salida prematura del usuario del componente.

En el próximo capítulo, aprenderemos a gestionar flujos de datos interconectados a nivel de negocio y a construir tiendas reactivas complejas dominando la **Gestión de Estado** con **NgRx Signals Store**.

---

← [Capítulo anterior](07-enrutamiento-y-navegacion.md) | [Inicio](README.md) | [Capítulo siguiente →](09-gestion-de-estado.md)
