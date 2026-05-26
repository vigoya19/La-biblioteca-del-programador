# Capítulo 14: Testing en React: Unitario, Integración y E2E

> "No escribes pruebas para demostrar que tu código funciona hoy; escribes pruebas para garantizar que tu código seguirá funcionando mañana cuando otra persona lo refactorice."

El desarrollo de software profesional en el frontend ha madurado significativamente. Atrás quedaron los días en los que probar una aplicación significaba hacer clics de forma manual en el navegador esperando que nada se rompiera. En la ingeniería de software actual, la automatización de pruebas es un requisito indispensable para garantizar la robustez, la estabilidad y la entrega continua de valor en producción.

En este capítulo, estudiaremos la filosofía moderna de pruebas, configuraremos un entorno de alta velocidad con **Vitest** y **React Testing Library (RTL)**, y aprenderemos a interceptar y simular llamadas de red reales utilizando **MSW (Mock Service Worker)**.

---

## 14.1 La Filosofía de Testing en el Frontend

Antes de escribir código de prueba, debemos comprender la regla de oro de **React Testing Library**: **"Cuanto más se parezcan tus pruebas a la forma en que se usa tu software, más confianza te darán."**

*   **Evita probar detalles de implementación**: No pruebes si una variable de estado local se llama `contador` o si un método interno tiene X nombre. Al usuario no le importa el código de fondo; solo le importa lo que ve e interactúa.
*   **Prueba el comportamiento**: Busca elementos en pantalla basándote en sus **roles de accesibilidad** (como `screen.getByRole('button', { name: /enviar/i })`) e interactúa con ellos simulando clics reales y eventos de teclado.

---

## 14.2 El Entorno Moderno: Vitest y React Testing Library

Durante años, Jest fue el estándar de la industria. Sin embargo, en proyectos modernos construidos con Vite, **Vitest** se ha consolidado como el sucesor indiscutible.

*   **Vitest**: Un framework de ejecución de pruebas ultrarrápido que comparte la misma configuración y pipeline de transformación de archivos de tu servidor Vite, eliminando archivos de configuración de compilación duplicados y pesados.
*   **React Testing Library**: Provee utilidades para montar componentes en un DOM virtualizado en memoria (`jsdom` o `happy-dom`) y realizar aserciones de comportamiento reales.

---

## 14.3 Mocking de Red con MSW (Mock Service Worker)

Cuando un componente hace peticiones HTTP a una API externa, **nunca debemos pegarle al servidor real en las pruebas unitarias o de integración**. Eso haría que las pruebas fueran lentas, frágiles ante caídas de internet y difíciles de reproducir en servidores de integración continua (CI/CD).

**MSW (Mock Service Worker)** es la herramienta estándar del ecosistema. En lugar de sobreescribir las funciones de fetch de JavaScript con mocks rudimentarios, MSW levanta un Service Worker en el navegador (o intercepta a nivel de red en Node.js), capturando las peticiones HTTP y devolviendo respuestas JSON mockeadas de forma transparente.

```
 [ React Component ] ─── Fetch: /api/usuarios ───► [ API Real (Servidor) ]
                             │                            ▲
                             ▼                            │ (Interceptado)
                       ┌───────────┐                      │
                       │ MSW Mock  ├──────────────────────┘
                       │ Server    │ (Devuelve JSON Mockeado sin tocar red)
                       └───────────┘
```

> [!NOTE]
> ### ✈️ El Simulador de Vuelo de Alta Fidelidad
> 
> Imagina que eres el director de ingeniería a cargo del entrenamiento de pilotos para un gigantesco avión comercial de pasajeros:
> 
> - Probar tu aplicación frontend pegándole directamente a la base de datos real del servidor en cada prueba unitaria es equivalente a **entrenar a tus pilotos en el avión físico real volando en directo sobre un huracán categoría 5**. Si el piloto comete un error, el avión se destruye, los costos de combustible son masivos y pones en riesgo vidas humanas.
> - **Vitest y React Testing Library** son equivalentes a construir una **Cabina de Simulador de Vuelo de Alta Fidelidad** en tierra firme: El piloto se sienta en una réplica exacta de la cabina física con pantallas y controles idénticos (los roles de accesibilidad del DOM).
> - **Mock Service Worker (MSW)** es el software de simulación del clima: Intercepta los controles de vuelo y proyecta tormentas, viento o fallos mecánicos simulados de forma idéntica en las pantallas (**respuestas JSON mockeadas**). Si el piloto se estrella, no hay daños físicos, no gastas combustible, y puedes repetir la prueba mil veces hasta garantizar que el piloto está listo para volar de forma segura en el mundo real.

