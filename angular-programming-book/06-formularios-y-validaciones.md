# Capítulo 6: Formularios y Validaciones

La captura, gestión y validación de datos ingresados por el usuario es uno de los requisitos más críticos y desafiantes en las aplicaciones web empresariales. Ya sea en un sencillo panel de acceso o en un wizard complejo de múltiples pasos para el sector financiero, contar con una gestión de estado del formulario precisa, segura y reactiva es fundamental para garantizar la integridad de los datos.

Angular aborda este desafío ofreciendo dos enfoques potentes de desarrollo: los **Formularios Basados en Plantillas (Template-driven Forms)** y los **Formularios Reactivos (Reactive Forms)**.

En este capítulo, nos sumergiremos en la arquitectura avanzada de los **Formularios Reactivos con tipado estricto (Strictly Typed Forms)**. Aprenderemos a modelar estructuras de datos dinámicas con `FormArray`, a crear validadores síncronos y asíncronos personalizados de nivel profesional, y a integrar de forma fluida el flujo reactivo de los formularios con **Signals** para pintar interfaces dinámicas impecables y libres de deuda técnica.

---

## 6.1 Template-driven Forms vs. Reactive Forms

Angular proporciona dos metodologías bien diferenciadas para construir formularios en tu aplicación. Comprender cuándo aplicar cada una es clave para estructurar correctamente tu arquitectura:

| Característica | Template-driven Forms | Reactive Forms (Recomendado) |
|---|---|---|
| **Paradigma** | Declarativo (basado en directivas de plantilla) | Imperativo (basado en un modelo en TypeScript) |
| **Fuente de Verdad** | La plantilla HTML | El código TypeScript |
| **Flujo de Datos** | Asíncrono bidireccional (`ngModel`) | Síncrono unidireccional y reactivo |
| **Tipado Estricto** | Limitado o nulo en compilación | **Totalmente tipado en tiempo de compilación** |
| **Facilidad de Testeo** | Complejo (requiere renderizar el DOM) | **Extremadamente sencillo e independiente** |
| **Escalabilidad** | Adecuado para flujos muy simples o prototipos | **Diseñado para aplicaciones complejas y corporativas** |

### ¿Por qué elegir Reactive Forms en producción?
Los Formularios Reactivos se construyen creando un **modelo de objetos inmutable** en tu clase TypeScript. Esto significa que tienes un control absoluto sobre el valor, el estado de validez (`valid`, `invalid`), el estado de interacción (`pristine`, `dirty`, `touched`, `untouched`) y los cambios en tiempo de real mediante flujos reactivos de RxJS, permitiendo validaciones cruzadas complejas con total facilidad.

---

## 6.2 Formularios Reactivos con Tipado Estricto (Strictly Typed Forms)

A partir de Angular v14, la API de formularios reactivos fue reescrita para dotarla de un **tipado estricto**. Esto significa que TypeScript conoce exactamente la estructura del formulario, previniendo errores comunes en tiempo de compilación (como consultar propiedades inexistentes o asignar tipos incorrectos).

### Conceptos Fundamentales:
* **`FormControl`**: Administra el valor, validez e interacción de un único elemento de entrada individual (un campo de texto, checkbox, etc.).
* **`FormGroup`**: Agrupa una colección de instancias de `FormControl`, `FormGroup` o `FormArray`, facilitando la validación conjunta de un bloque.
* **`FormBuilder` / `NonNullableFormBuilder`**: Servicios de andamiaje que simplifican la declaración y creación de formularios complejos.

### Creación de un Formulario de Registro de Producción

Para crear un formulario reactivo standalone, debemos importar `ReactiveFormsModule` en nuestro componente standalone, o inyectar el servicio `FormBuilder`.

