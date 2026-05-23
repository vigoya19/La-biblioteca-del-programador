# Capítulo 4: Ciclo de Vida y Efectos con useEffect

> "Un efecto no es un ciclo de vida en sí mismo; es un mecanismo para sincronizar el estado de tu aplicación con un sistema externo fuera del control de React."

Para construir aplicaciones interactivas del mundo real, los componentes no pueden vivir aislados. Deben comunicarse con APIs externas, suscribirse a WebSockets, interactuar con el DOM del navegador directamente o activar temporizadores. En React moderno, todas estas operaciones secundarias se engloban bajo el concepto de **efectos secundarios (side effects)** y se gestionan a través del hook central **`useEffect`**.

En este capítulo, desmitificaremos el ciclo de vida de los componentes basados en funciones y aprenderemos a sincronizar la UI con sistemas externos de forma segura, eficiente y libre de fugas de memoria.

---

## 4.1 Fases del Ciclo de Vida del Componente

A diferencia de los antiguos componentes de clase con sus complejos métodos como `componentDidMount`, `componentDidUpdate` y `componentWillUnmount`, los componentes de función modernos simplifican este modelo mental. Un componente de React solo tiene dos momentos fundamentales: **renderizado** y **sincronización**.

El ciclo de vida se puede resumir en tres fases lógicas en el DOM real:

```
    1. MONTAJE (Mount)              2. ACTUALIZACIÓN (Update)            3. DESMONTAJE (Unmount)
┌───────────────────────┐        ┌─────────────────────────┐        ┌────────────────────────┐
│ - Inicializa Estado   │        │ - Recibe Nuevas Props   │        │ - Se ejecuta Cleanup   │
│ - Renderiza JSX       │ ──►    │ - Cambia Estado Local   │ ──►    │   del último efecto    │
│ - Inserta en el DOM   │        │ - Re-renderiza JSX      │        │ - Se remueve el nodo   │
│ - Ejecuta useEffect   │        │ - Ejecuta Cleanup y VDom│        │   del DOM físico       │
└───────────────────────┘        └─────────────────────────┘        └────────────────────────┘
```

1.  **Montaje**: El componente nace, ejecuta su cuerpo de función por primera vez, construye su Virtual DOM inicial, lo inserta en la pantalla del navegador y ejecuta los efectos correspondientes.
2.  **Actualización**: Ante cualquier cambio de Props o de Estado, el componente vuelve a ejecutar su función, calcula las diferencias del Virtual DOM, realiza actualizaciones quirúrgicas en la pantalla y ejecuta de nuevo los efectos si sus dependencias cambiaron.
3.  **Desmontaje**: Cuando el componente ya no es necesario (por ejemplo, al cambiar de página o cerrar un modal), React remueve físicamente el nodo del DOM del navegador. Justo antes de irse, el componente ejecuta sus funciones de limpieza (*cleanup*) para liberar memoria.

---

## 4.2 La Regla de Oro: Efectos vs. Eventos

Uno de los errores más destructivos que cometen los desarrolladores en React es abusar de `useEffect` para gestionar flujos de datos que deberían ser manejados por simples eventos de usuario.

*   **Eventos (Event Handlers)**: Se disparan como respuesta directa a una **acción específica del usuario** (por ejemplo, hacer clic en un botón "Comprar", cambiar el valor de un input o enviar un formulario).
*   **Efectos (Effects)**: Se disparan de forma automática para **sincronizar el componente** con un sistema externo, siempre que un dato o estado específico cambie, sin importar *cómo* o *quién* provocó ese cambio.

> [!NOTE]
> ### ⌚ La Analogía del Reloj Inteligente (Efectos vs. Eventos)
> 
> Imagina que llevas un smartphone y un reloj inteligente (smartwatch) en tu muñeca:
> 
> - Un **Evento** es como presionar manualmente el botón del obturador de la cámara en la pantalla de tu teléfono para tomar una foto. Sucede única y exclusivamente porque realizaste una acción física deliberada en un momento exacto (**acción directa del usuario**).
> - Un **Efecto** es como el sensor de frecuencia cardíaca de tu reloj inteligente. No tienes que presionar un botón cada segundo para que mida tus pulsaciones. El reloj detecta automáticamente que tu pulso cambió o que ha pasado un minuto, y de forma transparente **sincroniza** tus datos cardíacos con el servidor en la nube en segundo plano (**sincronización automática ante cambios de estado**).

---

## 4.3 Sincronización Segura de Datos con APIs Externas

Hacer fetching de datos al montar un componente es el caso de uso más común de `useEffect`. Sin embargo, escribirlo de forma ingenua puede provocar un bug silencioso y devastador en producción: **las condiciones de carrera (Race Conditions)**.

Si el usuario hace clic rápido en varios botones para ver perfiles de usuarios diferentes (ID 1, luego ID 2, luego ID 3), se dispararán tres peticiones de red asíncronas en paralelo. No hay garantía de que la petición 3 responda al final; si la petición 1 tarda más de la cuenta por latencia de red y responde al último, la UI mostrará los datos del usuario 1 aunque el usuario actual seleccionado sea el 3.