---

## 14.4 Ejemplo Práctico: Testeando un Buscador Asíncrono con Vitest y MSW

Escribamos una prueba de integración para un componente que descarga datos de internet al hacer clic en un botón:

### `BuscadorUsuarios.test.tsx`
```typescript
import React from 'react';
import { describe, it, expect, beforeAll, afterEach, afterAll } from 'vitest';
import { render, screen, fireEvent } from '@testing-library/react';
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';
import { DetalleUsuario } from './04-ciclo-de-vida-y-efectos';

// 1. Configurar el servidor de MSW para interceptar llamadas de red
const mockUsuario = {
  id: 1,
  name: 'Andrés Vigoya',
  email: 'andres@correo.com',
};

const servidorMsw = setupServer(
  http.get('https://jsonplaceholder.typicode.com/users/1', () => {
    // Interceptar la llamada de red y retornar una respuesta mockeada al instante
    return HttpResponse.json(mockUsuario);
  })
);

// 2. Orquestar el ciclo de vida del servidor de simulación de red
beforeAll(() => servidorMsw.listen()); // Iniciar servidor antes de las pruebas
afterEach(() => servidorMsw.resetHandlers()); // Limpiar interceptores entre pruebas
afterAll(() => servidorMsw.close()); // Apagar servidor al finalizar todas las pruebas

describe('Componente DetalleUsuario (Integración con MSW)', () => {
  it('debe renderizar el estado de carga y posteriormente los datos del usuario validados', async () => {
    // Montar el componente en el DOM virtual de Vitest
    render(<DetalleUsuario usuarioId={1} />);

    // Aserción 1: Validar que se muestra el spinner de carga inicial
    const cargando = screen.getByText(/cargando datos del usuario/i);
    expect(cargando).toBeInTheDocument();

    // Aserción 2: Validar que los datos del usuario simulados por MSW se pintan en pantalla
    // Usamos findByRole (asíncrono) para esperar a que la promesa se resuelva
    const tituloUsuario = await screen.findByRole('heading', { name: /andrés vigoya/i });
    expect(tituloUsuario).toBeInTheDocument();

    const emailUsuario = screen.getByText(/email: andres@correo.com/i);
    expect(emailUsuario).toBeInTheDocument();
  });
});
```

---

## Resumen del Capítulo

*   La filosofía moderna de testing se enfoca en **probar el comportamiento e interactividad del componente** en lugar de testear detalles y nombres de variables internos.
*   **Vitest** es el ejecutor de pruebas de nueva generación para el ecosistema de Vite, logrando arranques y ejecuciones en milisegundos gracias al pipeline compartido.
*   **React Testing Library (RTL)** monta tus componentes en un DOM simulado y expone selectores basados en **roles de accesibilidad** para simular la experiencia del usuario.
*   **Mock Service Worker (MSW)** intercepta llamadas HTTP a nivel de red, actuando como un **simulador de clima de alta fidelidad** para proveer respuestas JSON estables sin tocar servidores reales.

En el próximo capítulo (el gran cierre del libro), uniremos cada pieza arquitectónica, patrón de diseño, optimización y prueba aprendida a lo largo del libro para construir un **Proyecto Práctico E-Commerce Premium** feature-driven listo para producción.

---

[Capítulo anterior](13-optimizacion-y-rendimiento.md) | [Inicio](README.md) | [Capítulo siguiente →](15-proyecto-practico.md)