```typescript
import { Component, inject } from "@angular/core";
import { 
  NonNullableFormBuilder, 
  ReactiveFormsModule, 
  Validators 
} from "@angular/forms";

@Component({
  selector: "app-registro",
  standalone: true,
  imports: [ReactiveFormsModule], // Importación obligatoria
  templateUrl: "./registro.component.html"
})
export class RegistroComponent {
  // 1. Inyectamos la versión moderna de FormBuilder no-nula (NonNullableFormBuilder)
  private readonly fb = inject(NonNullableFormBuilder);

  // 2. Definimos la estructura del formulario fuertemente tipada
  readonly registroForm = this.fb.group({
    nombre: ["", [Validators.required, Validators.minLength(3)]],
    email: ["", [Validators.required, Validators.email]],
    password: ["", [Validators.required, Validators.minLength(8)]],
    terminos: [false, Validators.requiredTrue]
  });

  enviar(): void {
    if (this.registroForm.invalid) {
      this.registroForm.markAllAsTouched(); // Dispara la visualización de errores visuales
      return;
    }

    // El valor del formulario está fuertemente tipado de forma implícita
    const datosEnvio = this.registroForm.getRawValue();
    console.log("Datos de envío validados y tipados:", datosEnvio);
  }
}
```

#### ¿Qué es `NonNullableFormBuilder` (o `fb.nonNullable`)?
Por defecto en Angular clásico, cuando llamas a `form.reset()`, los valores de los controles se reseteaban a `null`. Esto obligaba a que todos los tipos de tus formularios fueran `T | null` (ej. `string | null`).
Al utilizar **`NonNullableFormBuilder`**, le indicamos a Angular que al resetear el formulario, los controles deben regresar a su **valor por defecto inicial** (por ejemplo, string vacío `""` o booleano `false`), garantizando tipos puramente limpios en TypeScript (`string`, `boolean`, etc.).

---

## 6.3 Manejo de Colecciones Dinámicas con `FormArray`

Un **`FormArray`** es una clase especializada que gestiona una lista indexada dinámicamente de instancias de `FormControl`, `FormGroup` o `FormArray`. Es la herramienta idónea cuando necesitas que el usuario agregue dinámicamente múltiples registros idénticos (por ejemplo, múltiples números de teléfono, direcciones de envío o ítems de una factura).

Veamos un ejemplo avanzado para agregar tecnologías de especialización en un formulario de postulación:

```typescript
import { Component, inject } from "@angular/core";
import { NonNullableFormBuilder, ReactiveFormsModule, FormArray, Validators } from "@angular/forms";

@Component({
  selector: "app-perfil-profesional",
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="perfilForm" (ngSubmit)="guardar()" class="p-6 max-w-md bg-white rounded shadow">
      <h2 class="text-xl font-bold mb-4">Postulación Laboral</h2>

      <div class="mb-4">
        <label class="block font-semibold mb-1">Nombre Completo</label>
        <input formControlName="nombre" class="w-full border p-2 rounded">
      </div>

      <div class="mb-4">
        <div class="flex justify-between items-center mb-2">
          <label class="font-semibold">Tecnologías Dominadas</label>
          <button type="button" (click)="agregarTecnologia()" class="text-sm bg-blue-600 text-white px-2 py-1 rounded">
            + Agregar
          </button>
        </div>

        <!-- Renderizado dinámico del FormArray utilizando el control flow @for -->
        <div formArrayName="tecnologias" class="space-y-2">
          @for (control of tecnologias.controls; track $index) {
            <div class="flex gap-2">
              <input [formControlName]="$index" placeholder="Ej. Angular, NestJS" class="w-full border p-2 rounded">
              <button type="button" (click)="removerTecnologia($index)" class="bg-red-500 text-white px-3 rounded">
                X
              </button>
            </div>
          }
        </div>
      </div>

      <button type="submit" class="w-full bg-green-600 text-white py-2 rounded">Guardar Perfil</button>
    </form>
  `
})
export class PerfilProfesionalComponent {
  private readonly fb = inject(NonNullableFormBuilder);

