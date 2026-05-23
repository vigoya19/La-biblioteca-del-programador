# Capítulo 11: Spring Boot — El Framework de Desarrollo Java

> *"La simplicidad es la máxima sofisticación."* — Leonardo da Vinci

---

Spring Boot ha transformado la forma en que millones de desarrolladores Java construimos aplicaciones. Este capítulo te guiará desde los fundamentos del ecosistema Spring hasta la creación de APIs REST completas, pasando por acceso a datos con JPA, monitoreo en producción, testing automatizado y despliegue en contenedores Docker.

Aprenderás por qué Spring Boot es el framework más utilizado en el ecosistema Java empresarial y cómo su filosofía de "convención sobre configuración" te permite enfocarte en la lógica de negocio en lugar de perder horas configurando XML. No asumimos que conozcas Spring Framework: comenzamos desde cero y construimos cada concepto paso a paso, con ejemplos que puedes copiar, ejecutar y modificar.

Al finalizar este capítulo, serás capaz de diseñar, implementar, probar y desplegar aplicaciones Spring Boot con la confianza de un desarrollador profesional. Comencemos.

---

## 11.1 ¿Qué es Spring Boot?

Spring Boot no es un framework nuevo. Es una **capa de auto-configuración** construida sobre el ecosistema Spring Framework que elimina la complejidad de configurar manualmente cada componente. Mientras que Spring Framework te da todas las piezas para construir una aplicación, Spring Boot te da una aplicación ya ensamblada que puedes personalizar.

> [!NOTE]
> ### 🏗️ La Analogía del Kit de Construcción vs. La Casa Prefabricada
>
> Imagina que quieres construir una casa:
>
> - **Spring Framework** es como comprar un terreno y recibir un catálogo gigantesco con ladrillos, tuberías, cables eléctricos, ventanas, puertas y planos arquitectónicos. Tú tienes que decidir cuántos ladrillos pedir, cómo conectarlos, dónde pasar los cables y cómo unir las tuberías. Tienes control absoluto, pero también pasas semanas configurando antes de ver una sola pared en pie.
> - **Spring Boot** es como una empresa de casas prefabricadas que te entrega la casa ya construida sobre ruedas: las paredes están pintadas, los enchufes funcionan, el agua corre. Si quieres cambiar el color de la sala, abres el manual (`application.properties`) y eliges otro color. Si prefieres instalar un jacuzzi en lugar de una bañera, cambias la dependencia en el `pom.xml` y la casa se reconfigura sola.
>
> **En resumen:** Spring Boot te entrega una aplicación funcional desde el primer minuto. Tú decides qué personalizar y qué mantener con los valores predeterminados inteligentes que el equipo de Spring ha depurado durante años.

### 11.1.1 El Ecosistema Spring: Una Breve Historia

Para entender Spring Boot, debemos comprender el problema que resolvió.

En 2002, Rod Johnson publicó *Expert One-on-One J2EE Design and Development*, donde criticaba la complejidad de J2EE (el predecesor de Jakarta EE). Las aplicaciones empresariales requerían servidores de aplicaciones pesados (WebLogic, WebSphere), archivos XML interminables, interfaces `Home` y `Remote`, y un modelo de componentes que hacía que incluso un simple "Hola Mundo" requiriera decenas de archivos.

De ese libro nació Spring Framework en 2003 como una alternativa ligera basada en POJOs (Plain Old Java Objects). Sus pilares eran:

1. **Inversión de Control (IoC):** En lugar de que tus objetos creen sus dependencias, un contenedor las inyecta.
2. **Programación Orientada a Aspectos (AOP):** Separar preocupaciones transversales (transacciones, seguridad, logging) del código de negocio.
3. **Abstracciones sobre APIs de infraestructura:** Spring te daba `JdbcTemplate` para no tener que lidiar con `try-catch-finally` de JDBC.

El problema era que, aunque Spring Framework eliminaba la complejidad de J2EE, **introducía su propia complejidad de configuración**. Un proyecto típico en 2012 requería:

- Archivos `applicationContext.xml` de cientos de líneas.
- Configuración manual de `DispatcherServlet` en `web.xml`.
- Configuración manual del `EntityManagerFactory`, `DataSource`, `TransactionManager`.
- Gestión manual de dependencias entre beans.

