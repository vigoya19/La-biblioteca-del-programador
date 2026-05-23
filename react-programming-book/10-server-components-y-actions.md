# Capítulo 10: Server Components y React Server Actions

> "La verdadera madurez de un framework web consiste en difuminar la frontera física entre el cliente y el servidor, convirtiendo el desarrollo fullstack en una experiencia integrada, segura y de altísimo rendimiento."

Durante la última década, las aplicaciones construidas con React dependieron mayoritariamente del modelo **Client-Side Rendering (CSR)**: el servidor entregaba un archivo HTML vacío y un bundle masivo de JavaScript que el navegador del usuario debía descargar, compilar, ejecutar y renderizar desde cero. Esto provocaba tiempos de carga inicial lentos y problemas severos de SEO en aplicaciones públicas.

En **React v19**, la arquitectura híbrida se ha consolidado definitivamente a través del estándar de **React Server Components (RSC)** y las mutaciones nativas de datos mediante **Server Actions**. En este capítulo, estudiaremos cómo opera este modelo fullstack, cuándo alternar entre componentes de cliente y de servidor, y cómo mutar datos en bases de datos directamente desde tus formularios en React.

---

## 10.1 La Arquitectura Híbrida: RSC y Client Components

En los frameworks modernos (como Next.js o Remix/React Router v7), **todos los componentes son React Server Components por defecto**. 

*   **React Server Components (RSC)**: Se ejecutan y renderizan **única y exclusivamente en el servidor**. Generan un formato de datos serializado ligero que se transmite al navegador. Tienen acceso directo a bases de datos, sistemas de archivos locales y claves API privadas, con un impacto de **0 bytes en el bundle de JavaScript del cliente**.
*   **Client Components**: Componentes tradicionales de React que se ejecutan en el navegador. Se activan explícitamente agregando la directiva de texto **`'use client'`** en la primera línea del archivo. Son los únicos autorizados a usar Hooks (`useState`, `useEffect`), declarar manejadores de eventos interactivos (`onClick`, `onChange`) o consumir APIs del navegador (como `window` o `document`).

```
  [ Servidor (Node.js/Edge) ]               [ Cliente (Navegador Web) ]
┌─────────────────────────────┐           ┌─────────────────────────────┐
│ - Server Components (RSC)   │           │ - Client Components         │
│ - Acceso Directo a DB/APIs  │           │   (Marcados con 'use client')│
│ - Renderizado a HTML/JSON   │ ───────►  │ - Interactividad (Hooks)    │
│ - 0 JS enviado al cliente   │ (Stream)  │ - Eventos del DOM (onClick) │
└─────────────────────────────┘           └─────────────────────────────┘
```

> [!NOTE]
> ### 🏭 La Analogía de las Paredes Prefabricadas vs. Los Ladrillos Sueltos
> 
> Imagina que estás construyendo una nueva casa en tu terreno (la pantalla del navegador del cliente):
> 
> - **Los React Server Components (RSC)** son equivalentes a encargar **Paredes Prefabricadas de Fábrica**: En lugar de enviar materiales sueltos, la fábrica del servidor construye las paredes completas de hormigón, las pinta y las entrega listas en un camión de transporte. Cuando llegan a tu terreno, los obreros simplemente las encajan al instante con grúas en milisegundos (**Streaming de HTML**). No hay desperdicio de espacio, ni herramientas pesadas que comprar, ni ruido en tu terreno (**0 bytes de bundle JavaScript en el cliente**).
> - **Los Client Components** son equivalentes a comprar **Ladrillos Sueltos y Sacos de Cemento**: Te envían un camión lleno de arena, ladrillos individuales, palas, y un manual de construcción de 500 páginas en inglés (**el bundle de JavaScript pesado**). El dueño del terreno (el procesador de la computadora del usuario) debe leer el manual desde cero, mezclar el cemento con agua, y apilar cada ladrillo a mano uno por uno para levantar la pared (**hidratación y ejecución en el navegador**). Toma tiempo, energía y retrasa el día de mudanza.

---

## 10.2 Cuándo usar cada componente (Tabla de Criterios)

| Criterio / Tarea | Server Component (RSC) | Client Component (`'use client'`) |
| :--- | :---: | :---: |
| Hacer fetch de datos desde APIs / Bases de Datos directamente | **Sí** | No (Requiere useEffect/use) |
| Mantener credenciales, tokens de API y llaves privadas seguras | **Sí** | No (Quedan expuestas en el JS) |
| Importar librerías de gran tamaño (ej: Markdown parsers, date formatting) | **Sí** | No (Inflaría el bundle del cliente) |
| Utilizar hooks de estado local o efectos (`useState`, `useEffect`) | No | **Sí** |
| Agregar eventos de interactividad física como `onClick` o `onSubmit` | No | **Sí** |
| Usar APIs nativas de navegación como `localStorage` o `window` | No | **Sí** |

