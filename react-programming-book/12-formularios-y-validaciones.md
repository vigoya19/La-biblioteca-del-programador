# Capítulo 12: Formularios Avanzados y Validaciones

> "Un formulario en producción no es solo un conjunto de inputs de texto; es la primera línea de defensa de la integridad de los datos de tu sistema backend."

El manejo de formularios interactivos ha sido históricamente una de las áreas más tediosas del desarrollo frontend. En React, la aproximación básica mediante **componentes controlados** (donde guardas cada carácter tecleado en un estado local con `useState`) provoca serios cuellos de botella de rendimiento por re-renderizados masivos en cada pulsación de tecla, y ensucia el código con lógica manual redundante.

En este capítulo, analizaremos la diferencia entre componentes controlados y no controlados, y aprenderemos a construir formularios profesionales de alto rendimiento utilizando **React Hook Form** y validaciones estrictas de esquemas empresariales con **Zod** en TypeScript.

---

## 12.1 Componentes Controlados vs. No Controlados

En React existen dos formas complementarias de interactuar con los campos de entrada de un formulario:

1.  **Componentes Controlados**: React es la "fuente única de verdad". El valor de cada input está enlazado a una variable de estado (`value={texto}`) y cada cambio invoca a una callback (`onChange={e => setTexto(e.target.value)}`).
    *   *Desventaja:* Cada carácter ingresado por el usuario fuerza al componente y a todos sus hijos a renderizarse nuevamente. En formularios con 20 inputs, esto genera un retardo visible al escribir (*typing lag*).
2.  **Componentes No Controlados**: El DOM del navegador retiene de forma nativa los valores escritos en los inputs. React solo los extrae cuando el usuario decide enviar el formulario utilizando referencias (`useRef`) o APIs nativas del navegador.
    *   *Ventaja:* Cero re-renderizados inútiles al escribir. Rendimiento óptimo y fluido.

---

## 12.2 React Hook Form y Zod: El Puesto de Control Perfecto

Para combinar la ligereza de los componentes no controlados con la reactividad y las validaciones de React, la industria ha adoptado **React Hook Form** junto a **Zod** (una librería de declaración y validación de esquemas con inferencia de tipos estática en TypeScript).

> [!NOTE]
> ### 🛂 El Puesto de Control de Aduana con Escáner Automático
> 
> Imagina que eres el encargado de seguridad del puerto de carga de un país (tu aplicación web):
> 
> - Un formulario básico con `useState` (**Componente Controlado**) es equivalente a un **Oficial de aduana lento y burocrático**: Detiene el camión en la puerta, abre cada caja una por una, saca cada artículo, lo pesa, lo anota en un papel con lápiz, y repite esto cada segundo mientras el conductor intenta explicarle qué lleva. El tráfico se congestiona por kilómetros (**retardos severos en la renderización**).
> - **React Hook Form** es equivalente a instalar un **Escáner Láser de Código de Barras Automático** en la puerta: El camión de carga avanza a toda velocidad sin detenerse. El escáner registra y procesa el cargamento completo de forma silenciosa en memoria y solo emite un pitido cuando el conductor pulsa el botón de entrada (**uncontrolled inputs procesados al hacer submit**).
> - **Zod** es equivalente al **Inspector de Aduana con el Reglamento Rígido**: Verifica el reporte digital emitido por el escáner en milisegundos contra el manual de importación oficial: *"¿Vino con el peso correcto?, ¿el código de barras es válido?, ¿el email del conductor es real?"*. Si algo no cumple las normas, emite un reporte de error al instante, protegiendo la frontera del país (**tu base de datos**) contra cargamentos ilegales.

---

## 12.3 Ejemplo Profesional: Registro de Usuario con Tipado Estricto

Implementemos un formulario de registro profesional con validaciones complejas (como confirmación de contraseña) y tipado automático inferido en TypeScript:

