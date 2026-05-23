# Capítulo 7: Gestión de Estado Avanzada con useReducer

> "Un reductor es una función matemática pura de transición de estado. Dado un estado A y una acción B, siempre, sin excepción, devolverá el mismo estado C."

Cuando construimos componentes complejos que gestionan múltiples variables de estado interrelacionadas (por ejemplo, un formulario de varios pasos, una tabla con filtros, ordenación y paginación, o un carrito de compras interactivo), depender exclusivamente de múltiples hooks `useState` dispersos provoca un código frágil, difícil de testear y propenso a bugs de inconsistencia de datos.

Para gobernar estados masivos y complejos de forma predecible, React nos provee el hook **`useReducer`**. En este capítulo, estudiaremos los fundamentos del patrón Reducer, las reglas matemáticas de las funciones de transición pura y aprenderemos a fusionar `useReducer` con Context API para erigir un **motor de estado global robusto y tipado** sin añadir librerías externas.

---

## 7.1 El Desafío del Estado Complejo e Interrelacionado

Imagina que estás construyendo una bandeja de entrada de correos electrónicos. Tu estado requiere rastrear:
*   La lista de correos descargados.
*   Si los datos están cargándose de la API (`loading`).
*   Si ocurrió un error de red (`error`).
*   Los correos que el usuario ha seleccionado para borrar.

Si gestionas esto con `useState`, tus funciones se llenarán de llamadas manuales desordenadas:

```typescript
// ⚠️ ESTADO DISPERSO Y PROTÓTIPO DE INCONSISTENCIA
const [correos, setCorreos] = useState([]);
const [loading, setLoading] = useState(false);
const [error, setError] = useState(null);

const cargarCorreos = async () => {
  setLoading(true);
  setError(null);
  try {
    const res = await fetch('/api/emails');
    const datos = await res.json();
    setCorreos(datos);
    setLoading(false); // ¿Qué pasa si te olvidas de apagar el loading en el catch?
  } catch (err) {
    setError("Error al cargar");
    // Si te olvidas de apagar el loading aquí, la UI se congelará en spinner eterno
  }
};
```

Este enfoque descentralizado permite transiciones de estado imposibles o absurdas en la vida real, como tener `loading: true` al mismo tiempo que mostramos un mensaje de `error` y datos antiguos en pantalla.

---

## 7.2 El Patrón Reducer y la Máquina de Estados

El patrón Reducer centraliza todas las mutaciones en un único punto de control. Se basa en tres pilares conceptuales:

1.  **Estado (State)**: Un único objeto de lectura inmutable que representa la verdad absoluta del componente.
2.  **Acciones (Actions)**: Objetos que describen *qué* ocurrió en la interfaz de usuario, pero no *cómo* debe cambiar el estado. Contienen un tipo identificador (`type`) y opcionalmente un cargamento de datos (`payload`).
3.  **Reductor (Reducer)**: Una **función pura** que intercepta la acción, lee el estado actual, realiza cálculos matemáticos en memoria y retorna el *nuevo* estado resultante.

```
                  ┌────────────────────────┐
                  │   Interfaz (UI View)   │
                  └───────────┬────────────┘
                              │
                              │  (Gatilla Acción: dispatch)
                              ▼
                  ┌────────────────────────┐
                  │    Acción (Action)     │
                  └───────────┬────────────┘
                              │
                              ▼
┌───────────────────┐    ┌────┴────┐
│  Estado Anterior  │───►│ Reducer │ (Función Pura de Transición)
└───────────────────┘    └────┬────┘
                              │
                              ▼
                  ┌───────────┴────────────┐
                  │   Nuevo Estado (State) │  (Re-renderiza la UI)
                  └────────────────────────┘
```

> [!NOTE]
> ### 🎛️ La Analogía de la Cabina del Piloto del Avión
> 
> Para comprender la diferencia entre `useState` y `useReducer`, imagina la diferencia entre conducir un coche de juguete y pilotar un avión Boeing 747:
> 
> - En un coche de juguete simple, un pequeño volante y un pedal son suficientes (**`useState`**). Presionas el pedal y el coche avanza de forma síncrona y directa. Es un mecanismo de control ligero.
> - Pero para pilotar un avión gigante con 300 pasajeros, no puedes dejar que cualquiera tire de cables mecánicos o mueva alerones de forma directa en caliente. Sería catastrófico.
> - En su lugar, el avión cuenta con una **Cabina de Mando (El Reducer)** estructurada y segura.
> - El copiloto no altera la posición de los flaps directamente con las manos. Envía un mensaje de radio estandarizado (**dispatch**) al sistema electrónico: *"Acción: DESPLEGAR_TREN_DE_ATERRIZAJE"*.
> - El software de vuelo puro del avión (**El Reducer**) intercepta ese comando preciso, consulta el manual de seguridad electrónica y cambia la altitud y los alerones de forma perfectamente coordinada y blindada contra fallos humanos. La nave transiciona a un estado de vuelo seguro y estable.

---

## 7.3 Firma y Reglas de la Función Reductora

Una función reductora debe ser obligatoriamente una **función pura**:
*   **No debe realizar efectos colaterales**: Queda terminantemente prohibido hacer fetch de APIs, escribir en LocalStorage o interactuar con el DOM dentro del reductor.
*   **No debe generar valores aleatorios**: No uses `Math.random()`, `Date.now()` o UUIDs dinámicos dentro del reductor.
*   **Debe respetar la inmutabilidad**: Nunca modifiques el objeto `state` recibido directamente; debes retornar un objeto completamente nuevo utilizando el operador de propagación (*spread operator* `...`).

```typescript
// Firma matemática del reductor
const reducer = (state: State, action: Action): State => { ... }
```

---

