# Capítulo 5: Hooks Fundamentales y Custom Hooks

> "Los Hooks no son magia negra; son un diseño elegante basado en arrays secuenciales que permite asociar estado mutable y lógica a componentes puramente funcionales."

La introducción de los Hooks en React 16.8 supuso una de las mayores simplificaciones de la arquitectura del desarrollo frontend. Al permitir que los componentes de función controlen estado, efectos y referencias sin recurrir a la verbosidad de las clases (`this`, `bind`), los Hooks abrieron las puertas a una modularidad sin precedentes.

En este capítulo, profundizaremos en el motor de ejecución secuencial interno de los Hooks, estudiaremos las herramientas de optimización y memorización nativas, y aprenderemos a extraer lógica empresarial compleja en nuestros propios **Custom Hooks** robustamente tipados.

---

## 5.1 El Motor Interno: Las Reglas de los Hooks

Para usar Hooks con total confianza, debemos desmitificar cómo los gestiona React tras bambalinas. Hay dos reglas de oro que el linter te obligará a cumplir:
1.  **Llamar a los Hooks solo en el nivel superior**: No los llames dentro de bucles (`for`), condiciones (`if`) o funciones anidadas.
2.  **Llamar a los Hooks solo desde funciones de React**: Llámalos desde componentes funcionales o desde tus propios Custom Hooks.

### ¿Por qué existe esta restricción?
Internamente, **React no identifica a tus Hooks por su nombre o su firma**. Para React, un componente funcional es una función que ejecuta en orden secuencial un conjunto de declaraciones de Hooks. React asocia el estado almacenándolo en una **lista enlazada simple (o un array en memoria)** donde el puntero avanza secuencialmente en cada llamada a un Hook.

```
Render 1 (Inicial):
[ Hook 1 (useState) ]  ──►  [ Hook 2 (useEffect) ]  ──►  [ Hook 3 (useRef) ]
  (Puntero: 0)                 (Puntero: 1)                 (Puntero: 2)
```

Si decides envolver el Hook 2 en una condicional `if` y en el Render 2 la condición resulta falsa, React ejecutará el Hook 1, se saltará el Hook 2 e intentará asignar el estado guardado del Hook 2 al Hook 3, **desfasando y corrompiendo por completo los datos de la aplicación**.

> [!NOTE]
> ### 💊 La Analogía del Pastillero Semanal Secuencial
> 
> Imagina que tienes un organizador de pastillas semanal de plástico con compartimentos numerados de lunes a domingo:
> 
> - Cada día de la semana representa un **Hook** secuencial dentro de tu componente de React.
> - El lunes tomas la pastilla del compartimento 1 (**`useState`**), el martes la del compartimento 2 (**`useEffect`**), y el miércoles la del compartimento 3 (**`useRef`**).
> - Tu cuerpo (el motor de React) funciona asumiendo que sigues estrictamente esta rutina secuencial sin saltarte días.
> - Si decides que el martes amaneció nublado y pones la pastilla del martes dentro de un condicional: *"Si llueve, me salto el martes"*, tu cuerpo mecánicamente abrirá el siguiente casillero disponible. Pensará que el casillero del miércoles es en realidad el del martes. Acabas de desfasar toda tu medicación, tomando la dosis equivocada y rompiendo el equilibrio interno de tu salud (**corrupción del grafo de estados en memoria**).

---

## 5.2 Memorización Avanzada: useMemo y useCallback

Cada vez que un componente cambia de estado, ejecuta su función completa de nuevo, recreando todas las variables locales, cálculos y referencias de funciones internas. Para evitar cálculos ineficientes o renders inútiles en componentes hijos, React nos provee dos herramientas de memorización.

### 1. useMemo: Memorización de Valores Computados
`useMemo` almacena en caché el **resultado de un cálculo costoso** entre renders. Solo vuelve a calcular el valor si una de sus dependencias declaradas en el array cambia.

```typescript
// Optimización de filtrado de datos masivo
const itemsFiltrados = useMemo(() => {
  console.log("Calculando filtrado complejo...");
  return itemsPesados.filter(item => item.precio > umbralMinimo);
}, [itemsPesados, umbralMinimo]); // Solo recalcula si el array o el umbral cambian
```

### 2. useCallback: Memorización de Referencias de Funciones
En JavaScript, dos funciones con el mismo cuerpo no son idénticas por referencia (`() => {} !== () => {}`). Cada vez que un componente se re-renderiza, crea una nueva referencia de sus funciones internas. Si pasas estas funciones como props a componentes hijos, los hijos se re-renderizarán inútilmente.

`useCallback` memoriza **la referencia de la función** en sí misma:

```typescript
const handleSeleccion = useCallback((id: number) => {
  console.log("Elemento seleccionado:", id);
}, []); // La referencia de la función es estable y nunca cambia
```

> [!TIP]
> **No abuses de la memorización**: Memorizar tiene un costo (guardar referencias en memoria y comparar el array de dependencias en cada render). Si tu cálculo es una simple suma de arrays pequeños, el costo de usar `useMemo` es mayor que el cálculo nativo directo. Úsalo solo para operaciones realmente costosas o para asegurar estabilidad de referencias en props que van a componentes hijos optimizados con `React.memo`.

---

## 5.3 useRef: Persistencia Mutable y Acceso al DOM

El hook `useRef` sirve para dos propósitos fundamentales:

1.  **Persistir valores mutables sin disparar re-renderizados**: A diferencia de `useState`, si modificas el valor de `referencia.current`, el componente no vuelve a pintarse en pantalla. Es ideal para almacenar temporizadores (`NodeJS.Timeout`), estados de scroll previos o flags de montaje.
2.  **Acceso directo a nodos del DOM físico**: Permite interactuar con APIs del navegador como enfocar un input, controlar reproductores de video HTML5 o medir dimensiones físicas de elementos.

### Ejemplo Práctico de useRef:
```typescript
import React, { useRef, useEffect } from 'react';

export function AutofocusInput(): React.JSX.Element {
  // 1. Declarar la referencia tipando el elemento HTML específico
  const inputRef = useRef<HTMLInputElement | null>(null);

  const enfocarElemento = () => {
    // 3. Acceder al elemento físico usando .current
    inputRef.current?.focus();
  };

  useEffect(() => {
    // Enfocar automáticamente el input al montarse el componente
    enfocarElemento();
  }, []);

  return (
    <div>
      <input ref={inputRef} type="text" placeholder="Escribe aquí..." />
      <button onClick={enfocarElemento}>Enfocar Input</button>
    </div>
  );
}
```

---

## 5.4 Custom Hooks: Abstracción de Lógica Reactiva

Los **Custom Hooks** son simplemente funciones de JavaScript convencionales cuyo nombre debe comenzar con la palabra `use` y que pueden llamar a otros Hooks de React. Permiten desacoplar la lógica de estado y sincronización de la capa visual del componente.

### Creación de un Custom Hook de Fetching con TypeScript:

#### [useFetch.ts](file:///Users/andres/Documents/biblioteca/react-programming-book/src/hooks/useFetch.ts)
```typescript
import { useState, useEffect } from 'react';

interface FetchState<T> {
  data: T | null;
  loading: boolean;
  error: string | null;
}

export function useFetch<T>(url: string): FetchState<T> {
  const [state, setState] = useState<FetchState<T>>({
    data: null,
    loading: true,
    error: null,
  });

  useEffect(() => {
    const controlador = new AbortController();
    
    setState({ data: null, loading: true, error: null });

    const fetchData = async () => {
      try {
        const respuesta = await fetch(url, { signal: controlador.signal });
        
        if (!respuesta.ok) {
          throw new Error(`Error en la petición: ${respuesta.statusText}`);
        }
        
        const datos: T = await respuesta.json();
        setState({ data: datos, loading: false, error: null });
      } catch (err: any) {
        if (err.name !== 'AbortError') {
          setState({ data: null, loading: false, error: err.message || 'Error desconocido' });
        }
      }
    };

    fetchData();

    return () => {
      controlador.abort();
    };
  }, [url]);

  return state;
}
```

---

## Resumen del Capítulo

*   React gestiona el estado de los Hooks de forma secuencial en una **lista enlazada en memoria**. Por ello, es obligatorio mantener el orden exacto de ejecución en cada renderizado (sin condicionales ni bucles).
*   **`useMemo`** memoriza el resultado de operaciones matemáticas o filtros costosos en JavaScript.
*   **`useCallback`** memoriza la referencia física de una función para evitar re-renderizados en componentes hijos optimizados con `React.memo`.
*   **`useRef`** permite acceder a nodos físicos del DOM del navegador y persistir valores mutables entre renders sin gatillar ciclos de repintado de pantalla.
*   Los **Custom Hooks** son la herramienta por excelencia para aislar, empaquetar y reutilizar la lógica de estado en cualquier lugar de la aplicación de forma limpia y testeable.

En el próximo capítulo, resolveremos el gran dolor de cabeza de las aplicaciones de gran profundidad de componentes: el intercambio y la sincronización de estado compartido mediante **Context API** de forma óptima.

---

[Capítulo anterior](04-ciclo-de-vida-y-efectos.md) | [Inicio](README.md) | [Capítulo siguiente →](06-context-api.md)