Spring Boot nació en 2014 (con la versión 1.0) para resolver exactamente esto. Su mantra:

> **"Convención sobre configuración. Si el 80% de las aplicaciones usa la misma configuración, ¿por qué obligar a cada desarrollador a escribirla?"**

### 11.1.2 Convención sobre Configuración y Auto-Configuración

La **auto-configuración** es el mecanismo estrella de Spring Boot. Cuando agregas una dependencia a tu proyecto, Spring Boot detecta automáticamente qué beans necesitas crear y los configura con valores predeterminados sensatos.

Por ejemplo, si agregas `spring-boot-starter-data-jpa` a tu proyecto:

1. Spring Boot detecta que Hibernate está en el classpath.
2. Crea automáticamente un `DataSource` (si detecta H2, configura una base en memoria).
3. Crea un `EntityManagerFactory` vinculado a ese `DataSource`.
4. Crea un `TransactionManager` vinculado al `EntityManagerFactory`.
5. Configura `JpaRepositories` para que puedas inyectar repositorios sin escribir una línea de configuración.

Todo esto ocurre **sin que tú escribas una sola línea de XML o de configuración Java**. Por supuesto, puedes sobrescribir cualquier componente creando tu propio bean — la auto-configuración de Spring Boot siempre cede el paso a tu configuración explícita.

La anotación `@SpringBootApplication` que veremos en detalle más adelante habilita este mecanismo. Internamente, Spring Boot recorre una cadena de clases `XxxAutoConfiguration` (hay más de 200) que evalúan condiciones como "¿está tal clase en el classpath?", "¿existe ya un bean de este tipo?", "¿está definida cierta propiedad?" y deciden si configurar o no cada componente.

### 11.1.3 Servidores Embebidos

Un cambio revolucionario que introdujo Spring Boot fue el **servidor embebido**. Antes de Spring Boot, desplegar una aplicación web Java requería:

1. Compilar tu código en un archivo WAR.
2. Instalar y configurar un servidor de aplicaciones (Tomcat, JBoss, WebLogic).
3. Desplegar el WAR en el servidor.
4. Rezar para que los classloaders no entraran en conflicto.

Con Spring Boot, el servidor **está dentro de tu aplicación**. Tu aplicación es un JAR ejecutable que contiene a Tomcat (o Jetty, o Undertow) embebido. Ejecutas `java -jar mi-app.jar` y la aplicación arranca con su propio servidor HTTP.

| Servidor Embebido | Dependencia Starter | Características |
|---|---|---|
| **Tomcat** | `spring-boot-starter-web` (por defecto) | El más probado, soporte completo de Servlet API |
| **Jetty** | `spring-boot-starter-jetty` | Ligero, ideal para aplicaciones que no usan todas las APIs de Jakarta EE |
| **Undertow** | `spring-boot-starter-undertow` | Alto rendimiento, diseñado para escenarios de alta concurrencia |

Para cambiar de Tomcat a Jetty, basta con excluir Tomcat y agregar Jetty en tu `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

### 11.1.4 Spring Initializr

Spring Initializr (`https://start.spring.io`) es una herramienta web (también accesible desde IntelliJ IDEA, Eclipse y VS Code) que genera la estructura base de un proyecto Spring Boot. Seleccionas:

- **Tipo de proyecto:** Maven o Gradle.
- **Lenguaje:** Java, Kotlin o Groovy.
- **Versión de Spring Boot.**
- **Metadatos:** Group, Artifact, Name, Description, Package name.
- **Dependencias (Starters):** Web, JPA, Security, Actuator, Validation, etc.

Al hacer clic en "Generate", obtienes un archivo ZIP con una estructura de proyecto lista para importar en tu IDE y ejecutar.

Los **Starters** son dependencias agrupadas que siguen el patrón `spring-boot-starter-*`. Cada starter incluye todas las dependencias transitivas necesarias para una funcionalidad concreta, en versiones compatibles entre sí. Esto elimina el "infierno de dependencias" que atormentaba a los proyectos Spring tradicionales.

