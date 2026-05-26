# Capítulo 3: Estado, Props y Flujo de Datos Unidireccional

> "El flujo de datos unidireccional no es una limitación técnica; es una decisión de diseño arquitectónico que dota a las aplicaciones de previsibilidad, trazabilidad y facilidad de depuración."

En React, los datos fluyen de arriba hacia abajo, de forma descendente y lineal. Este diseño arquitectónico se denomina **flujo de datos unidireccional (Unidirectional Data Flow)**. Para construir interfaces reactivas y dinámicas bajo este paradigma, debemos dominar las dos estructuras de datos fundamentales que gobiernan la vida de cualquier componente: las **Props** y el **Estado (State)**.

---

## 3.1 Props: El Contrato Inmutable

Las **Props** (abreviatura de *properties*) representan los parámetros de entrada que un componente padre le transmite a un componente hijo. En el paradigma funcional de React, las Props son equivalentes a los argumentos que le pasas a una función de JavaScript.

### Regla Fundamental: Las Props son de Solo Lectura (Inmutables)
Un componente **jamás debe modificar directamente sus propias props**. Si intentas alterar una propiedad dentro de un componente hijo, React emitirá advertencias severas y romperá la integridad computacional del árbol.

```typescript
// ❌ CÓDIGO INCORRECTO Y PROHIBIDO
interface CardProps {
  titulo: string;
}

export function Card(props: CardProps) {
  // Intentar reescribir una prop recibida romperá el paradigma de React
  props.titulo = "Nuevo Título"; 
  return <h2>{props.titulo}</h2>;
}
```

*Por qué la inmutabilidad es crítica:* Si un componente hijo pudiera alterar las propiedades que recibe, alteraría indirectamente el estado de su componente padre y de todos los demás componentes que comparten ese dato, creando un flujo bidireccional caótico y haciendo imposible rastrear de dónde proviene un bug.

---

## 3.2 Estado: El Corazón Reactivo

Si las Props son inmutables, ¿cómo hacemos para que nuestra interfaz responda a clics, entradas de texto, animaciones o peticiones de red? Para esto disponemos del **Estado (State)**.

El Estado representa los datos internos y mutables de un componente en un momento dado. A diferencia de las Props, el Estado es **privado y encapsulado**: pertenece exclusivamente al componente que lo declara, y ningún componente externo tiene acceso directo a él.

En React moderno, el estado se declara y manipula utilizando el hook **`useState`**.

> [!NOTE]
> ### 🛂 La Analogía del Pasaporte vs. El Diario Íntimo
> 
> Para entender la diferencia fundamental entre Props y Estado, imagina que eres un viajero recorriendo el mundo:
> 
> - Las **Props** son equivalentes a tu **Pasaporte de Viajero**: Es un documento emitido por una entidad externa y gubernamental (el componente padre). Tú no puedes escribir sobre él, cambiar tu foto, alterar tu fecha de nacimiento ni añadir sellos a tu antojo (inmutabilidad). Solo puedes presentarlo y mostrarlo para que otros lean tus datos en las aduanas para permitirte pasar (**renderizar**).
> - El **Estado (State)** es equivalente a tu **Diario Íntimo**: Es un cuaderno personal y privado que llevas dentro de tu mochila (el componente). Nadie fuera de ti puede leerlo ni escribir en él a menos que tú decidas compartir voluntariamente una página de tu diario con un amigo (**pasar el estado hacia abajo como una prop**). Eres libre de tomar tu bolígrafo especial (**la función `setState`**) y escribir o tachar tus notas cuando tus pensamientos cambien.

---

## 3.3 El Proceso de Actualización Asíncrono: State Batching

Cuando ejecutas la función actualizadora de estado (`setAlgo`), podrías cometer el error de pensar que el valor en memoria de la variable cambia inmediatamente en la siguiente línea de código de tu función.

### El Error Clásico de Lectura Síncrona:
```typescript
const [contador, setContador] = useState<number>(0);

const handleIncrementar = () => {
  setContador(contador + 1);
  
  // ⚠️ IMPRIME 0 EN CONSOLA, ¡NO 1!
  console.log("Contador actual:", contador); 
};
```