### La Solución: AbortController y Funciones de Limpieza (Cleanup)
Para evitar race conditions y memory leaks, debemos utilizar la **función de limpieza** que retorna el efecto para abortar peticiones asíncronas en curso si el componente se actualiza o desmonta:

```typescript
import React, { useState, useEffect } from 'react';

interface Usuario {
  id: number;
  name: string;
  email: string;
}

export function DetalleUsuario({ usuarioId }: { usuarioId: number }): React.JSX.Element {
  const [usuario, setUsuario] = useState<Usuario | null>(null);
  const [cargando, setCargando] = useState<boolean>(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    // 1. Crear un AbortController para cancelar la petición si el ID cambia a mitad de camino
    const controlador = new AbortController();
    const { signal } = controlador;

    const cargarDatos = async () => {
      try {
        setCargando(true);
        setError(null);
        
        const respuesta = await fetch(`https://jsonplaceholder.typicode.com/users/${usuarioId}`, { signal });
        
        if (!respuesta.ok) {
          throw new Error('No se pudo obtener la información del usuario.');
        }
        
        const datos: Usuario = await respuesta.json();
        setUsuario(datos);
      } catch (err: any) {
        if (err.name !== 'AbortError') {
          setError(err.message || 'Ocurrió un error inesperado.');
        }
      } finally {
        setCargando(false);
      }
    };

    cargarDatos();

    // 2. RETORNAR LA FUNCIÓN DE LIMPIEZA
    // React ejecutará esta función antes de ejecutar el efecto de nuevo o al desmontar
    return () => {
      controlador.abort(); // Cancela la petición HTTP activa instantáneamente
    };
  }, [usuarioId]); // El efecto se vuelve a ejecutar CADA VEZ que el usuarioId cambie

  if (cargando) return <p>Cargando datos del usuario...</p>;
  if (error) return <p style={{ color: 'red' }}>Error: {error}</p>;
  if (!usuario) return <p>No hay datos disponibles.</p>;

  return (
    <article style={{ border: '1px solid #ccc', padding: '1rem', borderRadius: '6px' }}>
      <h2>{usuario.name}</h2>
      <p>Email: {usuario.email}</p>
    </article>
  );
}
```

---

## 4.4 El Array de Dependencias: La Guía Definitiva

El segundo argumento de `useEffect` es un array que le indica a React cuándo debe disparar el efecto. Su comportamiento cambia radicalmente según lo configures:

```typescript
// CASO 1: Sin array de dependencias (⚠️ EVITAR GENERALMENTE)
useEffect(() => {
  console.log("Se ejecuta en CADA renderizado del componente.");
});

// CASO 2: Con array de dependencias vacío (📥 MONTAJE Y DESMONTAJE)
useEffect(() => {
  console.log("Se ejecuta ÚNICAMENTE al montarse el componente por primera vez.");
  return () => console.log("Se ejecuta ÚNICAMENTE al desmontarse el componente.");
}, []);

// CASO 3: Con variables en el array (🔄 ACTUALIZACIÓN CONDICIONAL)
useEffect(() => {
  console.log("Se ejecuta al montar, y luego solo si 'usuarioId' o 'token' cambian.");
}, [usuarioId, token]);
```

### La Prevención del Bucle Infinito
Un error clásico de bucle infinito ocurre cuando modificas dentro del efecto una variable de estado que está declarada como dependencia del propio efecto:

```typescript
// ❌ CÓDIGO INCORRECTO: BUCLE INFINITO
const [contador, setContador] = useState(0);

useEffect(() => {
  // Al cambiar el estado, se gatilla un re-render.
  // El re-render ejecuta el efecto nuevamente.
  // El efecto vuelve a cambiar el estado... bucle infinito.
  setContador(contador + 1); 
}, [contador]);
```

---

## Resumen del Capítulo

*   El ciclo de vida de un componente se resume en tres fases lógicas: **Montaje** (nace), **Actualización** (cambia de estado/props) y **Desmontaje** (se destruye).
*   **Efectos vs. Eventos**: Los eventos se disparan por acciones directas del usuario; los efectos se disparan automáticamente para sincronizar la aplicación con sistemas externos.
*   Para evitar bugs de **condiciones de carrera** al hacer fetching de APIs en `useEffect`, debes retornar una función de limpieza que implemente un **`AbortController`** para cancelar peticiones obsoletas.
*   El **array de dependencias** es el cerebro de `useEffect`. Debe incluir obligatoriamente cualquier variable reactiva (props o estado) que sea consumida dentro del cuerpo del efecto.

En el próximo capítulo, profundizaremos en la modularización de la lógica interactiva analizando las reglas internas de ejecución de los Hooks y aprendiendo a diseñar nuestros propios **Custom Hooks** reutilizables para separar el diseño de la UI de la lógica de negocio.

---

[Capítulo anterior](02-jsx-y-reconciliacion.md) | [Inicio](README.md) | [Capítulo siguiente →](05-hooks-y-custom-hooks.md)
