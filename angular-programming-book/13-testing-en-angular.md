# Capítulo 13: Testing en Angular

> "Un test no es una obligación burocrática. Es un contrato ejecutable que documenta el comportamiento esperado de tu código y te protege de regresiones cuando el mundo cambia."

El testing en Angular es una disciplina que separa a los desarrolladores competentes de los arquitectos de software confiables. Una aplicación sin tests es una aplicación que funciona "hasta que deja de funcionar", y cuando eso ocurre en producción a las 3 AM, no hay documentación ni red de seguridad que te salve.

En este capítulo, no solo aprenderás la teoría del testing — lo harás de forma práctica. Cada sección incluye ejercicios guiados donde escribirás tests reales paso a paso, desde tests unitarios de servicios y componentes hasta tests de extremo a extremo con Playwright. Aprenderás TDD (Test-Driven Development) con un ejemplo completo, y dominarás las herramientas modernas del ecosistema Angular.

---

## 13.1 Filosofía de Testing: Pirámide, ROI y Qué Testear Primero

### La Pirámide de Testing

```
                     ┌─────────────┐
                     │    E2E      │ ← Pocos, lentos, costosos, máxima confianza
                     │ (Playwright)│
                    ┌┴─────────────┴┐
                    │  Integración  │ ← Componentes con dependencias reales
                    │  (TestBed)    │
                   ┌┴───────────────┴┐
                   │   Unitarios     │ ← Muchos, rápidos, baratos, base sólida
                   │ (Vitest/Karma)  │
                   └─────────────────┘
```

### Qué Testear Primero (Prioridad por ROI)

| Prioridad | Qué testear | Por qué | Tipo de test |
|---|---|---|---|
| 🔴 Alta | Servicios con lógica de negocio | Son el corazón de la aplicación | Unitario |
| 🔴 Alta | Stores (NgRx Signals Store) | Gestionan el estado crítico | Unitario |
| 🟡 Media | Componentes Smart (páginas) | Integran múltiples piezas | Integración |
| 🟡 Media | Pipes y Validators personalizados | Lógica pura y determinística | Unitario |
| 🟢 Baja | Componentes Dumb (presentacionales) | Solo renderizan datos | Unitario (si tienen lógica) |
| 🔵 E2E | Flujos críticos de usuario | Verifican el sistema completo | E2E |

---

## 13.2 Configuración de Vitest para Angular

Vitest es un test runner ultrarrápido basado en Vite que reemplaza a Karma/Jasmine con una experiencia de desarrollo moderna.

### Migración de Karma a Vitest

```bash
# Instalar el builder experimental de Angular para Vitest
npm install -D @analogjs/vitest-angular @analogjs/vite-plugin-angular vitest jsdom
```

Crea `vite.config.ts` en la raíz del proyecto:

```typescript
/// <reference types="vitest" />
import { defineConfig } from "vite";
import angular from "@analogjs/vite-plugin-angular";

export default defineConfig({
  plugins: [angular()],
  test: {
    globals: true,
    environment: "jsdom",
    setupFiles: ["src/test-setup.ts"],
    include: ["src/**/*.spec.ts"],
    reporters: ["default", "html"]
  }
});
```

Crea `src/test-setup.ts`:

```typescript
import "@analogjs/vitest-angular/setup-zone";
import { getTestBed } from "@angular/core/testing";
import { BrowserDynamicTestingModule, platformBrowserDynamicTesting } from "@angular/platform-browser-dynamic/testing";

getTestBed().initTestEnvironment(
  BrowserDynamicTestingModule,
  platformBrowserDynamicTesting()
);
```

Añade el script a `package.json`:

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage"
  }
}
```

---

## 13.3 Ejercicio Práctico: TDD — Desarrollo Dirigido por Tests

En este ejercicio, construiremos un `CalculadoraDescuentoService` siguiendo estrictamente el ciclo TDD: **Red → Green → Refactor**.

### El Requisito de Negocio

> "El sistema debe calcular descuentos progresivos: 5% para compras de 100€+, 10% para 500€+, 15% para 1000€+. Los clientes VIP reciben un 5% adicional. El descuento máximo nunca puede superar el 20%."

### Paso 1: RED — Escribir el test ANTES del código

Crea `src/app/core/services/calculadora-descuento.service.spec.ts`:

```typescript
import { TestBed } from "@angular/core/testing";
import { CalculadoraDescuentoService } from "./calculadora-descuento.service";
import { describe, beforeEach, it, expect } from "vitest";