*¿Por qué sucede esto?* En JavaScript, el valor de `contador` se comporta como una constante que pertenece a la renderización actual. Cuando llamas a `setContador`, no estás modificando la variable local `contador` en caliente; estás **planificando una nueva renderización futura de React** con el nuevo valor.

Para optimizar drásticamente el rendimiento, React utiliza un proceso denominado **State Batching**.

> [!NOTE]
> ### 🛒 La Analogía de la Lista de Compras Agrupada
> 
> Imagina que estás descansando en tu sala y tu pareja está trabajando en la cocina preparando una cena especial.
> 
> - Si tu pareja descubre que falta sal, sale a la sala y te grita: *"Por favor, ve al supermercado a comprar sal"*. 
> - Un segundo después se da cuenta de que tampoco hay pimienta y grita: *"¡También trae pimienta!"*.
> - De camino a la puerta, añade: *"¡Y una botella de aceite de oliva!"*.
> - Si hicieras un viaje individual de ida y vuelta a la tienda por cada ingrediente que te grita por separado, perderías tiempo, energía y combustible de forma ineficiente (**múltiples ciclos de renderizado y repintado de pantalla**).
> - Lo que haces intuitivamente es esperar un momento en la sala, agrupar todos los ingredientes en una sola lista de compras en papel, y hacer **un único viaje consolidado** a la tienda para traer todo.
> - **React hace exactamente lo mismo**: Si disparas 3 o 4 actualizaciones de estado seguidas dentro de una misma función de evento, React no vuelve a calcular y pintar el Virtual DOM 4 veces. Agrupa todas las actualizaciones de estado en una "lista de compras" asíncrona, y ejecuta **un único viaje de renderizado óptimo**.

---

## 3.4 Actualizaciones de Estado Basadas en el Estado Anterior

Debido al carácter asíncrono y agrupado (*Batching*) de las actualizaciones de estado, programar múltiples incrementos seguidos de forma directa fallará estrepitosamente:

```typescript
const [contador, setContador] = useState<number>(0);

const handleTripleIncremento = () => {
  // React recibe: "Planificar contador como 0 + 1 = 1"
  setContador(contador + 1);
  // React recibe: "Planificar contador como 0 + 1 = 1"
  setContador(contador + 1);
  // React recibe: "Planificar contador como 0 + 1 = 1"
  setContador(contador + 1);
  
  // Al final del lote, el contador solo sube a 1, no a 3.
};
```

### La Solución Definitiva: Actualizador Funcional
Si tu nuevo estado depende matemáticamente o conceptualmente del estado anterior, debes pasar una **función callback** en lugar del valor directo. Esta función recibirá el valor más reciente y actualizado del estado garantizado por el motor de React:

```typescript
const handleTripleIncrementoCorrecto = () => {
  // React encadena las funciones callback en una cola secuencial garantizada:
  setContador(prev => prev + 1); // prev recibe 0, retorna 1
  setContador(prev => prev + 1); // prev recibe 1, retorna 2
  setContador(prev => prev + 1); // prev recibe 2, retorna 3
  
  // El contador sube a 3 correctamente al finalizar el lote.
};
```

---

## 3.5 Flujo de Datos Unidireccional y Elevación del Estado

Dado que en React los componentes secundarios no pueden comunicarse directamente entre sí de forma lateral (un componente hermano no puede ver el estado de su hermano), debemos aplicar el patrón de **Elevación del Estado (Lifting State Up)**.

Si dos o más componentes necesitan consumir o sincronizarse a partir de los mismos datos cambiantes, debemos **identificar al ancestro común más cercano** en el árbol de componentes, declarar el estado en dicho padre y distribuir el dato a los hijos mediante Props.

```
       ┌────────────────────────┐
       │   Padre (State: query) │
       └───────────┬────────────┘
                   │
         ┌─────────┴─────────┐
 ┌───────▼────────┐  ┌───────▼────────┐
 │ FiltroComponent│  │ ListaComponent │
 │ (Recibe: query)│  │ (Recibe: query)│
 └────────────────┘  └────────────────┘
```