## 7.4 Ejemplo Práctico: Bandeja de Entrada con useReducer y TypeScript

#### [emailReducer.ts](file:///Users/andres/Documents/biblioteca/react-programming-book/src/reducers/emailReducer.ts)
```typescript
import React, { useReducer } from 'react';

// 1. Declarar los tipos estrictos del Estado
interface Email {
  id: number;
  asunto: string;
  leido: boolean;
}

interface EmailState {
  emails: Email[];
  loading: boolean;
  error: string | null;
}

// 2. Declarar la unión de tipos para las Acciones (Action Union)
type EmailAction =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: Email[] }
  | { type: 'FETCH_FAILURE'; payload: string }
  | { type: 'MARCAR_LEIDO'; payload: number }
  | { type: 'ELIMINAR_EMAIL'; payload: number };

const ESTADO_INICIAL: EmailState = {
  emails: [],
  loading: false,
  error: null,
};

// 3. Crear la función reductora pura
function emailReducer(state: EmailState, action: EmailAction): EmailState {
  switch (action.type) {
    case 'FETCH_START':
      return {
        ...state,
        loading: true,
        error: null,
      };
    case 'FETCH_SUCCESS':
      return {
        ...state,
        loading: false,
        emails: action.payload,
      };
    case 'FETCH_FAILURE':
      return {
        ...state,
        loading: false,
        error: action.payload,
      };
    case 'MARCAR_LEIDO':
      return {
        ...state,
        emails: state.emails.map(email =>
          email.id === action.payload ? { ...email, leido: true } : email
        ),
      };
    case 'ELIMINAR_EMAIL':
      return {
        ...state,
        emails: state.emails.filter(email => email.id !== action.payload),
      };
    default:
      return state;
  }
}

// 4. Componente de UI interactivo
export function BandejaEntrada(): React.JSX.Element {
  const [state, dispatch] = useReducer(emailReducer, ESTADO_INICIAL);

  const mockCargarDatos = () => {
    dispatch({ type: 'FETCH_START' });
    
    // Simulación de carga asíncrona (el fetch real iría en un useEffect o Callback)
    setTimeout(() => {
      const datosMock: Email[] = [
        { id: 1, asunto: 'Reunión de sprint trimestral', leido: false },
        { id: 2, asunto: 'Factura de AWS pendiente', leido: false },
        { id: 3, asunto: 'Feedback de Code Review', leido: true },
      ];
      dispatch({ type: 'FETCH_SUCCESS', payload: datosMock });
    }, 1000);
  };

  return (
    <div style={{ padding: '1rem', border: '1px solid #ccc', borderRadius: '8px' }}>
      <h2>Bandeja de Entrada Recibidos</h2>
      <button onClick={mockCargarDatos} disabled={state.loading}>
        {state.loading ? 'Cargando...' : 'Sincronizar Correos'}
      </button>

      {state.error && <p style={{ color: 'red' }}>Error: {state.error}</p>}

      <ul style={{ marginTop: '1rem' }}>
        {state.emails.map(email => (
          <li 
            key={email.id} 
            style={{ 
              padding: '0.5rem', 
              borderBottom: '1px solid #eee',
              backgroundColor: email.leido ? '#f9f9f9' : '#eef8ff',
              display: 'flex',
              justifyContent: 'space-between'
            }}
          >
            <span style={{ fontWeight: email.leido ? 'normal' : 'bold' }}>
              {email.asunto}
            </span>
            <div>
              {!email.leido && (
                <button onClick={() => dispatch({ type: 'MARCAR_LEIDO', payload: email.id })}>
                  Marcar Leído
                </button>
              )}
              <button 
                onClick={() => dispatch({ type: 'ELIMINAR_EMAIL', payload: email.id })}
                style={{ marginLeft: '0.5rem', color: 'red' }}
              >
                Eliminar
              </button>
            </div>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

## 7.5 Creando una Store Global Nativa: Context + useReducer

Una de las joyas arquitectónicas más elegantes de React es combinar **Context API** con **`useReducer`**. Al proveer el estado `state` en un contexto y la función `dispatch` en otro, creas un motor de estado global completo y robusto idéntico al patrón Redux clásico, pero sin dependencias externas:

*   **Contexto de Estado**: Emite el estado inmutable a los consumidores para pintar la UI.
*   **Contexto de Despacho**: Emite la función `dispatch` estable para que cualquier botón o componente secundario gatille acciones sin re-renderizar a otros componentes.

```typescript
// Contexto dividido para alta eficiencia
export const EmailEstadoContext = createContext<EmailState | undefined>(undefined);
export const EmailDespachoContext = createContext<React.Dispatch<EmailAction> | undefined>(undefined);
```

---

## Resumen del Capítulo

*   **`useReducer`** es ideal para gestionar estados complejos, masivos o interrelacionados, centralizando todas las transiciones de datos del componente en un solo punto.
*   Una **función reductora debe ser pura**: sin efectos secundarios, sin valores aleatorios y respetando estrictamente la inmutabilidad de los datos.
*   Las **Acciones** describen eventos del negocio ocurridos en la UI (`type`) y llevan cargamentos de datos opcionales (`payload`).
*   Fusionar **Context API con `useReducer`** nos permite armar una **arquitectura de Store Global nativa**, tipada y de alto rendimiento para aplicaciones enterprise sin sobrecargar el bundle de producción de JavaScript.

En el próximo capítulo (iniciando el Bloque de Producción Avanzado), exploraremos cómo migrar a gestores de estado a escala global analizando la arquitectura de stores y dominando el nuevo estándar de la industria: **Zustand**.

---

[Capítulo anterior](06-context-api.md) | [Inicio](README.md) | [Capítulo siguiente →](08-estado-global.md)
