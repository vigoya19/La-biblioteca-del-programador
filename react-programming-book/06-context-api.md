# Capítulo 6: Gestión del Estado Compartido con Context API

> "La inyección de dependencias implícita en React no requiere frameworks pesados. Context API es el cableado invisible que une tus fuentes de datos con tus consumidores más lejanos."

A medida que una aplicación de React escala y el árbol de componentes se vuelve más profundo y complejo, surge un problema clásico de diseño en el flujo unidireccional de datos: el **Prop Drilling** (o arrastre de propiedades). Este fenómeno ocurre cuando te ves obligado a pasar datos a través de múltiples capas de componentes intermedios que no necesitan esa información, solo para que pueda ser consumida por un componente hijo ubicado en lo más profundo del árbol.

En este capítulo, estudiaremos cómo solucionar este problema de raíz utilizando la herramienta de inyección de estado nativa de React: **Context API**, analizando su funcionamiento interno, sus problemas latentes de re-renderizado y las técnicas avanzadas para optimizar su rendimiento.

---

## 6.1 El Problema de Prop Drilling

Imagina una aplicación que cuenta con un tema visual (Claro / Oscuro) o almacena los datos de autenticación del usuario actual. El componente raíz `<App>` gestiona esta información en su estado local, pero el componente `<BotonConfiguracion>`, ubicado a 6 niveles de profundidad, es el que realmente necesita consumir y modificar este tema.

```
       ┌───────────┐
       │   App     │  (State: tema, setTema)
       └─────┬─────┘
             ▼
       ┌───────────┐
       │   Layout  │  (Recibe tema, setTema solo para pasarlo abajo)
       └─────┬─────┘
             ▼
       ┌───────────┐
       │  Sidebar  │  (Recibe tema, setTema)
       └─────┬─────┘
             ▼
       ┌───────────┐
       │  Ajustes  │  (Recibe tema, setTema)
       └─────┬─────┘
             ▼
 ┌─────────────────┐
 │ BotonConfigurac │  (¡Por fin consume tema y setTema!)
 └─────────────────┘
```

*Inconvenientes del Prop Drilling:*
*   **Acoplamiento rígido**: Todos los componentes del medio quedan acoplados a la estructura del tema, reduciendo su capacidad de ser reutilizados en otras partes de la aplicación.
*   **Complejidad de mantenimiento**: Si decides renombrar la propiedad o añadir un nuevo parámetro, tendrás que refactorizar de forma manual toda la cadena de componentes intermedios.
*   **Degradación del código**: Introduce verbosidad inútil y oscurece el verdadero propósito y responsabilidades de los componentes de presentación.

---

## 6.2 Context API: La Solución Nativa

**Context API** nos permite crear un "almacén de datos flotante" fuera del flujo tradicional de Props. El componente padre envuelve a su árbol en un proveedor de contexto, y cualquier componente hijo, sin importar lo profundo que esté, puede acceder directamente a ese almacén con un único Hook.

> [!NOTE]
> ### 📡 La Analogía de la Torre de Telecomunicaciones
> 
> Imagina que estás en una expedición de 15 personas caminando en fila india por la selva profunda:
> 
> - El primero de la fila (el componente padre `<App>`) lleva un mapa y una cantimplora de agua fresca.
> - Si el último de la fila (el componente hijo `<BotonConfiguracion>`) tiene sed, de forma tradicional debe gritarle al segundo, el segundo pasarle el mensaje al tercero, y así sucesivamente en una cadena tediosa (**Prop Drilling**). Si la octava persona de la fila se distrae, el mensaje o el agua se caerán, y además todos los del medio terminan agotados llevando un cubo de agua que ellos no desean beber.
> - **Context API** te propone instalar una **Torre de Telecomunicaciones de Radio (El Provider)** en la mochila del primero de la fila. La torre emite la señal del agua al aire de forma abierta.
> - El último de la fila saca un pequeño receptor de radio de su bolsillo, sintoniza la frecuencia de la expedición (**`useContext`**) y recibe la información y el agua al instante. Las personas del medio continúan su caminata ligeras de equipaje, sin enterarse ni cansarse de llevar datos que no les corresponden.

---

## 6.3 Creación, Provisión y Consumo de Contexto

El flujo de trabajo estándar con Context API consta de tres pasos muy claros en TypeScript:

### Paso 1: Crear el Contexto
Utilizamos la función `createContext` declarando un valor por defecto o un tipo nullable.

### Paso 2: Proveer el Contexto (`Provider`)
Envolvemos al árbol de componentes con el elemento `<Contexto.Provider value={...}>`, inyectándole los datos dinámicos del estado.

