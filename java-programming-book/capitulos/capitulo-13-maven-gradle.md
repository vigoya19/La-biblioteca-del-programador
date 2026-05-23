# Capítulo 13: Herramientas de Construcción — Maven y Gradle

> "Primero resuelve el problema. Luego, escribe el código."
> — John Johnson

> "La automatización aplicada a una operación ineficiente aumentará la ineficiencia."
> — Bill Gates

> "No es el lenguaje de programación lo que define el éxito de un proyecto, sino la disciplina con la que se construye."
> — Robert C. Martin

---

Imagina que eres el arquitecto de una catedral gótica en plena Edad Media. Tienes cientos de obreros, toneladas de piedra y madera, planos detallados y un plazo imposible. Si cada cantero tuviera que recordar de memoria qué bloque cortar, de qué cantera traerlo y en qué orden colocarlo, la catedral colapsaría antes de llegar al segundo piso. Necesitas un maestro de obras: alguien que coordine las tareas, sepa qué depende de qué, y garantice que cada material llegue en el momento preciso.

En el desarrollo de software moderno, ese maestro de obras son las **herramientas de construcción** (*build tools*). Maven y Gradle son los dos capataces más respetados del ecosistema Java. Este capítulo te enseñará a dominarlos para que tus proyectos no colapsen bajo su propio peso.

> [!NOTE]
> 🏗️ **En resumen:** Una herramienta de construcción automatiza la compilación, el testing, el empaquetado y la gestión de dependencias. Sin ella, un proyecto Java mediano es un castillo de naipes; con ella, es una fortaleza.