| Starter | Proporciona |
|---|---|
| `spring-boot-starter-web` | Spring MVC, Tomcat embebido, Jackson (JSON) |
| `spring-boot-starter-data-jpa` | Spring Data JPA, Hibernate, connection pooling (HikariCP) |
| `spring-boot-starter-validation` | Jakarta Bean Validation, Hibernate Validator |
| `spring-boot-starter-actuator` | Endpoints de monitoreo y métricas |
| `spring-boot-starter-security` | Spring Security, autenticación y autorización |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, Hamcrest, MockMvc |
| `spring-boot-starter-amqp` | Spring AMQP, RabbitMQ |
| `spring-boot-starter-data-redis` | Spring Data Redis, cliente Lettuce |

### 11.1.5 Estructura de un Proyecto Spring Boot

Un proyecto típico generado por Initializr tiene esta estructura:

```
miproyecto/
├── pom.xml                        (o build.gradle)
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/ejemplo/miproyecto/
│   │   │       ├── MiProyectoApplication.java    ← Clase principal
│   │   │       ├── controlador/
│   │   │       │   └── ProductoControlador.java
│   │   │       ├── servicio/
│   │   │       │   └── ProductoServicio.java
│   │   │       ├── repositorio/
│   │   │       │   └── ProductoRepositorio.java
│   │   │       ├── modelo/
│   │   │       │   └── Producto.java
│   │   │       └── configuracion/
│   │   │           └── ConfiguracionApp.java
│   │   └── resources/
│   │       ├── application.properties  (o .yml)
│   │       ├── static/                 ← CSS, JS, imágenes
│   │       └── templates/              ← Plantillas Thymeleaf, Freemarker
│   └── test/
│       └── java/
│           └── com/ejemplo/miproyecto/
│               └── MiProyectoApplicationTests.java
```

> **📖 Para principiantes**: Si vienes del mundo de los servlets tradicionales donde tenías que crear carpetas `WEB-INF`, `web.xml`, y archivos de configuración por cada componente, esta estructura puede parecerte "demasiado simple". Confía en ella: Spring Boot configura todo el `DispatcherServlet`, el escaneo de componentes y el servidor web por ti. El directorio `src/main/resources/static/` sirve automáticamente archivos estáticos (como `index.html`) sin que escribas una línea de código. El directorio `templates/` funciona con motores de plantillas como Thymeleaf si agregas su starter. Tu única responsabilidad es escribir clases Java dentro del paquete base (o subpaquetes) que contiene a `MiProyectoApplication`.

---

## 11.2 Primer Proyecto Spring Boot

En esta sección crearemos una aplicación Spring Boot desde cero, la ejecutaremos y analizaremos cada componente del código generado.

### 11.2.1 Creación del Proyecto

Tienes tres opciones para crear un proyecto Spring Boot:

1. **Spring Initializr web:** Ve a `https://start.spring.io`, selecciona Maven, Java, Spring Boot 3.3+ (compatible con Java 21), y agrega las dependencias `Spring Web` y `Spring Boot DevTools`. Descarga el ZIP.
2. **IntelliJ IDEA Ultimate:** File → New → Project → Spring Initializr.
3. **Línea de comandos:** Usa el CLI de Spring Boot o `curl` contra la API de Initializr.

Para este capítulo usaremos la estructura generada por Initializr con Maven. Nuestro `pom.xml` mínimo:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.0</version>
        <relativePath/>
    </parent>

    <groupId>com.biblioteca</groupId>
    <artifactId>demo-spring-boot</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>demo-spring-boot</name>
    <description>Demo del Capítulo 11</description>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <!-- Starter web: incluye Spring MVC, Tomcat embebido, Jackson -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Herramientas de desarrollo: reinicio automático, LiveReload -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>

        <!-- Pruebas -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

El `spring-boot-starter-parent` como POM padre es una de las genialidades de Spring Boot. Este POM proporciona:

- **Gestión de dependencias (dependency management):** Define las versiones compatibles de cientos de librerías. Tú agregas una dependencia sin especificar versión y Maven usa la definida por el parent.
- **Configuración de plugins:** Versiones de `maven-compiler-plugin` con soporte para Java 21, `maven-surefire-plugin` para tests, etc.
- **Filtrado de recursos:** Los placeholders `${...}` en `application.properties` se resuelven con propiedades de Maven.

### 11.2.2 La Clase Principal y @SpringBootApplication

El archivo más importante de tu proyecto es la clase principal. Veámosla en detalle:

```java
package com.biblioteca.demospringboot;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoSpringBootApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoSpringBootApplication.class, args);
    }
}
```