  readonly perfilForm = this.fb.group({
    nombre: ["", Validators.required],
    // Declaración del FormArray vacío de strings
    tecnologias: this.fb.array<string>([])
  });

  // Getter de conveniencia tipado para acceder al FormArray
  get tecnologias(): FormArray {
    return this.perfilForm.get("tecnologias") as FormArray;
  }

  agregarTecnologia(): void {
    // Agregamos un nuevo FormControl al arreglo dinámico
    this.tecnologias.push(this.fb.control("", Validators.required));
  }

  removerTecnologia(index: number): void {
    this.tecnologias.removeAt(index);
  }

  guardar(): void {
    if (this.perfilForm.invalid) return;
    console.log("Perfil con Tecnologías:", this.perfilForm.value);
  }
}
```

---

## 6.4 Validaciones Síncronas y Asíncronas Avanzadas

Angular permite validar los datos utilizando **Validadores**. Un validador es simplemente una función pura que recibe un control, analiza su valor y retorna un objeto de error si el valor no cumple con la regla, o `null` si el valor es válido.

### 1. Validadores Síncronos Personalizados

Construiremos un validador personalizado que verifique si una contraseña contiene al menos un carácter numérico y uno especial para cumplir políticas estrictas de seguridad corporativa.

```typescript
import { AbstractControl, ValidationErrors, ValidatorFn } from "@angular/forms";

export function passwordSeguroValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const valor = control.value || "";
    
    // Expresión regular que verifica existencia de número y carácter especial
    const tieneNumero = /\d/.test(valor);
    const tieneCaracterEspecial = /[!@#$%^&*(),.?":{}|<>]/.test(valor);

    if (!valor) return null; // Si no hay valor, la validación se delega al validador 'required'

    const esValido = tieneNumero && tieneCaracterEspecial;

    // Si es inválido, retornamos un mapa de errores. Si es válido, retornamos null.
    return !esValido 
      ? { passwordDebil: { tieneNumero, tieneCaracterEspecial } } 
      : null;
  };
}
```

#### Uso en el formulario:
```typescript
readonly registroForm = this.fb.group({
  password: ["", [Validators.required, passwordSeguroValidator()]]
});
```

---

### 2. Validadores Asíncronos Personalizados (Llamadas a Servidores)

Un validador asíncrono se ejecuta cuando las validaciones síncronas han pasado de manera exitosa. Debe retornar un **Observable** o una **Promesa** que emita un objeto de errores o `null` una vez finalizada la tarea asíncrona.

Es la herramienta perfecta para validar si un nombre de usuario o dirección de correo ya existe registrado en la base de datos de la empresa.

```typescript
import { Injectable, inject } from "@angular/core";
import { AbstractControl, AsyncValidator, ValidationErrors } from "@angular/forms";
import { Observable, of } from "rxjs";
import { delay, map, catchError } from "rxjs/operators";
import { HttpClient } from "@angular/common/http";

@Injectable({
  providedIn: "root"
})
export class EmailUnicoValidator implements AsyncValidator {
  private readonly http = inject(HttpClient);