describe("CalculadoraDescuentoService", () => {
  let service: CalculadoraDescuentoService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(CalculadoraDescuentoService);
  });

  describe("calcularDescuento()", () => {
    it("debe retornar 0% para compras menores a 100€", () => {
      expect(service.calcularDescuento(50, false)).toBe(0);
      expect(service.calcularDescuento(99.99, false)).toBe(0);
    });

    it("debe retornar 5% para compras de 100€ a 499.99€", () => {
      expect(service.calcularDescuento(100, false)).toBe(0.05);
      expect(service.calcularDescuento(250, false)).toBe(0.05);
      expect(service.calcularDescuento(499.99, false)).toBe(0.05);
    });

    it("debe retornar 10% para compras de 500€ a 999.99€", () => {
      expect(service.calcularDescuento(500, false)).toBe(0.10);
      expect(service.calcularDescuento(750, false)).toBe(0.10);
    });

    it("debe retornar 15% para compras de 1000€ o más", () => {
      expect(service.calcularDescuento(1000, false)).toBe(0.15);
      expect(service.calcularDescuento(5000, false)).toBe(0.15);
    });

    it("debe sumar 5% adicional para clientes VIP", () => {
      expect(service.calcularDescuento(100, true)).toBe(0.10); // 5% base + 5% VIP
      expect(service.calcularDescuento(500, true)).toBe(0.15); // 10% base + 5% VIP
    });

    it("debe limitar el descuento máximo al 20%", () => {
      // 1000€ = 15% + VIP 5% = 20% (justo en el límite)
      expect(service.calcularDescuento(1000, true)).toBe(0.20);
      // Nunca debe superar 20%
      expect(service.calcularDescuento(50000, true)).toBe(0.20);
    });

    it("debe lanzar error para montos negativos", () => {
      expect(() => service.calcularDescuento(-10, false)).toThrow();
    });
  });

  describe("calcularPrecioFinal()", () => {
    it("debe aplicar el descuento al monto original", () => {
      // 200€ con 5% de descuento = 190€
      expect(service.calcularPrecioFinal(200, false)).toBe(190);
    });

    it("debe aplicar descuento VIP correctamente", () => {
      // 600€ con 15% (10% base + 5% VIP) = 510€
      expect(service.calcularPrecioFinal(600, true)).toBe(510);
    });
  });
});
```

**Ejecuta los tests**: `npx vitest run` — Todos fallarán (RED ✅). Eso es correcto.

### Paso 2: GREEN — Escribir el código mínimo para que los tests pasen

Crea `src/app/core/services/calculadora-descuento.service.ts`:

```typescript
import { Injectable } from "@angular/core";

@Injectable({ providedIn: "root" })
export class CalculadoraDescuentoService {
  private readonly DESCUENTO_MAXIMO = 0.20;
  private readonly BONUS_VIP = 0.05;

  private readonly TRAMOS: { minimo: number; porcentaje: number }[] = [
    { minimo: 1000, porcentaje: 0.15 },
    { minimo: 500, porcentaje: 0.10 },
    { minimo: 100, porcentaje: 0.05 }
  ];

  calcularDescuento(monto: number, esVip: boolean): number {
    if (monto < 0) {
      throw new Error("El monto no puede ser negativo");
    }

    // Buscar el tramo de descuento aplicable
    let descuentoBase = 0;
    for (const tramo of this.TRAMOS) {
      if (monto >= tramo.minimo) {
        descuentoBase = tramo.porcentaje;
        break;
      }
    }

    const descuentoTotal = esVip ? descuentoBase + this.BONUS_VIP : descuentoBase;
    return Math.min(descuentoTotal, this.DESCUENTO_MAXIMO);
  }

