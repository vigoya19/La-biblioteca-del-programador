# Capítulo 2: Estrategias de Ramificación: GitFlow, Trunk-Based Development y GitLab Flow

> "El código que no está integrado en la rama principal no existe para el resto de la organización. Las ramas de larga duración son incubadoras de conflictos lógicos destructivos que ahogan la velocidad de entrega del equipo."

En el desarrollo de software colaborativo, controlar el flujo de cambios en el repositorio de Git es uno de los mayores desafíos de ingeniería. El código escrito por decenas de programadores debe converger de forma ordenada, ser validado y desplegarse en producción sin entorpecer el trabajo de los demás.

Sin embargo, elegir la estrategia de ramificación (*Branching Strategy*) equivocada puede paralizar a un equipo completo durante días en medio del infierno de resolución de conflictos de fusión (*Merge Hell*). En este capítulo, analizaremos comparativamente tres de las estrategias más influyentes en la industria: **GitFlow**, **Trunk-Based Development** y **GitLab Flow**, desmitificando el uso de **Feature Flags** y los modelos de promoción por entornos.

---

## 2.1 GitFlow: La Estrategia Tradicional de Ramas de Larga Duración

Diseñado por Vincent Driessen en 2010, **GitFlow** es un flujo de trabajo altamente estructurado e idóneo para proyectos de software con ciclos de lanzamiento de versiones planificados de forma rígida (por ejemplo, software embebido o apps móviles nativas).

```
                      [ master ]  (Producción - Código estable con tags)
                           ▲
                           │ (Release merge)
                      [ release/* ] (Ramas de estabilización y QA)
                           ▲
                           │ (Merge al completar sprint)
                      [ develop ] (Rama de integración principal)
                           ▲
                           │ (Feature merges)
               ┌───────────┴───────────┐
          [ feature/A ]           [ feature/B ] (Ramas efímeras de trabajo)
```

### Ramas Clave en GitFlow:
* **`master` / `main`**: Contiene únicamente código consolidado que corre en producción. Cada despliegue se etiqueta físicamente con un Tag de versión (ej. `v1.2.0`).
* **`develop`**: La rama de integración central. Todos los desarrolladores unen sus cambios aquí de forma regular.
* **`feature/*`**: Ramas independientes abiertas desde `develop` para desarrollar una característica específica. Se unen de vuelta a `develop` al finalizar la tarea.
* **`release/*`**: Ramas de estabilización creadas desde `develop` cuando se aproxima la fecha del sprint. Aquí solo se corrigen bugs de última hora detectados en QA antes de fusionar concurrentemente a `master` y `develop`.
* **`hotfix/*`**: Ramas de emergencia abiertas directamente desde `master` para solventar caídas críticas del servidor de producción, saltándose el ciclo regular de releases.

### Pros y Contras de GitFlow:
* **Pros**: Extremadamente ordenado, predecible y permite auditorías exhaustivas antes de la consolidación.
* **Contras**: Fomenta el aislamiento prolongado del código. Si una característica tarda 3 semanas en desarrollarse en su rama `feature/`, fusionarla de vuelta a `develop` disparará decenas de conflictos lógicos complejos. **Es el antipatrón de la Integración Continua (CI)**.

---

## 2.2 Trunk-Based Development: Velocidad a Escala (Continuous Integration Real)

**Trunk-Based Development (TBD)** es la estrategia preferida por las organizaciones de alto rendimiento tecnológico (como Google, Meta y Netflix). 

En lugar de trabajar en ramas aisladas durante semanas, todos los desarrolladores trabajan sobre una **única rama central llamada `trunk` (o `main`)** utilizando ramas de características extremadamente efímeras (que viven apenas unas horas o un par de días como máximo) unidas de forma agresiva de vuelta al `trunk`.

```
                    [ trunk / main ] (Despliegues continuos bajo demanda)
                       ▲   ▲   ▲
                       │   │   │  (Fusiones ultra-rápidas a las pocas horas)
                       │   ├───┼─── [ feature/C ] (Ramas de 1 día de vida)
                       ├───┘   └─── [ feature/D ]
                       │
```

### El Pilar Técnico: Feature Flags (Banderas de Característica)
Si un desarrollador está construyendo una funcionalidad compleja que requiere 2 semanas de trabajo, ¿cómo puede fusionar su código a `main` todos los días sin romper la producción para los usuarios concurrentes?

La solución son las **Feature Flags**:
* El código incompleto se envuelve en una condicional dinámica controlada por base de datos o APIs de configuración (como LaunchDarkly):
  ```typescript
  if (featureFlags.isEnabled('NUEVO_MOTOR_PAGOS_2026')) {
    ejecutarNuevoFlujoDePagos();
  } else {
    ejecutarFlujoTradicional();
  }
  ```
* El código incompleto viaja seguro a producción de forma diaria detrás de la Feature Flag desactivada. Esto permite:
  * Validar la integración del código físicamente de forma continua en producción (**CI Real**).
  * Realizar pruebas en caliente activando la flag exclusivamente para usuarios internos o betatesters (**Canary Testing**).
  * Encender la característica de golpe al 100% de usuarios desde un panel administrativo sin redesplegar código.

### Pros y Contras de Trunk-Based:
* **Pros**: Erradica el "Merge Hell" por completo, acelera la velocidad DORA del equipo al mínimo absoluto y fomenta la Integración Continua pura.
* **Contras**: Requiere de una disciplina de ingeniería excelsa, cobertura masiva de pruebas unitarias automatizadas y control sofisticado de Feature Flags en el backend para evitar deudas técnicas de flags obsoletas.