  validate(control: AbstractControl): Observable<ValidationErrors | null> {
    const email = control.value;
    
    if (!email) return of(null);

    // Consultamos de forma asíncrona a nuestra API REST corporativa
    return this.http.get<{ disponible: boolean }>(`/api/usuarios/verificar-email?email=${email}`).pipe(
      delay(500), // Simula latencia de red para no sobrecargar el servidor
      map((res) => (res.disponible ? null : { emailTomado: true })),
      catchError(() => of(null)) // Si la API falla, dejamos pasar la validación por seguridad
    );
  }
}
```

#### Uso en el formulario:
Los validadores asíncronos se proveen como el **tercer argumento** de la declaración del control:

```typescript
readonly registroForm = this.fb.group({
  email: ["", {
    validators: [Validators.required, Validators.email],
    asyncValidators: [inject(EmailUnicoValidator).validate.bind(inject(EmailUnicoValidator))],
    updateOn: "blur" // Ejecuta la validación asíncrona únicamente cuando el usuario sale del input (blur)
  }]
});
```

---

## 6.5 Reactividad en Formularios con Signals y RxJS

El estado de los formularios puede acoplarse con la reactividad nativa de Signals para pintar interfaces de usuario reactivas e interactivas complejas con una limpieza sin precedentes.

Podemos convertir los flujos asíncronos del formulario a Signals utilizando `toSignal()` para reaccionar inmediatamente a los cambios:

```typescript
import { Component, inject } from "@angular/core";
import { NonNullableFormBuilder, ReactiveFormsModule } from "@angular/forms";
import { toSignal } from "@angular/core/rxjs-interop";
import { computed } from "@angular/core";

@Component({
  selector: "app-calculador-interactivo",
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="cotizadorForm" class="p-6 bg-slate-50 rounded border">
      <h3 class="font-bold mb-4">Cotizador de Plan de Hosting</h3>
      
      <div class="mb-4">
        <label class="block mb-1">Capacidad GB</label>
        <input type="number" formControlName="capacidad" class="w-full p-2 border rounded">
      </div>

      <div class="mb-4">
        <label class="block mb-1">Periodo Anual</label>
        <input type="checkbox" formControlName="esAnual">
      </div>

      <div class="p-4 bg-blue-100 rounded text-blue-900">
        <p class="font-semibold text-lg">Total Estimado: {{ precioTotal() }} € / mes</p>
        <small>Valores actualizados dinámicamente con Signals</small>
      </div>
    </form>
  `
})
export class CalculadorInteractivoComponent {
  private readonly fb = inject(NonNullableFormBuilder);

  readonly cotizadorForm = this.fb.group({
    capacidad: [10],
    esAnual: [false]
  });

  // 1. Convertimos los flujos de cambio de valor del formulario a Signals síncronos
  private readonly _formValue = toSignal(this.cotizadorForm.valueChanges, {
    initialValue: this.cotizadorForm.value
  });

  // 2. Usamos computed para computar dinámicamente el precio total del plan
  readonly precioTotal = computed(() => {
    const datos = this._formValue();
    const base = (datos.capacidad || 0) * 0.5; // 0.5€ por GB
    const descuento = datos.esAnual ? 0.8 : 1.0; // 20% descuento por plan anual
    
    return base * descuento;
  });
}
```

---

## Resumen del Capítulo

* Los **Formularios Reactivos** proporcionan un modelo de objetos mutable estructurado en TypeScript, ofreciendo tipado estricto, flujos reactivos de datos y facilidad de testeo unitario.
* **`NonNullableFormBuilder`** previene el tipado de controles con `null` al retornar el control a su valor por defecto al invocarse un reset.
* **`FormArray`** gestiona dinámicamente colecciones de formularios de forma indexada e interactiva, ideal para andamios dinámicos de producción.
* Los **Validadores Síncronos Personalizados** son funciones puras capaces de resolver lógicas de negocio locales complejas.
* Los **Validadores Asíncronos Personalizados** resuelven la validación distribuida contra bases de datos remotas mediante flujos asíncronos enlazados al evento blur para evitar sobrecarga del servidor.
* La **interoperabilidad con Signals** (`toSignal`) permite calcular estados derivados en tiempo real basados en los datos del formulario de manera rápida, limpia y declarativa.

En el próximo capítulo, aprenderemos a configurar la navegación y a proteger el flujo de pantallas en nuestra aplicación moderna dominando el **Enrutamiento y la Carga Perezosa (Lazy Loading)**.

---

← [Capítulo anterior](05-servicios-e-inyeccion-de-dependencias.md) | [Inicio](README.md) | [Capítulo siguiente →](07-enrutamiento-y-navegacion.md)