  calcularPrecioFinal(monto: number, esVip: boolean): number {
    const descuento = this.calcularDescuento(monto, esVip);
    return Math.round((monto * (1 - descuento)) * 100) / 100;
  }
}
```

**Ejecuta los tests**: `npx vitest run` — Todos pasan (GREEN ✅).

### Paso 3: REFACTOR — Mejorar sin romper tests

Los tests te protegen mientras refactorizas. Podrías, por ejemplo, extraer la lógica de tramos a una configuración inyectable, y mientras los tests sigan pasando, sabes que el comportamiento no ha cambiado.

---

## 13.4 Ejercicio Práctico: Testing de Servicios HTTP

### Objetivo
Testear un servicio que consume una API REST, verificando las peticiones HTTP y simulando respuestas del servidor.

### El servicio a testear

```typescript
// producto-api.service.ts
import { Injectable, inject } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { Observable } from "rxjs";

export interface Producto {
  id: string;
  nombre: string;
  precio: number;
}

@Injectable({ providedIn: "root" })
export class ProductoApiService {
  private readonly http = inject(HttpClient);
  private readonly apiUrl = "/api/productos";

  listar(): Observable<Producto[]> {
    return this.http.get<Producto[]>(this.apiUrl);
  }

  obtenerPorId(id: string): Observable<Producto> {
    return this.http.get<Producto>(`${this.apiUrl}/${id}`);
  }

  crear(producto: Omit<Producto, "id">): Observable<Producto> {
    return this.http.post<Producto>(this.apiUrl, producto);
  }
}
```

### El test completo

```typescript
// producto-api.service.spec.ts
import { TestBed } from "@angular/core/testing";
import { provideHttpClient } from "@angular/common/http";
import { provideHttpClientTesting, HttpTestingController } from "@angular/common/http/testing";
import { ProductoApiService, Producto } from "./producto-api.service";
import { describe, beforeEach, afterEach, it, expect } from "vitest";

describe("ProductoApiService", () => {
  let service: ProductoApiService;
  let httpMock: HttpTestingController;

  // Datos de prueba reutilizables
  const productosMock: Producto[] = [
    { id: "1", nombre: "Laptop Pro", precio: 1200 },
    { id: "2", nombre: "Monitor 4K", precio: 450 }
  ];

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting() // Intercepta todas las peticiones HTTP
      ]
    });

    service = TestBed.inject(ProductoApiService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    // Verificar que no quedan peticiones pendientes
    httpMock.verify();
  });

  describe("listar()", () => {
    it("debe hacer GET a /api/productos y retornar la lista", () => {
      // Act: suscribirse al observable
      let resultado: Producto[] | undefined;
      service.listar().subscribe(data => resultado = data);

      // Assert: verificar la petición y simular la respuesta
      const req = httpMock.expectOne("/api/productos");
      expect(req.request.method).toBe("GET");
      req.flush(productosMock); // Simular respuesta del servidor

      expect(resultado).toEqual(productosMock);
      expect(resultado!.length).toBe(2);
    });

    it("debe manejar un error del servidor", () => {
      let errorCapturado: any;
      service.listar().subscribe({
        error: (err) => errorCapturado = err
      });

      const req = httpMock.expectOne("/api/productos");
      req.flush("Error interno", { status: 500, statusText: "Internal Server Error" });

      expect(errorCapturado.status).toBe(500);
    });
  });

  describe("obtenerPorId()", () => {
    it("debe hacer GET a /api/productos/:id", () => {
      let resultado: Producto | undefined;
      service.obtenerPorId("1").subscribe(data => resultado = data);

      const req = httpMock.expectOne("/api/productos/1");
      expect(req.request.method).toBe("GET");
      req.flush(productosMock[0]);

      expect(resultado).toEqual(productosMock[0]);
    });
  });

  describe("crear()", () => {
    it("debe hacer POST con el body correcto", () => {
      const nuevoProducto = { nombre: "Teclado RGB", precio: 150 };
      const respuestaEsperada: Producto = { id: "3", ...nuevoProducto };

      let resultado: Producto | undefined;
      service.crear(nuevoProducto).subscribe(data => resultado = data);

      const req = httpMock.expectOne("/api/productos");
      expect(req.request.method).toBe("POST");
      expect(req.request.body).toEqual(nuevoProducto);
      req.flush(respuestaEsperada);

      expect(resultado!.id).toBe("3");
    });
  });
});
```

### ✅ Verificación
- [ ] Todos los tests pasan con `npx vitest run`
- [ ] `httpMock.verify()` confirma que no hay peticiones pendientes

---

## 13.5 Ejercicio Práctico: Testing de Componentes Standalone

### Objetivo
Testear un componente presentacional que recibe inputs, emite outputs y renderiza contenido condicional.

### El test del componente

```typescript
// tarjeta-producto.component.spec.ts
import { ComponentFixture, TestBed } from "@angular/core/testing";
import { TarjetaProductoComponent } from "./tarjeta-producto.component";
import { describe, beforeEach, it, expect, vi } from "vitest";