### `RegistroForm.tsx`
```typescript
import React from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// 1. Declarar el esquema de validación de negocio usando Zod
const esquemaRegistro = z.object({
  nombre: z.string()
    .min(3, 'El nombre debe tener al menos 3 caracteres')
    .max(50, 'El nombre no puede superar los 50 caracteres'),
  email: z.string()
    .email('Introduce una dirección de correo electrónico válida'),
  password: z.string()
    .min(8, 'La contraseña debe tener al menos 8 caracteres')
    .regex(/[A-Z]/, 'Debe incluir al menos una letra mayúscula')
    .regex(/[0-9]/, 'Debe incluir al menos un número'),
  confirmarPassword: z.string()
}).refine((data) => data.password === data.confirmarPassword, {
  message: 'Las contraseñas no coinciden',
  path: ['confirmarPassword'], // Indica en qué campo debe inyectarse el mensaje de error
});

// 2. Inferir estáticamente el tipo de TypeScript desde el esquema de Zod
type FormularioRegistroData = z.infer<typeof esquemaRegistro>;

export function RegistroForm(): React.JSX.Element {
  // 3. Inicializar React Hook Form integrando el resolver de Zod
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    reset
  } = useForm<FormularioRegistroData>({
    resolver: zodResolver(esquemaRegistro)
  });

  // 4. Callback de envío seguro y tipado
  const alEnviar = async (data: FormularioRegistroData) => {
    // Simulación de petición de red
    await new Promise(resolve => setTimeout(resolve, 1500));
    console.log('Datos enviados con éxito y validados por Zod:', data);
    alert('¡Registro completado con éxito!');
    reset(); // Vaciar el formulario
  };

  return (
    <form 
      onSubmit={handleSubmit(alEnviar)}
      style={{ maxWidth: '400px', margin: '2rem auto', display: 'flex', flexDirection: 'column', gap: '1rem' }}
    >
      <h2>Registro de Cuenta</h2>

      <div>
        <label htmlFor="nombre" style={{ display: 'block', fontWeight: 'bold' }}>Nombre Completo:</label>
        {/* El método register inyecta las referencias y listeners de forma invisible */}
        <input id="nombre" type="text" {...register('nombre')} style={{ width: '100%', padding: '0.5rem' }} />
        {errors.nombre && <span style={{ color: 'red', fontSize: '0.85rem' }}>{errors.nombre.message}</span>}
      </div>

      <div>
        <label htmlFor="email" style={{ display: 'block', fontWeight: 'bold' }}>Email corporativo:</label>
        <input id="email" type="email" {...register('email')} style={{ width: '100%', padding: '0.5rem' }} />
        {errors.email && <span style={{ color: 'red', fontSize: '0.85rem' }}>{errors.email.message}</span>}
      </div>

      <div>
        <label htmlFor="password" style={{ display: 'block', fontWeight: 'bold' }}>Contraseña:</label>
        <input id="password" type="password" {...register('password')} style={{ width: '100%', padding: '0.5rem' }} />
        {errors.password && <span style={{ color: 'red', fontSize: '0.85rem' }}>{errors.password.message}</span>}
      </div>

      <div>
        <label htmlFor="confirmarPassword" style={{ display: 'block', fontWeight: 'bold' }}>Confirmar Contraseña:</label>
        <input id="confirmarPassword" type="password" {...register('confirmarPassword')} style={{ width: '100%', padding: '0.5rem' }} />
        {errors.confirmarPassword && <span style={{ color: 'red', fontSize: '0.85rem' }}>{errors.confirmarPassword.message}</span>}
      </div>

      <button type="submit" disabled={isSubmitting} style={{ padding: '0.75rem', cursor: 'pointer' }}>
        {isSubmitting ? 'Registrando...' : 'Crear Cuenta'}
      </button>
    </form>
  );
}
```

---

## Resumen del Capítulo

*   Los **componentes controlados** enlazan el estado al DOM forzando renders continuos; los **no controlados** delegan en el navegador para lograr velocidad.
*   **React Hook Form** aprovecha el rendimiento de los componentes no controlados, abstrayendo refs y listeners mediante el método `register`.
*   **Zod** permite declarar esquemas de datos empresariales con validaciones complejas, y exporta tipados estáticos de forma automática a TypeScript.
*   La combinación de **React Hook Form + Zod** es el estándar industrial moderno en React para construir formularios ligeros, de alto rendimiento y 100% seguros contra tipos corruptos.

En el próximo capítulo, abordaremos la optimización matemática del renderizado, analizando cómo evitar repintados costosos mediante **React.memo**, cómo manejar listas dinámicas masivas con virtualización y cómo diagnosticar cuellos de botella reales usando las React Developer Tools.

---

[Capítulo anterior](11-react-router.md) | [Inicio](README.md) | [Capítulo siguiente →](13-optimizacion-y-rendimiento.md)