---

## 10.3 React Server Actions: Mutación Directa en Servidor

Las **React Server Actions** permiten definir funciones asíncronas de servidor que el cliente puede invocar de forma transparente como si fuesen funciones locales normales. React se encarga por debajo de instrumentar la llamada HTTP POST asíncrona de forma segura.

Se declaran agregando la directiva **`'use server'`** al inicio del archivo o del cuerpo de la función.

### Ejemplo Profesional: Creación de un Post en base de datos usando Server Actions y Formulario Nativo

#### 1. [postActions.ts](file:///Users/andres/Documents/biblioteca/react-programming-book/src/actions/postActions.ts) (Lógica que corre estrictamente en Servidor)
```typescript
'use server';

// Esta función se ejecuta y compila ÚNICAMENTE en el servidor Node.js
export async function crearNuevoPost(formData: FormData): Promise<{ success: boolean; message: string }> {
  const titulo = formData.get('titulo') as string;
  const contenido = formData.get('contenido') as string;

  // Validaciones básicas de servidor
  if (!titulo || !contenido) {
    return { success: false, message: 'Todos los campos son estrictamente obligatorios.' };
  }

  try {
    console.log(`Guardando en Base de Datos de forma segura en Servidor: ${titulo}`);
    
    // Aquí puedes importar tu cliente Prisma, mongoose o fetch de base de datos directamente
    // await db.post.create({ data: { titulo, contenido } });
    
    // Simulación de retraso de red
    await new Promise(resolve => setTimeout(resolve, 800));

    return { success: true, message: 'Post guardado con éxito.' };
  } catch (error) {
    return { success: false, message: 'Error interno del servidor al procesar el post.' };
  }
}
```

#### 2. [FormularioPost.tsx](file:///Users/andres/Documents/biblioteca/react-programming-book/src/components/FormularioPost.tsx) (Componente de Cliente Interactivo)
```typescript
'use client';

import React, { useState } from 'react';
import { crearNuevoPost } from '../actions/postActions';

export function FormularioPost(): React.JSX.Element {
  const [mensaje, setMensaje] = useState<string | null>(null);
  const [pendiente, setPendiente] = useState<boolean>(false);

  // Acción del formulario que invoca a la Server Action de forma directa
  const handleAction = async (formData: FormData) => {
    setPendiente(true);
    setMensaje(null);
    
    // Llamar a la Server Action de forma asíncrona transparente
    const resultado = await crearNuevoPost(formData);
    
    setMensaje(resultado.message);
    setPendiente(false);
  };

  return (
    <form 
      action={handleAction} // React 19 permite pasar funciones asíncronas directamente al prop action
      style={{ display: 'flex', flexDirection: 'column', gap: '1rem', maxWidth: '400px', margin: '2rem auto' }}
    >
      <h3>Crear Publicación</h3>
      
      <div>
        <label htmlFor="titulo" style={{ display: 'block', fontWeight: 'bold' }}>Título:</label>
        <input id="titulo" name="titulo" type="text" required style={{ width: '100%', padding: '0.5rem' }} />
      </div>

      <div>
        <label htmlFor="contenido" style={{ display: 'block', fontWeight: 'bold' }}>Contenido:</label>
        <textarea id="contenido" name="contenido" rows={4} required style={{ width: '100%', padding: '0.5rem' }} />
      </div>

      <button type="submit" disabled={pendiente} style={{ padding: '0.75rem', cursor: 'pointer' }}>
        {pendiente ? 'Guardando en Base de Datos...' : 'Publicar Post'}
      </button>

      {mensaje && (
        <p style={{ color: mensaje.includes('éxito') ? 'green' : 'red', fontWeight: 'bold' }}>
          {mensaje}
        </p>
      )}
    </form>
  );
}
```

---

## Resumen del Capítulo

*   La **arquitectura híbrida** de React v19 combina componentes ejecutados en servidor (RSC) y cliente (`'use client'`) para optimizar el bundle de descarga.
*   Los **React Server Components (RSC)** son el estándar por defecto. Tienen acceso directo al backend físico y pesan **0 bytes en el bundle de JavaScript** del navegador.
*   Los **Client Components** manejan la interactividad de la pantalla (hooks, listeners) y se definen con la directiva `'use client'`.
*   Las **React Server Actions** permiten invocar funciones seguras de backend de Node.js de forma directa desde formularios de cliente empleando el prop `action`, habilitando la mutación segura de datos y la eliminación de APIs intermediarias duplicadas.

En el próximo capítulo, estudiaremos el motor de enrutamiento dinámico indispensable para la creación de Single Page Applications profesionales utilizando **React Router v6+** y sus mecanismos modernos de precarga de datos (*Data Loaders*).

---

[Capítulo anterior](09-suspense-y-concurrencia.md) | [Inicio](README.md) | [Capítulo siguiente →](11-react-router.md)