describe("TarjetaProductoComponent", () => {
  let component: TarjetaProductoComponent;
  let fixture: ComponentFixture<TarjetaProductoComponent>;

  const productoMock = {
    id: "1", nombre: "Laptop Test", descripcion: "Desc test",
    precio: 999, categoria: "laptops" as const, imagenUrl: "",
    stock: 5, destacado: true
  };

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [TarjetaProductoComponent] // Standalone: se importa, no se declara
    }).compileComponents();

    fixture = TestBed.createComponent(TarjetaProductoComponent);
    component = fixture.componentInstance;
    
    // Establecer el input requerido
    fixture.componentRef.setInput("producto", productoMock);
    fixture.detectChanges();
  });

  it("debe renderizar el nombre del producto", () => {
    const nombre = fixture.nativeElement.querySelector("h3");
    expect(nombre.textContent).toContain("Laptop Test");
  });

  it("debe mostrar el badge de destacado para productos destacados", () => {
    const badge = fixture.nativeElement.querySelector("[class*='bg-amber']");
    expect(badge).toBeTruthy();
    expect(badge.textContent).toContain("Destacado");
  });

  it("debe mostrar el stock bajo con mensaje de advertencia", () => {
    const stockText = fixture.nativeElement.textContent;
    expect(stockText).toContain("¡Solo quedan 5!");
  });

  it("debe emitir el evento 'comprar' al hacer clic en el botón", () => {
    // Espiar el output
    const emitSpy = vi.fn();
    component.comprar.subscribe(emitSpy);

    const boton = fixture.nativeElement.querySelector("button");
    boton.click();

    expect(emitSpy).toHaveBeenCalledWith(productoMock);
  });

  it("debe deshabilitar el botón cuando el stock es 0", () => {
    fixture.componentRef.setInput("producto", { ...productoMock, stock: 0 });
    fixture.detectChanges();

    const boton: HTMLButtonElement = fixture.nativeElement.querySelector("button");
    expect(boton.disabled).toBe(true);
    expect(boton.textContent).toContain("Agotado");
  });
});
```

---

## 13.6 Testing de Signals y Computeds

Los Signals son síncronos, lo que hace que testearlos sea extremadamente directo:

```typescript
import { signal, computed } from "@angular/core";
import { describe, it, expect } from "vitest";

describe("Signals reactivos", () => {
  it("computed debe recalcularse cuando cambia una dependencia", () => {
    const precio = signal(100);
    const cantidad = signal(3);
    const total = computed(() => precio() * cantidad());

    expect(total()).toBe(300);

    precio.set(200);
    expect(total()).toBe(600); // Se recalcula síncronamente

    cantidad.set(5);
    expect(total()).toBe(1000);
  });

  it("computed con equal personalizado no debe cambiar si el valor es equivalente", () => {
    const datos = signal({ x: 1, y: 2 });
    let computeCount = 0;
    
    const suma = computed(() => {
      computeCount++;
      return datos().x + datos().y;
    });

    expect(suma()).toBe(3);
    expect(computeCount).toBe(1);

    // Mismo valor con nueva referencia: computed se recalcula pero el resultado es igual
    datos.set({ x: 1, y: 2 });
    expect(suma()).toBe(3);
    // computeCount será 2 porque la referencia cambió
  });
});
```

---

## 13.7 Testing Asíncrono: `fakeAsync`, `tick` y `flush`

Para testear código que depende de timers, debounces o delays:

```typescript
import { fakeAsync, tick, flush } from "@angular/core/testing";
import { describe, it, expect } from "vitest";