---

## 2.3 GitLab Flow: Promoción por Ramas de Entorno

**GitLab Flow** es una estrategia intermedia y pragmática que simplifica la estructura de GitFlow adaptándola a la realidad del desarrollo en la nube basado en contenedores. 

En lugar de forzar ramas complejas como `develop` o `release/`, GitLab Flow se basa en **promocionar código a través de ramas dedicadas que representan entornos físicos**:

```
                 [ feature/F ] (Desarrollador)
                      │
                      ▼
                 [ main / master ] ───► Despliegue automático a [ Staging ]
                      │
                      ▼ (Merge Request / Promoción)
                 [ pre-production ] ───► Despliegue automático a [ UAT ]
                      │
                      ▼ (Merge Request / Promoción)
                 [ production ] ───► Despliegue automático a [ Production ]
```

### Mecánica del Flujo:
1. El desarrollador trabaja en una rama de característica efímera y crea un Merge Request hacia **`main`**.
2. Al fusionar a `main`, el pipeline de CI/CD despliega de forma automática el contenedor en el entorno de **Staging**.
3. Cuando el equipo de control de calidad aprueba las pruebas en Staging, se realiza un Merge Request desde `main` hacia **`pre-production`** (desplegándose en el entorno de preproducción o UAT).
4. Finalmente, se promociona de `pre-production` hacia **`production`** mediante Merge Request para lanzar los cambios a producción.
* **Resultado**: Un rastreo visual impecable en Git de qué versión exacta de commit reside físicamente en cada servidor en caliente.

---

> [!NOTE]
> ### 🛤️ Las Agujas del Ferrocarril y las Vías de Alta Velocidad
> 
> Entendamos la diferencia conceptual entre las estrategias de Git utilizando una analogía física ferroviaria:
> 
> - **El Enfoque GitFlow (La Estación de Trenes Antigua)**:
>   - Imagina una estación de trenes con decenas de andenes laterales y vías de estacionamiento temporal (**Ramas `feature/`, `develop`, `release/`**).
>   - El tren de carga de carbón (tu código) sale de la mina. En lugar de ir directo al puerto, se desvía a una vía muerta lateral a esperar que el inspector revise los vagones. Luego se mueve a otra vía de acoplamiento para engancharse con otros trenes. 
>   - Este proceso de coordinar desvíos lógicos y esperar en andenes secundarios tarda días completos (**unión lenta de ramas y estabilización release**). 
>   - Si dos trenes gigantes intentan entrar a la misma vía de acoplamiento al mismo tiempo, chocarán de frente y bloquearán la estación entera por días (**Merge Hell**).
> 
> - **El Enfoque Trunk-Based (La Vía de Alta Velocidad con Desvíos Automatizados)**:
>   - Decides demoler todos los andenes de espera laterales. Construyes **una sola vía principal de alta velocidad, directa y sin curvas** que conecta la mina con el puerto de producción (**El `trunk` o `main`**).
>   - El tren nunca se detiene. Viaja a 300 km/h de forma continua.
>   - ¿Cómo evitas los choques de trenes? Utilizas sensores y desvíos inteligentes controlados por computadora (**Feature Flags y Testing Automatizado**). 
>   - Si un vagón lleva mercancía incompleta (código a medio desarrollar), viaja en el tren principal pero se desvía de forma física invisible a un almacén de carga sellado antes de llegar al andén de pasajeros gracias a las Feature Flags. El flujo del tren principal de alta velocidad nunca sufre interrupciones.
> 
> - **El Enfoque GitLab Flow (Los Puentes Levadizos de Entorno)**:
>   - Tienes la vía principal, pero el tren de carga debe cruzar **tres islas fortificadas consecutivas** antes de llegar al continente de producción: *Isla Staging (Entorno 1)*, *Isla Pre-Producción (Entorno 2)*, e *Isla Producción (Entorno 3)*.
>   - Entre cada isla hay un gran puente levadizo resguardado por guardias militares (**Los Merge Requests de Promoción**).
>   - El tren cruza la primera isla. Para cruzar a la segunda, los guardias deben bajar el puente levadizo tras confirmar que la carga no tiene explosivos o contrabando. Cruzas isla por isla de forma estructurada y con total control fronterizo visual de dónde está parado el tren.

---

## Resumen del Capítulo

* **GitFlow** es ideal para lanzamientos de software rígidos y tradicionales mediante un andamiaje de ramas dedicado, a costa de aislar el código provocando "Merge Hell" e interrumpiendo la CI.
* **Trunk-Based Development** simplifica el repositorio a una sola rama central activa (`trunk`) apoyándose en **Feature Flags** para desplegar código incompleto de forma segura en producción de forma diaria.
* **GitLab Flow** rutea y promociona la entrega de código a través de Git mediante ramas que representan entornos físicos (`staging`, `production`) controlados por Merge Requests.
* Una excelente branch strategy reduce la contención de desarrollo y acelera las métricas DORA del negocio de forma espectacular.

En el próximo capítulo, ingresaremos al diseño del motor de automatización mediante el estudio de **GitHub Actions Internals y la Sintaxis de Workflows** en producción.

---

[← Capítulo anterior (Capítulo 1)](01-fundamentos-devops.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 3) →](03-github-actions-workflows.md)