Si el componente hijo necesita modificar el estado del padre, el padre le pasa una **función Callback de actualización** como Prop.

### Ejemplo Completo y Tipado Estricto de Elevación de Estado:

#### 1. `Filtro.tsx` (Componente Dumb/Hijo)
```typescript
import React from 'react';

// Tipar estrictamente las Props que se reciben, incluyendo la callback
interface FiltroProps {
  valorActual: string;
  onCambio: (nuevoValor: string) => void;
}

export function Filtro({ valorActual, onCambio }: FiltroProps): React.JSX.Element {
  return (
    <div style={{ marginBottom: '1rem' }}>
      <label htmlFor="input-filtro" style={{ fontWeight: 'bold', display: 'block' }}>
        Filtrar Elementos:
      </label>
      <input
        id="input-filtro"
        type="text"
        value={valorActual}
        onChange={(e) => onCambio(e.target.value)}
        placeholder="Escribe para buscar..."
        style={{
          padding: '0.5rem',
          width: '100%',
          boxSizing: 'border-box',
          marginTop: '0.5rem'
        }}
      />
    </div>
  );
}
```

#### 2. `Dashboard.tsx` (Componente Smart/Padre)
```typescript
import React, { useState } from 'react';
import { Filtro } from './Filtro';

interface Item {
  id: number;
  nombre: string;
}

const ITEMS_INICIALES: Item[] = [
  { id: 1, nombre: 'React Native' },
  { id: 2, nombre: 'Vite Compiler' },
  { id: 3, nombre: 'TypeScript Framework' },
  { id: 4, nombre: 'Zustand Store' }
];

export default function Dashboard(): React.JSX.Element {
  // Declarar el estado en el padre común
  const [terminoBusqueda, setTerminoBusqueda] = useState<string>('');

  // Computar valores derivados dinámicamente en cada renderizado (optimiza memoria)
  const itemsFiltrados = ITEMS_INICIALES.filter(item =>
    item.nombre.toLowerCase().includes(terminoBusqueda.toLowerCase())
  );

  return (
    <section style={{ maxWidth: '500px', margin: '2rem auto', border: '1px solid #ddd', padding: '1.5rem', borderRadius: '8px' }}>
      <h2>Buscador de Tecnologías</h2>
      
      {/* Pasar el estado y la callback de actualización al componente Filtro */}
      <Filtro 
        valorActual={terminoBusqueda} 
        onCambio={setTerminoBusqueda} 
      />

      <h3>Resultados:</h3>
      <ul>
        {itemsFiltrados.map(item => (
          <li key={item.id} style={{ padding: '0.25rem 0' }}>
            {item.nombre}
          </li>
        ))}
        {itemsFiltrados.length === 0 && (
          <li style={{ color: 'red', listStyleType: 'none' }}>No se encontraron elementos.</li>
        )}
      </ul>
    </section>
  );
}
```

---

## Resumen del Capítulo

*   Las **Props** son de **solo lectura e inmutables**. Representan los datos externos que el componente padre pasa al hijo para configurar su comportamiento visual.
*   El **Estado (State)** es **privado, mutable y encapsulado**. Permite que el componente gestione dinámicamente los datos que cambian a lo largo del tiempo.
*   React agrupa múltiples actualizaciones de estado seguidas en un único ciclo de renderizado (**State Batching**), optimizando drásticamente la interacción con el DOM del navegador.
*   Cuando el nuevo estado depende del valor anterior, se debe usar la **firma funcional del actualizador** (`setContador(prev => prev + 1)`) para evitar colisiones asíncronas y bugs visuales.
*   Para sincronizar la comunicación entre componentes laterales, se aplica el patrón **Elevación del Estado (Lifting State Up)**, gestionando el flujo unidireccional de datos mediante Props de datos y Callbacks.

En el próximo capítulo, estudiaremos cómo conectar nuestra aplicación de React con el mundo exterior mediante el **ciclo de vida de los componentes** y las suscripciones/sincronizaciones utilizando el hook **`useEffect`**.

---

[Capítulo anterior](02-jsx-y-reconciliacion.md) | [Inicio](README.md) | [Capítulo siguiente →](04-ciclo-de-vida-y-efectos.md)