### Paso 3: Consumir el Contexto (`useContext`)
Usamos el hook nativo `useContext` para extraer el valor inyectado en el subárbol.

### Ejemplo Profesional: Sistema de Autenticación de Usuario

#### [AuthContext.tsx](file:///Users/andres/Documents/biblioteca/react-programming-book/src/context/AuthContext.tsx)
```typescript
import React, { createContext, useState, useContext, ReactNode } from 'react';

// 1. Definir la interfaz de los datos que guardará el contexto
interface Usuario {
  id: string;
  nombre: string;
  email: string;
}

interface AuthContextType {
  usuario: Usuario | null;
  login: (nombre: string, email: string) => void;
  logout: () => void;
}

// 2. Crear el contexto con un valor inicial indefinido
const AuthContext = createContext<AuthContextType | undefined>(undefined);

// 3. Crear el componente Proveedor personalizado
export function AuthProvider({ children }: { children: ReactNode }): React.JSX.Element {
  const [usuario, setUsuario] = useState<Usuario | null>(null);

  const login = (nombre: string, email: string) => {
    // Simulación de autenticación
    setUsuario({ id: '100', nombre, email });
  };

  const logout = () => {
    setUsuario(null);
  };

  return (
    <AuthContext.Provider value={{ usuario, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// 4. Custom Hook de consumo seguro (evita verificar undefined en cada componente)
export function useAuth(): AuthContextType {
  const contexto = useContext(AuthContext);
  
  if (!contexto) {
    throw new Error('useAuth debe utilizarse dentro de un AuthProvider.');
  }
  
  return contexto;
}
```

---

## 6.4 Optimización Avanzada: El Problema de los Re-renders Inútiles

Aunque Context API es extremadamente potente, esconde una trampa de rendimiento muy severa: **cualquier cambio en el objeto `value` provisto por el Provider gatillará un re-renderizado automático en TODOS los componentes del árbol que consuman ese contexto, incluso si solo leen una propiedad que no cambió.**

Si tu `AuthContext` provee `{ usuario, login, logout }` y cambia el valor de `usuario`, un componente que solo consumía la función `logout` (que nunca cambia de referencia) se volverá a renderizar de todas formas.

### Técnicas de Optimización de Contextos:

#### Técnica 1: Dividir el Contexto (Context Splitting)
La solución más limpia y recomendada es **crear dos contextos separados**: uno para almacenar el estado dinámico y otro para almacenar las funciones actualizadoras estables.

```typescript
// Contexto 1: Guarda únicamente el dato reactivo cambiante
export const TemaEstadoContext = createContext<string>("light");

// Contexto 2: Guarda únicamente las funciones callbacks de cambio
export const TemaAccionesContext = createContext<(() => void) | undefined>(undefined);
```

Al dividir los canales de transmisión, un botón que solo altera el tema (las acciones) nunca se re-renderizará cuando el tema en sí cambie de valor.

#### Técnica 2: Memorizar los Hijos (`children`) y el Valor del Provider
Asegúrate de envolver el objeto de valor de tu Provider en un `useMemo` para evitar que la recreación de objetos planos en cada render del padre propague invalidaciones de referencia innecesarias:

```typescript
const valorMemoizado = useMemo(() => ({ usuario, login, logout }), [usuario]);

return (
  <AuthContext.Provider value={valorMemoizado}>
    {children}
  </AuthContext.Provider>
);
```

---

## Resumen del Capítulo

*   El **Prop Drilling** acopla innecesariamente componentes intermedios de presentación a estructuras de datos que no necesitan consumir, deteriorando la mantenibilidad.
*   **Context API** inyecta de forma implícita un canal de comunicación descendente a nivel de subárbol, actuando como una **torre emisora de señal de radio**.
*   Para evitar errores de consumo fuera del Provider, es una excelente práctica encapsular el acceso en un **Custom Hook de consumo seguro** (`useAuth`).
*   Los cambios en el valor del contexto invalidan y re-renderizan a todos sus consumidores. Para optimizar aplicaciones complejas, aplica la técnica de **división de contextos (Context Splitting)** separando el estado de las callbacks actualizadoras.

En el próximo capítulo, aprenderemos a dominar la gestión de estados locales masivos y complejos implementando máquinas de estado estructuradas con el patrón Reducer nativo mediante **`useReducer`**.

---

[Capítulo anterior](05-hooks-y-custom-hooks.md) | [Inicio](README.md) | [Capítulo siguiente →](07-usereducer.md)