`@SpringBootApplication` es una anotación compuesta que equivale a declarar tres anotaciones simultáneamente:

```java
// Lo que @SpringBootApplication realmente contiene:
@SpringBootConfiguration   // (1) Indica que esta clase es una clase de configuración
@EnableAutoConfiguration   // (2) Activa la auto-configuración de Spring Boot
@ComponentScan             // (3) Escanea componentes en este paquete y subpaquetes
public @interface SpringBootApplication {
    // ...
}
```

**1. @SpringBootConfiguration** — Es una especialización de `@Configuration` (que a su vez es un estereotipo de `@Component`). Indica que esta clase puede contener métodos `@Bean` que definen beans en el contenedor. La clase principal puede (aunque no es obligatorio) albergar definiciones de beans.

**2. @EnableAutoConfiguration** — Es la anotación mágica. Cuando está presente, Spring Boot examina el classpath, detecta qué librerías están disponibles y aplica configuraciones automáticas. Internamente utiliza el mecanismo `spring.factories` (o `org.springframework.boot.autoconfigure.AutoConfiguration.imports` en Spring Boot 3.x) para cargar una lista de clases de auto-configuración. Cada clase de auto-configuración está anotada con `@ConditionalOnClass`, `@ConditionalOnMissingBean`, etc., para decidir si debe aplicarse.

```
spring-boot-autoconfigure.jar
└── META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
    ├── ...WebMvcAutoConfiguration
    ├── ...DataSourceAutoConfiguration
    ├── ...JpaRepositoriesAutoConfiguration
    ├── ...JacksonAutoConfiguration
    └── ... (más de 140 clases de auto-configuración)
```

**3. @ComponentScan** — Escanea recursivamente el paquete donde reside la clase anotada y todos sus subpaquetes en busca de clases anotadas con `@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`, `@Configuration`, etc. Cada clase encontrada se registra como un bean en el contenedor de IoC.

> **📖 Para principiantes**: ¿Por qué la clase principal debe estar en el paquete raíz? Porque `@ComponentScan` escanea desde el paquete de la clase anotada **hacia abajo**. Si colocas tu clase principal en `com.biblioteca.config` y tus controladores en `com.biblioteca.web`, los controladores **nunca serán encontrados** porque `web` no es un subpaquete de `config`. La convención es colocar la clase principal en el paquete base (`com.biblioteca.demospringboot`) y organizar el resto en subpaquetes.

#### El método main() y SpringApplication.run()

`SpringApplication.run()` es el punto de entrada que orquesta todo el arranque. Lo que hace internamente:

```
SpringApplication.run()
├── 1. Crea un ApplicationContext (tipo determinado por el classpath)
│       └── Servidor web → AnnotationConfigServletWebServerApplicationContext
│       └── Sin servidor → AnnotationConfigApplicationContext
├── 2. Registra un ApplicationListener para eventos de arranque
├── 3. Imprime el banner de Spring Boot
├── 4. Carga el ApplicationContext
│       ├── Escanea componentes (@ComponentScan)
│       ├── Aplica auto-configuración (@EnableAutoConfiguration)
│       ├── Registra beans definidos por el usuario (@Bean)
│       └── Carga propiedades externas
├── 5. Arranca el servidor web embebido (Tomcat por defecto)
└── 6. Publica ApplicationReadyEvent → ¡la aplicación está lista!
```

### 11.2.3 application.properties vs application.yml

Spring Boot busca automáticamente archivos `application.properties` o `application.yml` (o `application.yaml`) en `src/main/resources/`. Ambos formatos son válidos, pero YAML ofrece una sintaxis más legible para configuraciones jerárquicas.

**application.properties** (formato clave=valor):

```properties
# Servidor
server.port=9000
server.servlet.context-path=/api

# Base de datos
spring.datasource.url=jdbc:postgresql://localhost:5432/biblioteca
spring.datasource.username=admin
spring.datasource.password=secreto123
spring.jpa.hibernate.ddl-auto=validate

# Logging
logging.level.com.biblioteca=DEBUG
logging.file.name=logs/aplicacion.log
```

**application.yml** (formato YAML — el mismo contenido):

```yaml
server:
  port: 9000
  servlet:
    context-path: /api

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/biblioteca
    username: admin
    password: secreto123
  jpa:
    hibernate:
      ddl-auto: validate

logging:
  level:
    com.biblioteca: DEBUG
  file:
    name: logs/aplicacion.log
```