describe("Operaciones asíncronas", () => {
  it("debe ejecutar código después de un delay con fakeAsync", fakeAsync(() => {
    let resultado = "";
    
    setTimeout(() => resultado = "completado", 1000);
    
    expect(resultado).toBe(""); // Aún no ha pasado el tiempo
    
    tick(500);
    expect(resultado).toBe(""); // Solo han pasado 500ms
    
    tick(500);
    expect(resultado).toBe("completado"); // Ahora sí
  }));

  it("flush() avanza todos los timers pendientes", fakeAsync(() => {
    let contador = 0;
    
    setTimeout(() => contador++, 100);
    setTimeout(() => contador++, 200);
    setTimeout(() => contador++, 300);
    
    flush(); // Avanza TODOS los timers pendientes
    
    expect(contador).toBe(3);
  }));
});
```

---

## 13.8 Testing de Guards, Resolvers y Pipes

### Testing de un Guard funcional

```typescript
// auth.guard.spec.ts
import { TestBed } from "@angular/core/testing";
import { Router } from "@angular/router";
import { authGuard } from "./auth.guard";
import { AuthService } from "../services/auth.service";
import { signal } from "@angular/core";
import { describe, beforeEach, it, expect, vi } from "vitest";

describe("authGuard", () => {
  let authServiceMock: { estaAutenticado: ReturnType<typeof signal> };
  let routerMock: { navigate: ReturnType<typeof vi.fn> };

  beforeEach(() => {
    authServiceMock = { estaAutenticado: signal(false) };
    routerMock = { navigate: vi.fn() };

    TestBed.configureTestingModule({
      providers: [
        { provide: AuthService, useValue: authServiceMock },
        { provide: Router, useValue: routerMock }
      ]
    });
  });

  it("debe denegar acceso y redirigir a /login si no está autenticado", () => {
    const resultado = TestBed.runInInjectionContext(() => authGuard({} as any, {} as any));
    
    expect(resultado).toBe(false);
    expect(routerMock.navigate).toHaveBeenCalledWith(["/login"]);
  });

  it("debe permitir acceso si está autenticado", () => {
    authServiceMock.estaAutenticado.set(true);
    const resultado = TestBed.runInInjectionContext(() => authGuard({} as any, {} as any));
    
    expect(resultado).toBe(true);
  });
});
```

### Testing de un Pipe puro

```typescript
// moneda-formato.pipe.spec.ts
import { MonedaFormatoPipe } from "./moneda-formato.pipe";
import { describe, it, expect } from "vitest";

describe("MonedaFormatoPipe", () => {
  const pipe = new MonedaFormatoPipe();

  it("debe formatear un número como moneda EUR", () => {
    expect(pipe.transform(1234.5, "EUR")).toBe("1.234,50 €");
  });

  it("debe manejar valores nulos retornando cadena vacía", () => {
    expect(pipe.transform(null, "EUR")).toBe("");
  });

  it("debe formatear cero correctamente", () => {
    expect(pipe.transform(0, "EUR")).toBe("0,00 €");
  });
});
```

---

## 13.9 Cobertura de Código

Configura umbrales mínimos de cobertura para tu CI/CD:

```typescript
// vite.config.ts (sección test)
test: {
  coverage: {
    provider: "v8",
    reporter: ["text", "html", "lcov"],
    thresholds: {
      branches: 70,
      functions: 80,
      lines: 80,
      statements: 80
    },
    include: ["src/app/**/*.ts"],
    exclude: ["src/app/**/*.spec.ts", "src/app/**/*.module.ts"]
  }
}
```

```bash
npx vitest run --coverage
```

---

## 13.10 Ejercicio Práctico: Tests E2E con Playwright

### Paso 1: Instalar Playwright

```bash
npm init playwright@latest
# Seleccionar TypeScript, carpeta 'e2e/', instalar browsers
```

### Paso 2: Crear un Page Object

```typescript
// e2e/pages/catalogo.page.ts
import { Page, Locator } from "@playwright/test";

