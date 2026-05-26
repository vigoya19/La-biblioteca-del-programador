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

---

## 13.1 ¿Qué Problema Resuelven Maven y Gradle?

Java no es solo `javac MiClase.java`. Un proyecto profesional necesita resolver dependencias transitivas, compilar módulos, ejecutar pruebas, generar artefactos, publicar librerías, empaquetar aplicaciones y reproducir el mismo resultado en la máquina de un desarrollador, en CI y en producción.

Sin una herramienta de construcción, cada equipo termina escribiendo scripts frágiles:

```bash
javac -cp "lib/*" src/**/*.java
java -cp "lib/*:src" com.empresa.Main
```

Esto funciona para un ejemplo pequeño, pero se rompe cuando aparecen pruebas, perfiles de entorno, versiones incompatibles o varios módulos. Maven y Gradle existen para convertir ese caos en un proceso repetible.

## 13.2 Maven: Convención y Estabilidad

Maven se basa en una filosofía clara: **convención sobre configuración**. Si sigues su estructura estándar, Maven sabe dónde está el código, dónde están los tests y cómo empaquetar el proyecto.

```text
mi-app/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/
    │   └── resources/
    └── test/
        ├── java/
        └── resources/
```

El archivo central es `pom.xml`:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.empresa</groupId>
    <artifactId>catalogo-api</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <junit.version>5.10.2</junit.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
            </plugin>
        </plugins>
    </build>
</project>
```

Comandos esenciales:

```bash
mvn clean              # elimina target/
mvn compile            # compila src/main/java
mvn test               # ejecuta pruebas unitarias
mvn package            # genera el JAR/WAR
mvn verify             # ejecuta validaciones adicionales
mvn install            # instala el artefacto en el repositorio local
```

El ciclo de vida de Maven está encadenado. Si ejecutas `mvn package`, Maven ejecuta antes las fases necesarias: validación, compilación y tests.

## 13.3 Gestión de Dependencias en Maven

Maven descarga dependencias desde repositorios remotos como Maven Central y las almacena en `~/.m2/repository`. Una dependencia puede traer otras dependencias transitivas:

```text
tu app -> spring-boot-starter-web -> spring-web -> spring-core
```

Para inspeccionar el árbol:

```bash
mvn dependency:tree
```

Para detectar conflictos de versiones, este comando es oro. Si dos librerías traen versiones distintas de Jackson, Netty o Guava, el árbol te muestra quién las introdujo.

En proyectos grandes conviene centralizar versiones:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.3.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

Esto evita que cada módulo invente sus propias versiones.

## 13.4 Gradle: Flexibilidad y Rendimiento

Gradle nació para resolver casos donde Maven se vuelve rígido. Usa un DSL en Groovy o Kotlin, permite tareas personalizadas y tiene un sistema avanzado de caché e incremental builds.

Un proyecto Gradle moderno suele usar Kotlin DSL:

```text
mi-app/
├── build.gradle.kts
├── settings.gradle.kts
└── src/
    ├── main/java/
    └── test/java/
```

Ejemplo de `build.gradle.kts`:

```kotlin
plugins {
    java
    application
}

group = "com.empresa"
version = "1.0.0"

repositories {
    mavenCentral()
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(21))
    }
}

dependencies {
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.2")
}

application {
    mainClass.set("com.empresa.Main")
}

tasks.test {
    useJUnitPlatform()
}
```

Comandos equivalentes:

```bash
./gradlew clean
./gradlew compileJava
./gradlew test
./gradlew build
./gradlew run
```

El wrapper (`gradlew`) es obligatorio en proyectos profesionales. Garantiza que todos usen la misma versión de Gradle sin instalarla globalmente:

```bash
gradle wrapper --gradle-version 8.8
```

## 13.5 Maven vs Gradle

| Criterio | Maven | Gradle |
|---|---|---|
| Filosofía | Convención fuerte | Flexibilidad programable |
| Configuración | XML declarativo | Kotlin/Groovy DSL |
| Curva de aprendizaje | Más simple al inicio | Más potente, más superficie |
| Rendimiento | Bueno y predecible | Excelente con caché e incremental builds |
| Ecosistema enterprise | Muy maduro | Muy usado en Android y monorepos |
| Personalización | Plugins y perfiles | Tareas y lógica de build |

Regla práctica:

- Usa Maven si quieres estandarización, simplicidad y mínima sorpresa.
- Usa Gradle si necesitas builds complejos, muchos módulos, generación de código o optimización fuerte de tiempos.

## 13.6 Tests, Cobertura y Calidad

Una build profesional no solo compila. También protege el código.

Con Maven puedes agregar JaCoCo:

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.12</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>verify</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Con Gradle:

```kotlin
plugins {
    jacoco
}

tasks.test {
    useJUnitPlatform()
    finalizedBy(tasks.jacocoTestReport)
}

tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports {
        xml.required.set(true)
        html.required.set(true)
    }
}
```

El objetivo no es perseguir un porcentaje mágico, sino evitar que cambios críticos entren sin pruebas.

## 13.7 Builds Reproducibles en CI/CD

En CI nunca dependas de herramientas instaladas "a mano". Usa Maven Wrapper o Gradle Wrapper:

```bash
./mvnw clean verify
./gradlew clean build
```

Un workflow mínimo para Maven:

```yaml
name: java-ci

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
          cache: maven
      - run: ./mvnw clean verify
```

Y para Gradle:

```yaml
name: java-ci

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
      - uses: gradle/actions/setup-gradle@v3
      - run: ./gradlew clean build
```

## 13.8 Buenas Prácticas

- Versiona siempre `pom.xml`, `build.gradle.kts`, `settings.gradle.kts` y los wrappers.
- No subas `target/`, `build/`, `.gradle/` ni artefactos generados.
- Usa Java toolchains para fijar la versión del JDK.
- Ejecuta tests en cada pull request.
- Revisa el árbol de dependencias cuando aparezcan conflictos.
- Evita lógica excesiva en el build; si crece demasiado, extrae plugins o scripts mantenibles.
- Usa perfiles o propiedades para diferencias de entorno, no cambios manuales.
- Publica artefactos con versionado claro: SemVer, SHA o número de release.

## Resumen del Capítulo

Maven y Gradle convierten un proyecto Java en una unidad construible, testeable y publicable. Maven destaca por su estabilidad y convenciones; Gradle por su flexibilidad y rendimiento. En ambos casos, la meta es la misma: que cualquier persona o pipeline pueda ejecutar un comando y obtener el mismo resultado.

Con esto cerramos el recorrido principal del libro: desde fundamentos del lenguaje hasta las herramientas necesarias para trabajar en proyectos Java reales.

---

[← Capítulo anterior](capitulo-12-testing.md) | [Inicio](../README.md)