> [!NOTE]
> ### 🧅 La Analogía de la Cebolla de Configuración
>
> La configuración en Spring Boot funciona como las capas de una cebolla. Cada capa puede sobrescribir a la que está debajo:
>
> 1. **Capa exterior (más prioridad):** Argumentos de línea de comandos (`--server.port=9999`).
> 2. **Variables de entorno del SO:** `SERVER_PORT=9999` (Spring Boot convierte nombres como `SERVER_PORT` a `server.port` automáticamente).
> 3. **Archivos application-{profile}.yml:** Específicos por perfil (`application-produccion.yml`).
> 4. **Archivo application.yml:** Valores por defecto para todos los perfiles.
> 5. **Clases @ConfigurationProperties:** Valores tipados definidos en código Java.
> 6. **Capa interior (menos prioridad):** Valores por defecto de la auto-configuración (código Spring Boot).
>
> Esto significa que puedes arrancar la misma aplicación en desarrollo con `application.yml` apuntando a tu máquina local, y en producción con `--spring.datasource.url=jdbc:postgresql://prod-db:5432/biblioteca` sin cambiar el código ni el archivo de configuración.
>
> **En resumen:** La configuración externa fluye de afuera hacia adentro. Los valores más cercanos al entorno de ejecución tienen más prioridad.

### 11.2.4 Ejecutando la Aplicación y Entendiendo los Logs de Arranque

Para ejecutar la aplicación desde la línea de comandos:

```bash
./mvnw spring-boot:run
```

Si usas Gradle con el wrapper:

```bash
./gradlew bootRun
```

También puedes compilar y ejecutar el JAR directamente:

```bash
./mvnw clean package -DskipTests
java -jar target/demo-spring-boot-0.0.1-SNAPSHOT.jar
```

Los logs de arranque contienen información valiosa. Veamos un ejemplo comentado:

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/

 :: Spring Boot ::                (v3.3.0)

2024-05-15T10:30:45.123-03:00  INFO 12345 --- [main] c.b.d.DemoSpringBootApplication : Starting DemoSpringBootApplication...
2024-05-15T10:30:45.456-03:00  INFO 12345 --- [main] c.b.d.DemoSpringBootApplication : No active profile set, falling back to default: "default"
2024-05-15T10:30:46.789-03:00  INFO 12345 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized on port 8080
2024-05-15T10:30:47.012-03:00  INFO 12345 --- [main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2024-05-15T10:30:47.234-03:00  INFO 12345 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080
2024-05-15T10:30:47.345-03:00  INFO 12345 --- [main] c.b.d.DemoSpringBootApplication : Started DemoSpringBootApplication in 2.5 seconds
```

Puntos clave de los logs:

- **Número de proceso:** `12345` — útil para monitoreo.
- **Perfil activo:** Si no configuras `spring.profiles.active`, usa `default`.
- **Versión de Spring Boot:** `v3.3.0`.
- **Puerto de Tomcat:** `8080` por defecto. Cámbialo con `server.port=9090`.
- **Tiempo de arranque:** `2.5 seconds` — excelente métrica para detectar regresiones.

### 11.2.5 Nuestro Primer Endpoint REST

Para comprobar que todo funciona, creemos un endpoint simple:

```java
package com.biblioteca.demospringboot.controlador;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.time.LocalDateTime;
import java.util.Map;

@RestController
@RequestMapping("/api/v1")
public class SaludoControlador {

    @GetMapping("/saludo")
    public Map<String, Object> saludar() {
        return Map.of(
            "mensaje", "¡Hola desde Spring Boot!",
            "version", "3.3.0",
            "java", System.getProperty("java.version"),
            "timestamp", LocalDateTime.now().toString()
        );
    }
}
```

Accede a `http://localhost:8080/api/v1/saludo` y verás:

```json
{
    "mensaje": "¡Hola desde Spring Boot!",
    "version": "3.3.0",
    "java": "21.0.3",
    "timestamp": "2024-05-15T10:31:15.456"
}
```

Has creado tu primer endpoint REST con Spring Boot en menos de 10 líneas de código. Sin XML, sin `web.xml`, sin configurar manualmente Jackson. Así de sencillo.

---