export class CatalogoPage {
  readonly page: Page;
  readonly titulo: Locator;
  readonly tarjetasProducto: Locator;
  readonly inputBusqueda: Locator;
  readonly botonesCategoria: Locator;
  readonly botonAgregar: Locator;

  constructor(page: Page) {
    this.page = page;
    this.titulo = page.locator("h1");
    this.tarjetasProducto = page.locator("app-tarjeta-producto");
    this.inputBusqueda = page.locator("#busqueda");
    this.botonesCategoria = page.locator("[class*='rounded-full']");
    this.botonAgregar = page.locator("button:has-text('Agregar')");
  }

  async navegar(): Promise<void> {
    await this.page.goto("/catalogo");
  }

  async buscar(termino: string): Promise<void> {
    await this.inputBusqueda.fill(termino);
  }

  async filtrarPorCategoria(categoria: string): Promise<void> {
    await this.page.locator(`button:has-text('${categoria}')`).click();
  }

  async agregarPrimerProducto(): Promise<void> {
    await this.botonAgregar.first().click();
  }

  async contarProductos(): Promise<number> {
    return this.tarjetasProducto.count();
  }
}
```

### Paso 3: Escribir el test E2E

```typescript
// e2e/catalogo.spec.ts
import { test, expect } from "@playwright/test";
import { CatalogoPage } from "./pages/catalogo.page";

test.describe("Catálogo de productos", () => {
  let catalogo: CatalogoPage;

  test.beforeEach(async ({ page }) => {
    catalogo = new CatalogoPage(page);
    await catalogo.navegar();
  });

  test("debe mostrar el título y los productos", async () => {
    await expect(catalogo.titulo).toContainText("Catálogo");
    const total = await catalogo.contarProductos();
    expect(total).toBeGreaterThan(0);
  });

  test("debe filtrar productos por búsqueda", async () => {
    const totalInicial = await catalogo.contarProductos();
    await catalogo.buscar("Laptop");
    const totalFiltrado = await catalogo.contarProductos();
    expect(totalFiltrado).toBeLessThan(totalInicial);
    expect(totalFiltrado).toBeGreaterThan(0);
  });

  test("debe filtrar por categoría", async () => {
    await catalogo.filtrarPorCategoria("monitores");
    const total = await catalogo.contarProductos();
    expect(total).toBeGreaterThan(0);
    // Verificar que todos los productos visibles son monitores
    const textos = await catalogo.tarjetasProducto.allTextContents();
    textos.forEach(texto => {
      expect(texto.toLowerCase()).not.toContain("laptop");
    });
  });
});
```

### Paso 4: Ejecutar

```bash
npx playwright test
npx playwright show-report  # Abrir el reporte visual
```

---

## Resumen del Capítulo

* La **Pirámide de Testing** establece la distribución óptima: muchos tests unitarios rápidos en la base, pocos tests E2E lentos en la cima.
* **Vitest** reemplaza a Karma/Jasmine con un runner ultrarrápido basado en Vite, compatible con la sintaxis de Jasmine/Jest.
* **TDD (Red → Green → Refactor)** produce código más robusto: primero defines el comportamiento esperado con tests, luego escribes el código mínimo para cumplirlo.
* Los tests de servicios HTTP usan `HttpTestingController` para interceptar peticiones, verificar métodos/URLs y simular respuestas del servidor.
* Los componentes standalone se testean importándolos directamente en `TestBed`, usando `setInput()` para inputs y espiando outputs con `vi.fn()`.
* Los **Signals y Computeds son síncronos**, lo que hace que testearlos sea extremadamente directo: set → assert.
* **Guards funcionales** se testean con `TestBed.runInInjectionContext()` y mocks de dependencias.
* **Playwright** con Page Objects proporciona tests E2E mantenibles, paralelos y con reportes visuales.
* La **cobertura de código** debe configurarse con umbrales mínimos (80% lines/functions) y ejecutarse en CI/CD.
