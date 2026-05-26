# Capítulo 6: Entrada/Salida (I/O) y NIO en Java

## 6.1 Introducción

Todo programa útil necesita interactuar con el mundo exterior: leer datos de un archivo, escribir resultados, comunicarse por red o persistir objetos. Java proporciona un modelo unificado para estas operaciones mediante **streams** (flujos de datos).

Un **stream** es una abstracción que representa una secuencia ordenada de datos. Puede ser una fuente de datos (stream de entrada) o un destino de datos (stream de salida). La gran ventaja de este modelo es que, independientemente de si leemos de un archivo, un socket de red o un array en memoria, el código que procesa los datos es esencialmente el mismo.

### 6.1.1 Streams de bytes vs. streams de caracteres

Java distingue dos grandes familias de streams:

|                 | **Streams de bytes**           | **Streams de caracteres**     |
|-----------------|-------------------------------|-------------------------------|
| **Clases base** | `InputStream` / `OutputStream` | `Reader` / `Writer`           |
| **Unidad de dato** | `byte` (8 bits)            | `char` (16 bits Unicode)      |
| **Uso típico**  | Datos binarios (imágenes, audio, archivos ejecutables, objetos serializados) | Texto (archivos .txt, .csv, .json, .xml, código fuente) |
| **Paquete**     | `java.io`                     | `java.io`                     |

La diferencia es fundamental: un stream de caracteres se ocupa automáticamente de la codificación de caracteres (charset), traduciendo entre bytes y caracteres Unicode según el encoding especificado (UTF-8, ISO-8859-1, etc.). Los streams de bytes trabajan con bytes crudos, sin interpretación.

### 6.1.2 Visión general del paquete `java.io`

El paquete `java.io` contiene más de 80 clases e interfaces organizadas en una arquitectura decoradora: se envuelven streams básicos con streams más especializados para añadir funcionalidad (buffering, filtrado, conversión de tipos). Los cuatro pilares son:

- `InputStream` / `OutputStream` — para datos binarios.
- `Reader` / `Writer` — para datos de texto.
- Clases puente: `InputStreamReader` / `OutputStreamWriter` conectan ambos mundos.
- `File`, `RandomAccessFile`, `Serializable`, etc.

---

## 6.2 Streams de bytes

### 6.2.1 `InputStream` y `OutputStream`

Son las clases abstractas raíz de todos los streams de bytes.

**`InputStream`** — métodos principales:

```java
public abstract class InputStream implements Closeable {
    public abstract int read() throws IOException;          // Lee un byte (0-255), retorna -1 al final
    public int read(byte[] b) throws IOException;           // Lee hasta b.length bytes
    public int read(byte[] b, int off, int len) throws IOException;
    public void close() throws IOException;                 // Cierra el stream (hereda de Closeable)
    public long skip(long n) throws IOException;            // Salta n bytes
    public int available() throws IOException;              // Bytes disponibles sin bloquear
}
```

**`OutputStream`** — métodos principales:

```java
public abstract class OutputStream implements Closeable, Flushable {
    public abstract void write(int b) throws IOException;   // Escribe un byte
    public void write(byte[] b) throws IOException;         // Escribe todo el array
    public void write(byte[] b, int off, int len) throws IOException;
    public void flush() throws IOException;                 // Fuerza escritura de buffers
    public void close() throws IOException;                 // Cierra el stream
}
```

### 6.2.2 `FileInputStream` y `FileOutputStream`

Permiten leer y escribir bytes directamente en archivos. **Siempre deben usarse con try-with-resources** para garantizar el cierre automático.

**Lectura de un archivo byte a byte:**

```java
import java.io.FileInputStream;
import java.io.IOException;

public class LecturaBytes {
    public static void main(String[] args) {
        // try-with-resources: cierra el stream automáticamente
        try (FileInputStream fis = new FileInputStream("entrada.bin")) {
            int byteLeido;
            while ((byteLeido = fis.read()) != -1) {
                System.out.printf("%02X ", byteLeido);  // Imprime en hexadecimal
            }
        } catch (IOException e) {
            System.err.println("Error al leer el archivo: " + e.getMessage());
        }
    }
}
```

**Escritura de bytes a un archivo:**

```java
import java.io.FileOutputStream;
import java.io.IOException;

public class EscrituraBytes {
    public static void main(String[] args) {
        byte[] datos = { 0x48, 0x6F, 0x6C, 0x61 };  // "Hola" en ASCII

        try (FileOutputStream fos = new FileOutputStream("salida.bin")) {
            fos.write(datos);
            System.out.println("Archivo escrito correctamente.");
        } catch (IOException e) {
            System.err.println("Error al escribir: " + e.getMessage());
        }
    }
}
```

> **Nota:** `FileOutputStream` sobrescribe el archivo por defecto. Para añadir al final (append), usa el constructor `new FileOutputStream("archivo", true)`.

### 6.2.3 `BufferedInputStream` y `BufferedOutputStream`

Leer/escribir byte a byte desde disco es extremadamente ineficiente (cada operación implica una llamada al sistema operativo). El buffering soluciona esto leyendo o escribiendo bloques grandes en memoria y sirviendo los datos desde ahí.

```java
import java.io.*;

public class CopiaConBuffer {
    public static void main(String[] args) {
        long inicio = System.currentTimeMillis();

        try (BufferedInputStream  bis = new BufferedInputStream(
                 new FileInputStream("origen-grande.bin"));
             BufferedOutputStream bos = new BufferedOutputStream(
                 new FileOutputStream("destino.bin"))) {

            byte[] buffer = new byte[8192]; // 8 KB
            int bytesLeidos;
            while ((bytesLeidos = bis.read(buffer)) != -1) {
                bos.write(buffer, 0, bytesLeidos);
            }
        } catch (IOException e) {
            System.err.println("Error: " + e.getMessage());
        }

        long fin = System.currentTimeMillis();
        System.out.println("Copia completada en " + (fin - inicio) + " ms");
    }
}
```

Comparativa de rendimiento típica al copiar un archivo de 100 MB:

| Método | Tiempo aproximado |
|--------|-------------------|
| `FileInputStream.read()` byte a byte | ~15 segundos |
| `FileInputStream.read(byte[])` con buffer de 8 KB sin BufferedStream | ~200 ms |
| `BufferedInputStream` + `BufferedOutputStream` con buffer de 8 KB | ~150 ms |

### 6.2.4 `DataInputStream` y `DataOutputStream`

Permiten leer y escribir tipos primitivos de Java (`int`, `double`, `boolean`, `String` con `writeUTF`, etc.) en un formato binario portable.

```java
import java.io.*;

public class DatosPrimitivos {
    public static void main(String[] args) {
        // Escritura
        try (DataOutputStream dos = new DataOutputStream(
                 new FileOutputStream("datos.bin"))) {
            dos.writeInt(42);
            dos.writeDouble(3.1416);
            dos.writeBoolean(true);
            dos.writeUTF("Hola, mundo");
        } catch (IOException e) {
            e.printStackTrace();
        }

        // Lectura (mismo orden)
        try (DataInputStream dis = new DataInputStream(
                 new FileInputStream("datos.bin"))) {
            int entero     = dis.readInt();
            double doble   = dis.readDouble();
            boolean bool   = dis.readBoolean();
            String texto   = dis.readUTF();

            System.out.printf("int=%d, double=%.4f, boolean=%b, String=%s%n",
                              entero, doble, bool, texto);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

> **Importante:** el orden de lectura debe coincidir exactamente con el orden de escritura. Si no, los datos leídos serán incorrectos o se lanzará `EOFException`.

### 6.2.5 `ByteArrayInputStream` y `ByteArrayOutputStream`

Trabajan con arrays de bytes en memoria en lugar de archivos. Son muy útiles para pruebas unitarias o para procesar datos que ya están en memoria.

```java
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;

public class ByteArrayDemo {
    public static void main(String[] args) {
        // Escritura en memoria
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        baos.write(65);   // 'A'
        baos.write(66);   // 'B'
        baos.write(67);   // 'C'

        byte[] datos = baos.toByteArray();
        System.out.println("Bytes escritos: " + datos.length);

        // Lectura desde memoria
        try (ByteArrayInputStream bais = new ByteArrayInputStream(datos)) {
            int b;
            while ((b = bais.read()) != -1) {
                System.out.print((char) b + " ");  // A B C
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 6.2.6 Ejemplo práctico: copiar un archivo (comparativa de métodos)

El siguiente programa copia un archivo usando tres estrategias distintas y mide el tiempo de cada una:

```java
import java.io.*;

public class CopiaArchivoComparativa {

    public static void copiarByteAByte(String origen, String destino) throws IOException {
        try (FileInputStream  fis = new FileInputStream(origen);
             FileOutputStream fos = new FileOutputStream(destino)) {
            int b;
            while ((b = fis.read()) != -1) {
                fos.write(b);
            }
        }
    }

    public static void copiarConBuffer(String origen, String destino) throws IOException {
        try (FileInputStream  fis = new FileInputStream(origen);
             FileOutputStream fos = new FileOutputStream(destino)) {
            byte[] buffer = new byte[8192];
            int leidos;
            while ((leidos = fis.read(buffer)) != -1) {
                fos.write(buffer, 0, leidos);
            }
        }
    }

    public static void copiarBufferedStreams(String origen, String destino) throws IOException {
        try (BufferedInputStream  bis = new BufferedInputStream(
                 new FileInputStream(origen));
             BufferedOutputStream bos = new BufferedOutputStream(
                 new FileOutputStream(destino))) {
            byte[] buffer = new byte[8192];
            int leidos;
            while ((leidos = bis.read(buffer)) != -1) {
                bos.write(buffer, 0, leidos);
            }
        }
    }

    public static void main(String[] args) throws IOException {
        String origen  = "archivo-grande.bin";
        String destino = "copia.bin";

        long t0 = System.currentTimeMillis();
        copiarByteAByte(origen, destino + ".byte");
        System.out.println("Byte a byte:      " + (System.currentTimeMillis() - t0) + " ms");

        t0 = System.currentTimeMillis();
        copiarConBuffer(origen, destino + ".buf");
        System.out.println("Con buffer manual: " + (System.currentTimeMillis() - t0) + " ms");

        t0 = System.currentTimeMillis();
        copiarBufferedStreams(origen, destino + ".buffered");
        System.out.println("Buffered streams:  " + (System.currentTimeMillis() - t0) + " ms");
    }
}
```

---

## 6.3 Streams de caracteres

### 6.3.1 `Reader` y `Writer`

Son las clases abstractas raíz de todos los streams de caracteres. Su API es análoga a `InputStream`/`OutputStream` pero trabajan con `char` e `int` (que representa un carácter Unicode) en lugar de `byte`.

```java
public abstract class Reader implements Readable, Closeable {
    public int read() throws IOException;                // Lee un carácter (0-65535), -1 al final
    public int read(char[] cbuf) throws IOException;
    public abstract int read(char[] cbuf, int off, int len) throws IOException;
    public abstract void close() throws IOException;
}

public abstract class Writer implements Appendable, Closeable, Flushable {
    public void write(int c) throws IOException;         // Escribe un carácter
    public void write(char[] cbuf) throws IOException;
    public abstract void write(char[] cbuf, int off, int len) throws IOException;
    public void write(String str) throws IOException;
    public void write(String str, int off, int len) throws IOException;
    public abstract void flush() throws IOException;
    public abstract void close() throws IOException;
}
```

### 6.3.2 `FileReader` y `FileWriter`

Son las implementaciones más simples para leer y escribir archivos de texto. **No incluyen buffering**, por lo que se recomienda envolverlos con `BufferedReader`/`BufferedWriter`.

```java
import java.io.FileWriter;
import java.io.IOException;

public class EscrituraTexto {
    public static void main(String[] args) {
        String contenido = """
                Primera línea del archivo.
                Segunda línea.
                Tercera línea.
                """;

        try (FileWriter fw = new FileWriter("texto.txt")) {
            fw.write(contenido);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 6.3.3 `BufferedReader` y `BufferedWriter`

Añaden buffering y los métodos clave `readLine()` y `newLine()`.

```java
import java.io.*;

public class LecturaLineaPorLinea {
    public static void main(String[] args) {
        try (BufferedReader br = new BufferedReader(
                 new FileReader("texto.txt"))) {
            String linea;
            int numeroLinea = 0;
            while ((linea = br.readLine()) != null) {
                numeroLinea++;
                System.out.printf("%3d: %s%n", numeroLinea, linea);
            }
        } catch (IOException e) {
            System.err.println("Error: " + e.getMessage());
        }
    }
}
```

**Escritura con `BufferedWriter`:**

```java
import java.io.*;

public class EscrituraConBuffer {
    public static void main(String[] args) {
        String[] nombres = { "Alice", "Bob", "Charlie", "Diana" };

        try (BufferedWriter bw = new BufferedWriter(
                 new FileWriter("nombres.txt"))) {
            for (String nombre : nombres) {
                bw.write(nombre);
                bw.newLine();  // Independiente de la plataforma (\r\n en Windows, \n en Unix)
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 6.3.4 `InputStreamReader` y `OutputStreamWriter`

Son el **puente** entre el mundo de bytes y el de caracteres. Convierten bytes a caracteres (y viceversa) usando un charset especificado.

```java
import java.io.*;
import java.nio.charset.StandardCharsets;

public class PuenteBytesCaracteres {
    public static void main(String[] args) {
        // Leer un archivo UTF-8
        try (BufferedReader br = new BufferedReader(
                 new InputStreamReader(
                     new FileInputStream("documento.txt"),
                     StandardCharsets.UTF_8))) {
            String linea;
            while ((linea = br.readLine()) != null) {
                System.out.println(linea);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

        // Escribir en ISO-8859-1
        try (BufferedWriter bw = new BufferedWriter(
                 new OutputStreamWriter(
                     new FileOutputStream("salida-latin1.txt"),
                     StandardCharsets.ISO_8859_1))) {
            bw.write("Texto con codificación Latin-1: áéíóúñ");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

> **`StandardCharsets`** (desde Java 7) ofrece constantes para los charsets más comunes: `UTF_8`, `ISO_8859_1`, `US_ASCII`, `UTF_16`, etc. Evita usar strings como `"UTF-8"` que pueden causar `UnsupportedEncodingException`.

### 6.3.5 `PrintWriter`

Ofrece métodos familiares como `print()`, `println()` y `printf()` para escribir texto formateado. Es muy usado para escribir archivos de salida con formato legible.

```java
import java.io.*;

public class PrintWriterDemo {
    public static void main(String[] args) {
        try (PrintWriter pw = new PrintWriter(
                 new FileWriter("reporte.txt"))) {

            pw.println("=== Reporte de Ventas ===");
            pw.println();

            pw.printf("%-20s %10s %10s%n", "Producto", "Unidades", "Total");
            pw.println("-".repeat(42));

            pw.printf("%-20s %10d %10.2f%n", "Laptop", 5, 5999.95);
            pw.printf("%-20s %10d %10.2f%n", "Monitor", 12, 2399.50);
            pw.printf("%-20s %10d %10.2f%n", "Teclado", 30, 449.00);

            pw.println();
            pw.printf("Total general: $%.2f%n", 5*5999.95 + 12*2399.50 + 30*449.00);

            // Si ocurre un error, checkError() lo indica
            if (pw.checkError()) {
                System.err.println("Error durante la escritura.");
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

> **`PrintWriter` no lanza `IOException`** en sus métodos `print`/`println`/`printf`. En su lugar, debes verificar `checkError()`. Si necesitas manejo explícito de excepciones, envuélvelo en un bloque try-catch sobre el `FileWriter` subyacente.

### 6.3.6 Ejemplo práctico: leer un archivo de texto línea por línea y procesarlo

```java
import java.io.*;
import java.util.*;

public class ProcesarCSV {
    public static void main(String[] args) {
        List<String[]> registros = new ArrayList<>();

        try (BufferedReader br = new BufferedReader(
                 new FileReader("datos.csv"))) {

            String linea;
            while ((linea = br.readLine()) != null) {
                // Ignorar líneas vacías o comentarios
                if (linea.isBlank() || linea.startsWith("#")) {
                    continue;
                }
                String[] campos = linea.split(",");
                registros.add(campos);
            }
        } catch (IOException e) {
            System.err.println("Error al leer el archivo CSV: " + e.getMessage());
            return;
        }

        System.out.println("Registros leídos: " + registros.size());
        for (String[] registro : registros) {
            System.out.println(Arrays.toString(registro));
        }
    }
}
```

---


## 6.4 Codificación de caracteres en profundidad

### 6.4.1 ¿Qué es un charset?

Un **charset** (character set + encoding) es un mapeo entre caracteres abstractos y secuencias de bytes. Sin un charset correcto, los bytes no significan nada:

```
┌──────────────────────────────────────────────────────────────────┐
│            EL PROBLEMA DE LA CODIFICACIÓN                        │
│                                                                   │
│   Bytes:       [C3] [B1]                                         │
│                                                                   │
│   En UTF-8:    'ñ' (U+00F1)     ◀── correcto                     │
│   En ISO-8859-1: 'Ã' + '±'      ◀── mojibake (basura)           │
│   En CP-1252:  'Ã' + '±'       ◀── diferente basura             │
│                                                                   │
│   Conclusión: SIEMPRE especifica el charset explícitamente       │
└──────────────────────────────────────────────────────────────────┘
```

### 6.4.2 Unicode: el estándar universal

Unicode asigna un **code point** (número único) a cada carácter de todos los sistemas de escritura del mundo:

| Carácter | Code Point | Nombre |
|----------|-----------|--------|
| `A` | U+0041 | LATIN CAPITAL LETTER A |
| `ñ` | U+00F1 | LATIN SMALL LETTER N WITH TILDE |
| `€` | U+20AC | EURO SIGN |
| `汉` | U+6C49 | CJK UNIFIED IDEOGRAPH-6C49 |
| `😀` | U+1F600 | GRINNING FACE |
| `𐍈` | U+10348 | GOTHIC LETTER HWAIR |

> Unicode tiene espacio para 1,114,112 code points (U+0000 a U+10FFFF), organizados en 17 planos de 65,536 caracteres cada uno.

```
┌─────────────────────────────────────────────────────────────────┐
│   PLANOS DE UNICODE                                             │
│                                                                  │
│   Plano 0: BMP (Basic Multilingual Plane)  U+0000..U+FFFF       │
│   ┌──────────────────────────────────────────────────────┐     │
│   │ ASCII | Latin | Griego | Cirílico | Árabe | Chino... │     │
│   └──────────────────────────────────────────────────────┘     │
│        ↑ La mayoría de caracteres comunes están aquí             │
│                                                                  │
│   Planos 1-16: SMP (Supplementary Multilingual Plane)            │
│   ┌──────────────────────────────────────────────────────┐     │
│   │ Emojis 😀🎉🚀 | Música ♫ | Matemáticas 𝕏 | Gótico 𐍈 │     │
│   └──────────────────────────────────────────────────────┘     │
│        ↑ Caracteres que requieren surrogate pairs en Java       │
└─────────────────────────────────────────────────────────────────┘
```

### 6.4.3 UTF-8: la codificación dominante

UTF-8 codifica cada code point en 1 a 4 bytes, según su valor. Es compatible hacia atrás con ASCII (los primeros 127 caracteres se codifican en 1 byte).

```
┌──────────────────────────────────────────────────────────────────┐
│   CODIFICACIÓN UTF-8                                              │
│                                                                   │
│   Rango Unicode        Bytes   Formato binario                    │
│   ─────────────────    ─────   ─────────────────────────────     │
│   U+0000..U+007F       1       0xxxxxxx                          │
│   U+0080..U+07FF       2       110xxxxx 10xxxxxx                 │
│   U+0800..U+FFFF       3       1110xxxx 10xxxxxx 10xxxxxx        │
│   U+10000..U+10FFFF    4       11110xxx 10xxxxxx 10xxxxxx 10xxxxxx│
│                                                                   │
│   Ejemplos:                                                       │
│   'A'   U+0041 → [41]                     (1 byte)               │
│   'ñ'   U+00F1 → [C3 B1]                  (2 bytes)              │
│   '€'   U+20AC → [E2 82 AC]               (3 bytes)              │
│   '😀'  U+1F600 → [F0 9F 98 80]           (4 bytes)              │
└──────────────────────────────────────────────────────────────────┘
```

**Demostración práctica en Java:**

```java
import java.nio.charset.StandardCharsets;

public class DemoUTF8 {
    public static void main(String[] args) {
        String texto = "Añ€😀";
        System.out.println("Texto: " + texto);
        System.out.println("Longitud en chars: " + texto.length());

        byte[] utf8Bytes = texto.getBytes(StandardCharsets.UTF_8);
        System.out.println("Bytes en UTF-8 (" + utf8Bytes.length + "):");
        for (byte b : utf8Bytes) {
            System.out.printf("%02X ", b);
        }
        System.out.println();

        System.out.println("Code points:");
        texto.codePoints().forEach(cp ->
            System.out.printf("  U+%04X %s%n", cp, Character.getName(cp))
        );
    }
}
```

**Salida:**
```
Texto: Añ€😀
Longitud en chars: 5        ← ¡sorpresa! el emoji ocupa 2 chars
Bytes en UTF-8 (10):
41 C3 B1 E2 82 AC F0 9F 98 80
Code points:
  U+0041 LATIN CAPITAL LETTER A
  U+00F1 LATIN SMALL LETTER N WITH TILDE
  U+20AC EURO SIGN
  U+1F600 GRINNING FACE
```

### 6.4.4 UTF-16: la representación interna de Java

Java representa **internamente** todas las cadenas en UTF-16. Cada `char` son 16 bits. Pero esto crea un problema: los code points por encima de U+FFFF no caben en un solo `char` (16 bits = máx. 65535 = U+FFFF). La solución son los **surrogate pairs**.

### 6.4.5 Surrogate Pairs: cuando un carácter ocupa 2 `char`

Para representar code points > U+FFFF, UTF-16 usa **dos chars** consecutivos:

```
┌──────────────────────────────────────────────────────────────────┐
│   SURROGATE PAIRS EN JAVA (UTF-16)                                │
│                                                                   │
│   Code point:  U+1F600  😀  (GRINNING FACE)                      │
│                                                                   │
│   En UTF-16 se convierte en:                                      │
│   ┌──────────────┬──────────────┐                                │
│   │ High Surrogate│ Low Surrogate │                               │
│   │   U+D83D      │   U+DE00      │                               │
│   │   0xD83D      │   0xDE00      │                               │
│   └──────────────┴──────────────┘                                │
│                                                                   │
│   Fórmula:                                                        │
│   high = 0xD800 + ((cp - 0x10000) >> 10)                         │
│   low  = 0xDC00 + ((cp - 0x10000) & 0x3FF)                       │
│                                                                   │
│   Surrogate range: U+D800 .. U+DFFF (reservado, no son caracteres)│
└──────────────────────────────────────────────────────────────────┘
```

**Consecuencias prácticas en Java:**

```java
public class SurrogateDemo {
    public static void main(String[] args) {
        String emoji = "😀🎉";
        String gothic = "𐍈";   // U+10348 GOTHIC LETTER HWAIR

        System.out.println("=== '😀🎉' ===");
        System.out.println("  length():      " + emoji.length());       // 4 (¡no 2!)
        System.out.println("  codePoints():  " + emoji.codePoints().count());  // 2
        System.out.println("  chars:");
        for (int i = 0; i < emoji.length(); i++) {
            char c = emoji.charAt(i);
            System.out.printf("    [%d] = U+%04X (%s surrogado)%n",
                i, (int) c,
                Character.isSurrogate(c) ? "ES" : "NO"
            );
        }

        System.out.println("\n=== '𐍈' ===");
        System.out.println("  length():      " + gothic.length());       // 2
        System.out.println("  codePoints():  " + gothic.codePoints().count());  // 1

        System.out.println("\n=== Iteración correcta con codePoints ===");
        gothic.codePoints().forEach(cp ->
            System.out.printf("  U+%05X: %s%n", cp,
                new String(Character.toChars(cp)))
        );
    }
}
```

**Salida:**
```
=== '😀🎉' ===
  length():      4
  codePoints():  2
  chars:
    [0] = U+D83D (ES surrogado)
    [1] = U+DE00 (ES surrogado)
    [2] = U+D83D (ES surrogado)
    [3] = U+DE09 (ES surrogado)

=== '𐍈' ===
  length():      2
  codePoints():  1
```

**Regla de oro:** para iterar correctamente sobre caracteres que incluyan emojis o escrituras antiguas, usa **`codePoints()` en lugar de `charAt()`**.

```java
// INCORRECTO: rompe surrogate pairs
for (int i = 0; i < texto.length(); i++) {
    char c = texto.charAt(i);
}

// CORRECTO: itera code points completos
texto.codePoints().forEach(cp -> { /* ... */ });

// También correcto: usando offsetByCodePoints
for (int i = 0; i < texto.length(); ) {
    int cp = texto.codePointAt(i);
    System.out.println("U+" + Integer.toHexString(cp));
    i += Character.charCount(cp);
}
```

### 6.4.6 ISO-8859-1 (Latin-1) y otras codificaciones históricas

| Charset | Bits | Rango | Uso |
|---------|------|-------|-----|
| **US-ASCII** | 7 | 0-127 | Solo inglés básico, la base común |
| **ISO-8859-1** | 8 | 0-255 | Europa occidental (áéíóúñ¿¡) |
| **ISO-8859-15** | 8 | 0-255 | Como Latin-1, pero incluye € |
| **Windows-1252** | 8 | 0-255 | Extensión de Latin-1 de Microsoft |
| **UTF-8** | 8-32 | 0-1,114,111 | Universal, el estándar moderno |
| **UTF-16** | 16-32 | 0-1,114,111 | Representación interna de Java |
| **UTF-16LE/BE** | 16-32 | 0-1,114,111 | UTF-16 con endianness explícito |

### 6.4.7 BOM (Byte Order Mark): el marcador invisible

El **BOM** es un carácter especial (U+FEFF) colocado al inicio de un archivo para indicar su codificación y endianness:

```
┌─────────────────────────────────────────────────────────────────┐
│   BOM (BYTE ORDER MARK) — U+FEFF                                 │
│                                                                  │
│   UTF-8:    [EF BB BF]  ← BOM en UTF-8 (opcional, no necesario) │
│   UTF-16BE: [FE FF]     ← Big-Endian                             │
│   UTF-16LE: [FF FE]     ← Little-Endian                          │
│   UTF-32BE: [00 00 FE FF]                                        │
│   UTF-32LE: [FF FE 00 00]                                        │
│                                                                  │
│   ⚠ En UTF-8, el BOM NO es recomendado (rompe herramientas)     │
│   ⚠ En Windows, Notepad añade BOM a archivos UTF-8               │
└─────────────────────────────────────────────────────────────────┘
```

**Lectura segura de archivos con posible BOM:**

```java
import java.io.*;
import java.nio.charset.Charset;
import java.nio.charset.StandardCharsets;
import java.nio.file.*;

public class BOMHandler {

    public static String leerArchivoSinBOM(Path ruta) throws IOException {
        try (InputStream is = Files.newInputStream(ruta);
             PushbackInputStream pis = new PushbackInputStream(is, 4)) {

            byte[] bom = new byte[4];
            int leidos = pis.read(bom);
            Charset charset = StandardCharsets.UTF_8;

            if (leidos >= 3 && bom[0] == (byte)0xEF && bom[1] == (byte)0xBB && bom[2] == (byte)0xBF) {
                System.out.println("BOM UTF-8 detectado y eliminado");
                pis.unread(bom, 3, leidos - 3);
            } else if (leidos >= 2 && bom[0] == (byte)0xFE && bom[1] == (byte)0xFF) {
                System.out.println("BOM UTF-16BE detectado");
                charset = StandardCharsets.UTF_16BE;
                pis.unread(bom, 2, leidos - 2);
            } else if (leidos >= 2 && bom[0] == (byte)0xFF && bom[1] == (byte)0xFE) {
                System.out.println("BOM UTF-16LE detectado");
                charset = StandardCharsets.UTF_16LE;
                pis.unread(bom, 2, leidos - 2);
            } else {
                pis.unread(bom, 0, leidos);
            }

            try (BufferedReader reader = new BufferedReader(
                     new InputStreamReader(pis, charset))) {
                StringBuilder sb = new StringBuilder();
                String linea;
                while ((linea = reader.readLine()) != null) {
                    sb.append(linea).append("\n");
                }
                return sb.toString();
            }
        }
    }
}
```

### 6.4.8 `Charset.forName()` y `StandardCharsets` (Java 7+)

```java
import java.nio.charset.Charset;
import java.nio.charset.StandardCharsets;
import java.util.SortedMap;

public class CharsetDemo {
    public static void main(String[] args) {
        // StandardCharsets — no lanza excepción, tipos seguros
        Charset utf8    = StandardCharsets.UTF_8;
        Charset latin1  = StandardCharsets.ISO_8859_1;
        Charset utf16   = StandardCharsets.UTF_16;
        Charset ascii   = StandardCharsets.US_ASCII;

        // Charset.forName() — puede lanzar UnsupportedCharsetException
        Charset win1252 = Charset.forName("windows-1252");

        System.out.println("UTF-8 display name: " + utf8.displayName());
        System.out.println("UTF-8 aliases:      " + utf8.aliases());

        SortedMap<String, Charset> disponibles = Charset.availableCharsets();
        System.out.println("\nCharsets disponibles: " + disponibles.size());

        // Codificar y decodificar
        String texto = "España";
        byte[] bytes = texto.getBytes(utf8);
        System.out.print("Bytes (hex): ");
        for (byte b : bytes) System.out.printf("%02X ", b);
        System.out.println();
    }
}
```

### 6.4.9 Leer archivo con encoding incorrecto y cómo corregirlo

Problema clásico: recibes un archivo y no sabes su encoding, o te lo enviaron con el encoding equivocado.

```java
import java.nio.charset.*;
import java.nio.file.*;
import java.io.*;
import java.nio.ByteBuffer;

public class CorregirEncoding {

    // Caso 1: Archivo ISO-8859-1 leído como UTF-8 → mojibake
    public static void caso1ArchivoLatin1LeidoComoUTF8() throws IOException {
        Path archivo = Paths.get("texto-latin1.txt");

        String original = "José García — año 2024 — ¡Hola!";
        Files.write(archivo, original.getBytes(StandardCharsets.ISO_8859_1));

        // LEER CON ENCODING INCORRECTO
        String incorrecto = Files.readString(archivo, StandardCharsets.UTF_8);
        System.out.println("Leído como UTF-8 (INCORRECTO): " + incorrecto);
        // Salida basura: "JosÃ© GarcÃa â€” aÃ±o 2024 â€” Â¡Hola!"

        // CORRECCIÓN
        String correcto = Files.readString(archivo, StandardCharsets.ISO_8859_1);
        System.out.println("Leído como Latin-1 (CORRECTO): " + correcto);
    }

    // Caso 2: Archivo UTF-8 con BOM leído sin manejarlo
    public static void caso2BOMInesperado() throws IOException {
        Path archivo = Paths.get("texto-utf8-bom.txt");

        // Simular archivo UTF-8 con BOM (Notepad de Windows)
        try (OutputStream os = Files.newOutputStream(archivo)) {
            os.write(0xEF); os.write(0xBB); os.write(0xBF);
            os.write("Hola mundo".getBytes(StandardCharsets.UTF_8));
        }

        // Lectura ingenua — el BOM aparece como carácter basura
        String conBasura = Files.readString(archivo, StandardCharsets.UTF_8);
        System.out.println("Con BOM (basura): '" + conBasura + "'");

        // Solución: Eliminar BOM manualmente
        byte[] bytes = Files.readAllBytes(archivo);
        int inicio = (bytes.length >= 3 &&
                      bytes[0] == (byte)0xEF &&
                      bytes[1] == (byte)0xBB &&
                      bytes[2] == (byte)0xBF) ? 3 : 0;
        String corregido = new String(bytes, inicio, bytes.length - inicio,
            StandardCharsets.UTF_8);
        System.out.println("Sin BOM (corregido): '" + corregido + "'");

        Files.delete(archivo);
    }

    // Caso 3: Detección heurística de encoding
    public static Charset detectarEncoding(byte[] bytes) {
        if (bytes.length >= 3 &&
            (bytes[0] & 0xFF) == 0xEF &&
            (bytes[1] & 0xFF) == 0xBB &&
            (bytes[2] & 0xFF) == 0xBF) {
            return StandardCharsets.UTF_8;
        }
        if (bytes.length >= 2 &&
            (bytes[0] & 0xFF) == 0xFF && (bytes[1] & 0xFF) == 0xFE) {
            return StandardCharsets.UTF_16LE;
        }
        if (bytes.length >= 2 &&
            (bytes[0] & 0xFF) == 0xFE && (bytes[1] & 0xFF) == 0xFF) {
            return StandardCharsets.UTF_16BE;
        }

        try {
            StandardCharsets.UTF_8.newDecoder().decode(ByteBuffer.wrap(bytes));
            return StandardCharsets.UTF_8;
        } catch (CharacterCodingException e) {
            return StandardCharsets.ISO_8859_1;
        }
    }

    public static void main(String[] args) throws IOException {
        caso1ArchivoLatin1LeidoComoUTF8();
        caso2BOMInesperado();
    }
}
```

---

## 6.5 Archivos: la clase `File`

La clase `java.io.File` representa una ruta a un archivo o directorio en el sistema de archivos. **No representa el contenido del archivo**, sino su ubicación y metadatos.

### 6.7.1 Métodos principales de `File`

```java
import java.io.File;
import java.io.IOException;
import java.util.Date;

public class FileDemo {
    public static void main(String[] args) throws IOException {
        File archivo = new File("documento.txt");

        // Verificar existencia y tipo
        System.out.println("¿Existe?           " + archivo.exists());
        System.out.println("¿Es archivo?       " + archivo.isFile());
        System.out.println("¿Es directorio?    " + archivo.isDirectory());

        // Crear un archivo vacío
        if (!archivo.exists()) {
            boolean creado = archivo.createNewFile();
            System.out.println("¿Archivo creado?   " + creado);
        }

        // Metadatos
        System.out.println("Nombre:            " + archivo.getName());
        System.out.println("Ruta relativa:     " + archivo.getPath());
        System.out.println("Ruta absoluta:     " + archivo.getAbsolutePath());
        System.out.println("Directorio padre:  " + archivo.getParent());
        System.out.println("Tamaño (bytes):    " + archivo.length());
        System.out.println("Última modificación: " + new Date(archivo.lastModified()));

        // Permisos
        System.out.println("¿Se puede leer?    " + archivo.canRead());
        System.out.println("¿Se puede escribir? " + archivo.canWrite());
        System.out.println("¿Se puede ejecutar? " + archivo.canExecute());
    }
}
```

### 6.7.2 Trabajar con directorios

```java
import java.io.File;
import java.io.IOException;

public class DirectoriosDemo {
    public static void main(String[] args) throws IOException {
        File dir = new File("proyecto/submodulo/datos");

        // mkdir() solo crea el último directorio si el padre existe
        // mkdirs() crea toda la jerarquía de directorios necesaria
        if (!dir.exists()) {
            boolean creado = dir.mkdirs();
            System.out.println("¿Directorios creados? " + creado);
        }

        // Crear algunos archivos dentro
        new File(dir, "archivo1.txt").createNewFile();
        new File(dir, "archivo2.txt").createNewFile();
        new File(dir, "notas.md").createNewFile();

        // Listar contenido del directorio
        System.out.println("\nContenido de " + dir.getPath() + ":");
        String[] nombres = dir.list();
        if (nombres != null) {
            for (String nombre : nombres) {
                System.out.println("  " + nombre);
            }
        }

        // Listar con filtro (solo archivos .txt)
        System.out.println("\nSolo archivos .txt:");
        File[] archivosTxt = dir.listFiles((d, name) -> name.endsWith(".txt"));
        if (archivosTxt != null) {
            for (File f : archivosTxt) {
                System.out.println("  " + f.getName() + " (" + f.length() + " bytes)");
            }
        }

        // Eliminar (solo borra archivos vacíos o directorios vacíos)
        new File(dir, "archivo1.txt").delete();
        System.out.println("\n¿archivo1.txt eliminado? " +
                           !new File(dir, "archivo1.txt").exists());
    }
}
```

### 6.7.3 Separador de rutas y diferencias entre sistemas operativos

Las rutas de archivo varían entre sistemas operativos. Java ofrece constantes para abstraer estas diferencias:

```java
import java.io.File;

public class SeparadoresDemo {
    public static void main(String[] args) {
        System.out.println("Separador de archivos:  '" + File.separator + "'");
        System.out.println("Separador de classpath: '" + File.pathSeparator + "'");

        // Construcción portable de rutas
        String ruta = "docs" + File.separator + "manuales" + File.separator + "intro.txt";
        System.out.println("Ruta portable: " + ruta);

        // Alternativa: usar el constructor que acepta padre e hijo
        File portable = new File(new File("docs", "manuales"), "intro.txt");
        System.out.println("Con constructores: " + portable.getPath());
    }
}
```

| Sistema   | `File.separator` | `File.pathSeparator` | Ruta de ejemplo |
|-----------|------------------|----------------------|-----------------|
| Windows   | `\`              | `;`                  | `C:\Users\docs\archivo.txt` |
| Linux/macOS | `/`            | `:`                  | `/home/user/docs/archivo.txt` |

### 6.7.4 Ejemplo completo: recorrer un directorio recursivamente con `File`

```java
import java.io.File;

public class RecorrerDirectorio {
    public static void main(String[] args) {
        if (args.length == 0) {
            System.out.println("Uso: java RecorrerDirectorio <directorio>");
            return;
        }
        recorrer(new File(args[0]), 0);
    }

    private static void recorrer(File dir, int nivel) {
        File[] archivos = dir.listFiles();
        if (archivos == null) return;

        for (File f : archivos) {
            String indentacion = "  ".repeat(nivel);
            if (f.isDirectory()) {
                System.out.println(indentacion + "[DIR]  " + f.getName() + "/");
                recorrer(f, nivel + 1);
            } else {
                System.out.printf("%s[FILE] %s (%d bytes)%n",
                                  indentacion, f.getName(), f.length());
            }
        }
    }
}
```

---

## 6.6 Serialización

### 6.7.1 ¿Qué es la serialización?

La **serialización** es el proceso de convertir un objeto Java en una secuencia de bytes para almacenarlo en un archivo, enviarlo por red o guardarlo en una base de datos. La operación inversa, reconstruir el objeto a partir de los bytes, se llama **deserialización**.

### 6.7.2 La interfaz `Serializable`

Es una **interfaz marcadora** (marker interface): no declara ningún método, simplemente indica a la JVM que los objetos de esa clase pueden ser serializados.

```java
import java.io.Serializable;

public class Persona implements Serializable {
    // serialVersionUID: identificador único de versión de la clase
    private static final long serialVersionUID = 1L;

    private String nombre;
    private int edad;
    private transient String contrasenia;  // transient: no se serializa

    public Persona(String nombre, int edad, String contrasenia) {
        this.nombre = nombre;
        this.edad = edad;
        this.contrasenia = contrasenia;
    }

    @Override
    public String toString() {
        return "Persona{nombre='" + nombre + "', edad=" + edad +
               ", contrasenia='" + contrasenia + "'}";
    }
}
```

### 6.7.3 `ObjectOutputStream` y `ObjectInputStream`

```java
import java.io.*;

public class SerializacionDemo {
    public static void main(String[] args) {
        Persona persona = new Persona("Carlos", 30, "secreto123");

        // Serializar (escribir objeto a archivo)
        try (ObjectOutputStream oos = new ObjectOutputStream(
                 new FileOutputStream("persona.ser"))) {
            oos.writeObject(persona);
            System.out.println("Objeto serializado: " + persona);
        } catch (IOException e) {
            e.printStackTrace();
        }

        // Deserializar (leer objeto desde archivo)
        try (ObjectInputStream ois = new ObjectInputStream(
                 new FileInputStream("persona.ser"))) {
            Persona recuperada = (Persona) ois.readObject();
            System.out.println("Objeto deserializado: " + recuperada);
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

**Salida:**
```
Objeto serializado: Persona{nombre='Carlos', edad=30, contrasenia='secreto123'}
Objeto deserializado: Persona{nombre='Carlos', edad=30, contrasenia='null'}
```

Observa que `contrasenia` es `null` tras la deserialización: el modificador `transient` impidió su serialización.

### 6.7.4 `serialVersionUID`

Cuando deserializas un objeto, la JVM compara el `serialVersionUID` de la clase en el archivo con el de la clase cargada en tiempo de ejecución. Si no coinciden, lanza `InvalidClassException`.

```java
private static final long serialVersionUID = 1L;
```

**¿Por qué es importante?** Si evolucionas tu clase (añades un campo, cambias un tipo) sin actualizar el `serialVersionUID`, los archivos serializados previamente serán incompatibles. Las opciones son:

1. **No declarar `serialVersionUID`**: el compilador lo genera automáticamente basándose en la estructura de la clase. Cualquier cambio mínimo produce uno nuevo, rompiendo compatibilidad.
2. **Declararlo explícitamente**: mantienes control. Si haces cambios compatibles, conservas el mismo ID. Si haces cambios incompatibles, lo cambias deliberadamente.
3. **Usar serialización personalizada** para manejar la evolución con gracia.

### 6.7.5 Serialización personalizada con `writeObject` / `readObject`

Puedes definir exactamente cómo se serializa tu clase implementando estos métodos (que **no** son parte de la interfaz `Serializable`, sino un contrato especial reconocido por la JVM):

```java
import java.io.*;

public class CuentaBancaria implements Serializable {
    private static final long serialVersionUID = 2L;

    private String titular;
    private transient double saldo;         // No se serializa directamente
    private static final double TASA = 0.05; // static: nunca se serializa

    public CuentaBancaria(String titular, double saldo) {
        this.titular = titular;
        this.saldo = saldo;
    }

    // Método mágico: la JVM lo invoca durante la serialización
    private void writeObject(ObjectOutputStream oos) throws IOException {
        oos.defaultWriteObject();      // Serializa los campos no-transient normalmente
        oos.writeDouble(saldo * 1.10); // Serializa el saldo con un "impuesto" o transformación
    }

    // Método mágico: la JVM lo invoca durante la deserialización
    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ois.defaultReadObject();       // Deserializa los campos no-transient
        this.saldo = ois.readDouble() / 1.10; // Revierte la transformación
    }

    @Override
    public String toString() {
        return "Cuenta{titular='" + titular + "', saldo=" + saldo + ", TASA=" + TASA + "}";
    }

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        CuentaBancaria cuenta = new CuentaBancaria("María", 5000.0);
        System.out.println("Original: " + cuenta);

        // Serializar
        try (ObjectOutputStream oos = new ObjectOutputStream(
                 new FileOutputStream("cuenta.ser"))) {
            oos.writeObject(cuenta);
        }

        // Deserializar
        try (ObjectInputStream ois = new ObjectInputStream(
                 new FileInputStream("cuenta.ser"))) {
            CuentaBancaria recuperada = (CuentaBancaria) ois.readObject();
            System.out.println("Recuperada: " + recuperada);
        }
    }
}
```

### 6.7.6 La interfaz `Externalizable`

`Externalizable` extiende `Serializable` y te da control total sobre el formato de serialización. Debes implementar dos métodos:

```java
import java.io.*;

public class Configuracion implements Externalizable {
    private String servidor;
    private int puerto;
    private boolean tls;

    // Constructor sin argumentos OBLIGATORIO para Externalizable
    public Configuracion() {}

    public Configuracion(String servidor, int puerto, boolean tls) {
        this.servidor = servidor;
        this.puerto = puerto;
        this.tls = tls;
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeUTF(servidor);
        out.writeInt(puerto);
        out.writeBoolean(tls);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        servidor = in.readUTF();
        puerto   = in.readInt();
        tls      = in.readBoolean();
    }

    @Override
    public String toString() {
        return String.format("Config{servidor=%s, puerto=%d, tls=%b}", servidor, puerto, tls);
    }
}
```

> **Diferencia clave con `Serializable`:** `Externalizable` no usa mecanismos automáticos de serialización. El programador controla exactamente qué y cómo se escribe, lo que puede mejorar el rendimiento, pero requiere más código. Además, la clase debe tener un constructor público sin argumentos.

### 6.7.7 Consideraciones de seguridad

La deserialización de datos no confiables es una de las vulnerabilidades más explotadas en Java:

- **Ataques de inyección de objetos**: un atacante puede crear un flujo de bytes malicioso que, al ser deserializado, ejecute código arbitrario mediante "gadget chains".
- **Validación de entrada**: nunca deserialices datos de fuentes no confiables sin validación.
- **Alternativas modernas**: considera usar formatos como JSON (Jackson, Gson) o Protocol Buffers, que no ejecutan código durante el parsing.
- **`ObjectInputFilter`** (Java 9+): permite definir filtros para rechazar clases no permitidas durante la deserialización.

```java
// Ejemplo de filtro de deserialización (Java 9+)
ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
    "com.miapp.modelo.*;!*"  // Solo permite clases del paquete com.miapp.modelo
);
ObjectInputStream ois = new ObjectInputStream(fis);
ois.setObjectInputFilter(filter);
```

---

## 6.7 NIO.2 (`java.nio.file`, Java 7+)

El paquete `java.nio.file` (conocido como **NIO.2**) introducido en Java 7 moderniza el manejo de archivos y directorios, reemplazando en gran medida a la clase `java.io.File`.

### 6.7.1 La interfaz `Path`

`Path` reemplaza a `File` como representación de una ruta en el sistema de archivos. Es más potente, flexible y está mejor integrada con el resto de NIO.2.

```java
import java.nio.file.Path;
import java.nio.file.Paths;

public class PathDemo {
    public static void main(String[] args) {
        // Crear Path: método factory Paths.get()
        Path ruta = Paths.get("docs", "manuales", "intro.txt");
        System.out.println("Ruta:            " + ruta);
        System.out.println("Raíz:            " + ruta.getRoot());
        System.out.println("Nombre archivo:  " + ruta.getFileName());
        System.out.println("Directorio padre:" + ruta.getParent());
        System.out.println("Número de elementos: " + ruta.getNameCount());

        // Iterar elementos de la ruta
        System.out.println("\nElementos de la ruta:");
        for (int i = 0; i < ruta.getNameCount(); i++) {
            System.out.println("  [" + i + "] " + ruta.getName(i));
        }

        // Resolver rutas
        Path base = Paths.get("/home/usuario");
        Path completo = base.resolve("documentos/notas.txt");
        System.out.println("\nRuta resuelta: " + completo);

        // Obtener ruta absoluta normalizada
        Path relativa = Paths.get("proyecto/../proyecto/src/Main.java");
        System.out.println("Normalizada: " + relativa.normalize());

        // Convertir File <-> Path
        java.io.File fileLegacy = ruta.toFile();
        Path desdeFile = fileLegacy.toPath();
        System.out.println("\nConversión File -> Path: " + desdeFile);
    }
}
```

### 6.7.2 La clase utilitaria `Files`

`java.nio.file.Files` ofrece métodos estáticos para prácticamente todas las operaciones con archivos. Es el reemplazo moderno de muchas operaciones que antes requerían código manual con streams.

#### Operaciones básicas

```java
import java.nio.file.*;
import java.io.IOException;

public class FilesBasico {
    public static void main(String[] args) throws IOException {
        Path dir  = Paths.get("datos-procesados");
        Path arch = dir.resolve("resultados.txt");

        // Crear directorios
        if (!Files.exists(dir)) {
            Files.createDirectories(dir);
        }

        // Crear archivo
        if (!Files.exists(arch)) {
            Files.createFile(arch);
        }

        // Verificar existencia y tipo
        System.out.println("¿Existe?           " + Files.exists(arch));
        System.out.println("¿Es directorio?    " + Files.isDirectory(arch));
        System.out.println("¿Es archivo regular?" + Files.isRegularFile(arch));
        System.out.println("¿Es legible?       " + Files.isReadable(arch));
        System.out.println("¿Es escribible?    " + Files.isWritable(arch));

        // Metadatos
        System.out.println("Tamaño:            " + Files.size(arch) + " bytes");
        System.out.println("Última modificación: " + Files.getLastModifiedTime(arch));

        // Copiar y mover
        Path copia = dir.resolve("resultados-backup.txt");
        Files.copy(arch, copia, StandardCopyOption.REPLACE_EXISTING);
        System.out.println("Copia creada: " + copia);

        Path renombrado = dir.resolve("resultados-v2.txt");
        Files.move(arch, renombrado, StandardCopyOption.REPLACE_EXISTING);
        System.out.println("Movido a: " + renombrado);

        // Eliminar
        Files.deleteIfExists(copia);
        Files.deleteIfExists(renombrado);
        System.out.println("Archivos temporales eliminados.");
    }
}
```

#### Lectura y escritura simplificada

```java
import java.nio.file.*;
import java.io.IOException;
import java.util.List;

public class FilesLecturaEscritura {
    public static void main(String[] args) throws IOException {
        Path archivo = Paths.get("poema.txt");

        // Escribir líneas directamente
        List<String> lineas = List.of(
            "En un lugar de la Mancha,",
            "de cuyo nombre no quiero acordarme,",
            "no ha mucho tiempo que vivía",
            "un hidalgo de los de lanza en astillero...");

        Files.write(archivo, lineas, StandardOpenOption.CREATE);
        System.out.println("Archivo escrito.");

        // Leer todas las líneas a una List<String>
        List<String> leidas = Files.readAllLines(archivo);
        System.out.println("\nContenido leído:");
        leidas.forEach(System.out::println);

        // Leer todo el contenido como String
        String contenido = Files.readString(archivo);  // Java 11+
        System.out.println("\nComo String único:\n" + contenido);

        // lines() devuelve Stream<String> — procesamiento funcional
        System.out.println("\nLíneas que contienen 'de':");
        try (var stream = Files.lines(archivo)) {
            stream.filter(l -> l.contains("de"))
                  .forEach(System.out::println);
        }
    }
}
```

#### `walk()`, `find()` y `list()`

```java
import java.nio.file.*;
import java.io.IOException;
import java.util.stream.Stream;

public class FilesWalkFind {
    public static void main(String[] args) throws IOException {
        Path inicio = Paths.get(".");

        // list(): contenido inmediato de un directorio (no recursivo)
        System.out.println("=== Contenido directo de " + inicio.toAbsolutePath() + " ===");
        try (Stream<Path> stream = Files.list(inicio)) {
            stream.forEach(p -> System.out.println("  " + p.getFileName()));
        }

        // walk(): recorrido recursivo (depth-first)
        System.out.println("\n=== Árbol completo (profundidad máx. 2) ===");
        try (Stream<Path> stream = Files.walk(inicio, 2)) {
            stream.forEach(System.out::println);
        }

        // find(): búsqueda recursiva con predicado
        System.out.println("\n=== Archivos .java encontrados ===");
        try (Stream<Path> stream = Files.find(inicio, 10,
                (path, attrs) -> path.toString().endsWith(".java"))) {
            stream.forEach(System.out::println);
        }
    }
}
```

### 6.7.3 Lectura y escritura con buffering en NIO.2

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.io.*;

public class NioBuffering {
    public static void main(String[] args) throws IOException {
        Path archivo = Paths.get("datos-procesados", "salida.txt");
        Files.createDirectories(archivo.getParent());

        // newBufferedWriter con charset explícito
        try (BufferedWriter writer = Files.newBufferedWriter(
                 archivo, StandardCharsets.UTF_8,
                 StandardOpenOption.CREATE, StandardOpenOption.APPEND)) {
            writer.write("Registro agregado al final del archivo.");
            writer.newLine();
        }

        // newBufferedReader
        try (BufferedReader reader = Files.newBufferedReader(archivo, StandardCharsets.UTF_8)) {
            String linea;
            while ((linea = reader.readLine()) != null) {
                System.out.println(linea);
            }
        }

        // newInputStream / newOutputStream para datos binarios
        Path binario = Paths.get("datos.bin");
        try (OutputStream os = Files.newOutputStream(binario);
             DataOutputStream dos = new DataOutputStream(os)) {
            dos.writeInt(12345);
            dos.writeDouble(9.81);
        }

        try (InputStream is = Files.newInputStream(binario);
             DataInputStream dis = new DataInputStream(is)) {
            System.out.println("Leído: " + dis.readInt() + ", " + dis.readDouble());
        }
    }
}
```

### 6.7.4 Recorrido recursivo con `FileVisitor` y `SimpleFileVisitor`

Para operaciones complejas de recorrido de directorios, `FileVisitor` ofrece un control más fino que `walk()`.

```java
import java.nio.file.*;
import java.nio.file.attribute.BasicFileAttributes;
import java.io.IOException;

public class ExploradorArchivos extends SimpleFileVisitor<Path> {

    @Override
    public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) {
        System.out.println("[DIR]  " + dir);
        return FileVisitResult.CONTINUE;
    }

    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) {
        System.out.printf("[FILE] %s (%d bytes)%n", file.getFileName(), attrs.size());
        return FileVisitResult.CONTINUE;
    }

    @Override
    public FileVisitResult visitFileFailed(Path file, IOException exc) {
        System.err.println("Error al visitar: " + file + " — " + exc.getMessage());
        return FileVisitResult.CONTINUE;
    }

    public static void main(String[] args) throws IOException {
        Path inicio = Paths.get("src");
        Files.walkFileTree(inicio, new ExploradorArchivos());
    }
}
```

`FileVisitResult` controla el flujo del recorrido:

| Valor | Significado |
|-------|-------------|
| `CONTINUE` | Continuar normalmente |
| `SKIP_SUBTREE` | Saltar el contenido de este directorio (solo en `preVisitDirectory`) |
| `SKIP_SIBLINGS` | Saltar los hermanos restantes (no visitar más entradas en este directorio) |
| `TERMINATE` | Finalizar el recorrido inmediatamente |

#### Ejemplo avanzado: eliminar un directorio recursivamente

```java
import java.nio.file.*;
import java.nio.file.attribute.BasicFileAttributes;
import java.io.IOException;

public class BorrarDirectorio extends SimpleFileVisitor<Path> {

    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
        Files.delete(file);
        System.out.println("Borrado archivo: " + file);
        return FileVisitResult.CONTINUE;
    }

    @Override
    public FileVisitResult postVisitDirectory(Path dir, IOException exc) throws IOException {
        if (exc != null) throw exc;
        Files.delete(dir);
        System.out.println("Borrado directorio: " + dir);
        return FileVisitResult.CONTINUE;
    }

    public static void main(String[] args) throws IOException {
        Path aBorrar = Paths.get("directorio-temporal");
        if (Files.exists(aBorrar)) {
            Files.walkFileTree(aBorrar, new BorrarDirectorio());
            System.out.println("Directorio eliminado completamente.");
        }
    }
}
```

### 6.7.5 `WatchService`: Monitoreo de cambios en el sistema de archivos

Permite reaccionar ante eventos como creación, modificación o eliminación de archivos en tiempo real.

```java
import java.nio.file.*;
import java.util.List;

public class MonitorArchivos {
    public static void main(String[] args) throws Exception {
        Path dir = Paths.get("monitoreado");
        Files.createDirectories(dir);

        WatchService watcher = FileSystems.getDefault().newWatchService();
        dir.register(watcher,
                     StandardWatchEventKinds.ENTRY_CREATE,
                     StandardWatchEventKinds.ENTRY_MODIFY,
                     StandardWatchEventKinds.ENTRY_DELETE);

        System.out.println("Monitoreando " + dir.toAbsolutePath() + "...");
        System.out.println("(Crea, modifica o elimina archivos en ese directorio)");

        while (true) {
            WatchKey key = watcher.take();  // Bloquea hasta recibir un evento

            for (WatchEvent<?> event : key.pollEvents()) {
                WatchEvent.Kind<?> kind = event.kind();

                if (kind == StandardWatchEventKinds.OVERFLOW) {
                    System.out.println("!!! Eventos perdidos por overflow");
                    continue;
                }

                Path nombreArchivo = (Path) event.context();

                if (kind == StandardWatchEventKinds.ENTRY_CREATE)
                    System.out.println("[CREADO]    " + nombreArchivo);
                else if (kind == StandardWatchEventKinds.ENTRY_MODIFY)
                    System.out.println("[MODIFICADO] " + nombreArchivo);
                else if (kind == StandardWatchEventKinds.ENTRY_DELETE)
                    System.out.println("[ELIMINADO]  " + nombreArchivo);
            }

            if (!key.reset()) {
                System.out.println("El directorio ya no es accesible. Saliendo...");
                break;
            }
        }

        watcher.close();
    }
}
```

> **Limitaciones:** `WatchService` depende del sistema operativo subyacente. En algunos sistemas de archivos o configuraciones de red, los eventos pueden no ser confiables o no estar soportados. Además, monitorear directorios individuales es eficiente, pero monitorear un árbol completo requiere registrar cada subdirectorio manualmente.

### 6.7.6 Atributos de archivos con `BasicFileAttributes`

```java
import java.nio.file.*;
import java.nio.file.attribute.BasicFileAttributes;
import java.io.IOException;

public class AtributosDemo {
    public static void main(String[] args) throws IOException {
        Path archivo = Paths.get("atributos-demo.txt");
        Files.writeString(archivo, "Contenido de prueba");  // Java 11+

        BasicFileAttributes attrs = Files.readAttributes(archivo, BasicFileAttributes.class);

        System.out.println("=== Atributos básicos ===");
        System.out.println("Tamaño:                 " + attrs.size());
        System.out.println("Fecha de creación:      " + attrs.creationTime());
        System.out.println("Último acceso:          " + attrs.lastAccessTime());
        System.out.println("Última modificación:    " + attrs.lastModifiedTime());
        System.out.println("¿Es directorio?         " + attrs.isDirectory());
        System.out.println("¿Es archivo regular?    " + attrs.isRegularFile());
        System.out.println("¿Es enlace simbólico?   " + attrs.isSymbolicLink());
        System.out.println("Key del archivo:        " + attrs.fileKey());

        // Atributos específicos del sistema (ej: POSIX en Linux/macOS)
        try {
            var posix = Files.readAttributes(archivo, java.nio.file.attribute.PosixFileAttributes.class);
            System.out.println("\n=== Atributos POSIX ===");
            System.out.println("Permisos:  " + PosixFilePermissions.toString(posix.permissions()));
            System.out.println("Propietario: " + posix.owner().getName());
            System.out.println("Grupo:       " + posix.group().getName());
        } catch (UnsupportedOperationException e) {
            System.out.println("Atributos POSIX no soportados en este sistema.");
        }
    }
}
```

---



---

## 6.8 Memory-Mapped Files: Mapeo de archivos en memoria

### 6.8.1 ¿Qué es el mapeo de archivos en memoria?

Un **memory-mapped file** asigna una región de un archivo directamente en la memoria virtual del proceso. El sistema operativo se encarga de cargar y descargar páginas según sea necesario, sin copias intermedias explícitas.

> [!NOTE]
> ### 🪞 El Escritorio Mágico / El Espejo de Portales (Memory-Mapped Files)
>
> Imagina que necesitas leer y modificar un libro gigante de 1,000 páginas que se encuentra guardado en una bóveda lejana (el disco duro).
> - **El enfoque de lectura tradicional (Streams)**: Cada vez que quieres leer un renglón, tienes que enviar a un cartero a la bóveda. El cartero saca la página del libro, la mete en un portafolios (el buffer del sistema operativo), viaja de vuelta, copia la información en tu cuaderno de notas (el buffer de la JVM) y finalmente tú la lees en tu escritorio (tu aplicación). ¡Esto requiere un constante trasiego de mensajeros y copias de papel!
> - **El enfoque de mmap (El Espejo de Portales)**: En lugar de enviar mensajeros, colocas un **espejo de portales mágico en tu escritorio (un `MappedByteBuffer`)** que apunta directamente a las páginas del libro dentro de la bóveda lejana.
>   - Cuando posas tus ojos en el espejo, la página exacta se visualiza al instante (paginación bajo demanda administrada por el sistema operativo).
>   - Si tomas un bolígrafo y escribes sobre la superficie de tu espejo mágico, **la tinta se dibuja automáticamente y de forma instantánea sobre las hojas reales de papel dentro de la bóveda lejana**, omitiendo por completo a todos los mensajeros y cuadernos intermedios (Zero-Copy).
>
> **En resumen**: El mapeo de archivos en memoria virtual permite que Java acceda y altere archivos gigantescos directamente como si fueran arreglos en memoria RAM, dejando que el sistema operativo se encargue de sincronizar físicamente los bytes en disco de la forma más veloz y eficiente posible.

```
┌─────────────────────────────────────────────────────────────────┐
│        LECTURA TRADICIONAL vs MEMORY-MAPPED FILE                 │
│                                                                  │
│   TRADICIONAL:                                                   │
│   ┌────────┐    read()    ┌──────────┐    procesa   ┌─────────┐ │
│   │ Disco  │─────────────▶│ Buffer JVM│────────────▶│ Tu App  │ │
│   │        │              │ (copia #1)│              │         │ │
│   └────────┘              └──────────┘              └─────────┘ │
│         ↑                                               │       │
│         └─── Copia del SO al espacio del usuario ───────┘       │
│                                                                  │
│   MEMORY-MAPPED FILE (mmap):                                     │
│   ┌────────┐    mmap()    ┌──────────────────────────────────┐  │
│   │ Disco  │──────────────│ Memoria Virtual del Proceso       │  │
│   │        │              │ ┌──────────────────────────────┐ │  │
│   └────────┘              │ │ MappedByteBuffer (acceso     │ │  │
│                           │ │ directo como un array)       │ │  │
│         ╲                │ └──────────────────────────────┘ │  │
│          ╲ paginación    └──────────────────────────────────┘  │
│           ╲ bajo demanda                                        │
│                                                                  │
│   ✓ Sin copias intermedias (zero-copy)                           │
│   ✓ Acceso aleatorio ultrarrápido                                │
│   ✓ El SO gestiona la caché automáticamente                      │
└─────────────────────────────────────────────────────────────────┘
```

### 6.8.2 `FileChannel.map()`: ¿cómo funciona?

```java
import java.io.*;
import java.nio.*;
import java.nio.channels.*;
import java.nio.file.*;

public class MMapDemo {
    public static void main(String[] args) throws IOException {
        Path archivo = Paths.get("grande.bin");
        crearArchivoPrueba(archivo, 1000);

        try (FileChannel channel = FileChannel.open(archivo,
                StandardOpenOption.READ, StandardOpenOption.WRITE)) {

            // map() mapea una región del archivo a memoria
            // Parámetros: modo, posición inicial, tamaño
            MappedByteBuffer buffer = channel.map(
                FileChannel.MapMode.READ_WRITE,
                0,
                Files.size(archivo)
            );

            // Lectura directa como si fuera un array
            System.out.println("Primeros 10 bytes:");
            for (int i = 0; i < 10 && i < buffer.limit(); i++) {
                System.out.printf("%d ", buffer.get(i));
            }
            System.out.println();

            // Escritura directa (modifica el archivo en disco)
            buffer.put(0, (byte) 99);
            buffer.put(1, (byte) 100);
            System.out.println("Bytes modificados en posiciones 0 y 1.");

            buffer.force();  // Sincronizar a disco inmediatamente
        }
    }

    private static void crearArchivoPrueba(Path path, int tamano) throws IOException {
        byte[] datos = new byte[tamano];
        for (int i = 0; i < tamano; i++) {
            datos[i] = (byte) (i % 256);
        }
        Files.write(path, datos);
    }
}
```

### 6.8.3 Modos de `MappedByteBuffer`

| Modo | Constante | Lectura | Escritura | Propagación al archivo |
|------|-----------|---------|-----------|----------------------|
| Solo lectura | `MapMode.READ_ONLY` | Sí | No (`ReadOnlyBufferException`) | N/A |
| Lectura/escritura | `MapMode.READ_WRITE` | Sí | Sí | Los cambios se escriben al archivo |
| Privado (copy-on-write) | `MapMode.PRIVATE` | Sí | Sí | Los cambios NO afectan al archivo original |

```java
import java.io.*;
import java.nio.*;
import java.nio.channels.*;

public class ModosMMap {
    public static void main(String[] args) throws IOException {
        Path archivo = Paths.get("modos-test.bin");
        Files.write(archivo, new byte[]{10, 20, 30});

        // READ_ONLY
        try (FileChannel ch = FileChannel.open(archivo, StandardOpenOption.READ)) {
            MappedByteBuffer roBuf = ch.map(FileChannel.MapMode.READ_ONLY, 0, 3);
            System.out.println("READ_ONLY byte 0: " + roBuf.get(0));
            // roBuf.put(0, (byte)99); // Lanza ReadOnlyBufferException
        }

        // READ_WRITE: cambios van al archivo
        try (FileChannel ch = FileChannel.open(archivo,
                StandardOpenOption.READ, StandardOpenOption.WRITE)) {
            MappedByteBuffer rwBuf = ch.map(FileChannel.MapMode.READ_WRITE, 0, 3);
            rwBuf.put(0, (byte) 99);
            System.out.println("READ_WRITE: byte 0 modificado a 99");
        }

        byte[] leido = Files.readAllBytes(archivo);
        System.out.println("En disco tras READ_WRITE: byte[0]=" + leido[0]);

        // PRIVATE: copy-on-write — cambios solo en memoria
        try (FileChannel ch = FileChannel.open(archivo,
                StandardOpenOption.READ, StandardOpenOption.WRITE)) {
            MappedByteBuffer privBuf = ch.map(FileChannel.MapMode.PRIVATE, 0, 3);
            privBuf.put(0, (byte) 77);
            System.out.println("PRIVATE: byte 0 modificado a 77 (solo en memoria)");
        }

        leido = Files.readAllBytes(archivo);
        System.out.println("En disco tras PRIVATE: byte[0]=" + leido[0] + " (sigue siendo 99)");

        Files.delete(archivo);
    }
}
```

### 6.8.4 Ejemplo real: leer archivo enorme con mmap vs buffers

```java
import java.io.*;
import java.nio.*;
import java.nio.channels.*;

public class LecturaArchivoGrande {

    // Método 1: FileInputStream con buffer tradicional
    public static long lecturaTradicional(Path archivo) throws IOException {
        long suma = 0;
        byte[] buffer = new byte[8192];
        try (FileInputStream fis = new FileInputStream(archivo.toFile())) {
            int leidos;
            while ((leidos = fis.read(buffer)) != -1) {
                for (int i = 0; i < leidos; i++) {
                    suma += (buffer[i] & 0xFF);
                }
            }
        }
        return suma;
    }

    // Método 2: FileChannel con ByteBuffer directo
    public static long lecturaFileChannel(Path archivo) throws IOException {
        long suma = 0;
        try (FileChannel channel = FileChannel.open(archivo, StandardOpenOption.READ)) {
            ByteBuffer buffer = ByteBuffer.allocateDirect(8192);
            while (channel.read(buffer) != -1) {
                buffer.flip();
                while (buffer.hasRemaining()) {
                    suma += (buffer.get() & 0xFF);
                }
                buffer.clear();
            }
        }
        return suma;
    }

    // Método 3: Memory-Mapped File por fragmentos
    public static long lecturaMMap(Path archivo) throws IOException {
        long suma = 0;
        long tamano = Files.size(archivo);
        long tamanoFragmento = 128 * 1024 * 1024; // 128 MB

        try (FileChannel channel = FileChannel.open(archivo, StandardOpenOption.READ)) {
            long posicion = 0;
            while (posicion < tamano) {
                long actual = Math.min(tamanoFragmento, tamano - posicion);

                MappedByteBuffer buffer = channel.map(
                    FileChannel.MapMode.READ_ONLY, posicion, actual);

                while (buffer.hasRemaining()) {
                    suma += (buffer.get() & 0xFF);
                }

                posicion += actual;
            }
        }
        return suma;
    }

    public static void main(String[] args) throws IOException {
        Path archivo = Paths.get("archivo-prueba-grande.bin");

        if (!Files.exists(archivo)) {
            System.out.println("Creando archivo de prueba de 500 MB...");
            try (FileChannel ch = FileChannel.open(archivo,
                    StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {
                ByteBuffer buf = ByteBuffer.allocateDirect(1024 * 1024);
                for (int i = 0; i < 500; i++) {
                    buf.clear();
                    while (buf.hasRemaining()) buf.put((byte) (i % 256));
                    buf.flip();
                    ch.write(buf);
                }
            }
        }

        System.out.println("Tamaño: " + Files.size(archivo) / (1024*1024) + " MB");

        long t0 = System.currentTimeMillis();
        long r1 = lecturaTradicional(archivo);
        System.out.printf("FileInputStream:  %d ms%n", System.currentTimeMillis() - t0);

        t0 = System.currentTimeMillis();
        long r2 = lecturaFileChannel(archivo);
        System.out.printf("FileChannel:      %d ms%n", System.currentTimeMillis() - t0);

        t0 = System.currentTimeMillis();
        long r3 = lecturaMMap(archivo);
        System.out.printf("MappedByteBuffer: %d ms%n", System.currentTimeMillis() - t0);
    }
}
```

### 6.8.5 Ventajas y desventajas de mmap

| Ventajas | Desventajas |
|----------|-------------|
| **Zero-copy**: Sin copias intermedias JVM↔SO | **Tamaño limitado**: Máximo `Integer.MAX_VALUE` (2 GB) por fragmento |
| **Rendimiento**: Hasta 2-5x más rápido en accesos aleatorios | **Liberación impredecible**: El buffer se libera cuando el GC recolecta el `MappedByteBuffer` |
| **Carga bajo demanda**: El SO carga solo las páginas accedidas | **No portable**: Comportamiento dependiente del SO |
| **Acceso aleatorio**: Como si fuera un array en memoria | **Recursos**: Consume espacio de direcciones virtuales |
| **Compartición**: Múltiples procesos pueden mapear el mismo archivo | **Errores fatales**: `SIGBUS` si el archivo se trunca mientras está mapeado |

### 6.8.6 Fragmentación para archivos > 2 GB

Dado que `map()` solo acepta `int` como tamaño, para archivos enormes se mapea por fragmentos:

```java
import java.io.*;
import java.nio.*;
import java.nio.channels.*;

public class MMapArchivoGrande {

    private static final long TAMANO_FRAGMENTO = 512 * 1024 * 1024; // 512 MB

    public static long contarByte(Path archivo, byte valor) throws IOException {
        long contador = 0;
        long tamano = Files.size(archivo);

        try (FileChannel channel = FileChannel.open(archivo, StandardOpenOption.READ)) {
            for (long pos = 0; pos < tamano; pos += TAMANO_FRAGMENTO) {
                long frag = Math.min(TAMANO_FRAGMENTO, tamano - pos);

                MappedByteBuffer buf = channel.map(
                    FileChannel.MapMode.READ_ONLY, pos, frag);

                while (buf.hasRemaining()) {
                    if (buf.get() == valor) contador++;
                }
            }
        }
        return contador;
    }

    public static void main(String[] args) throws IOException {
        Path archivo = Paths.get("archivo-5gb.bin");
        if (Files.exists(archivo)) {
            System.out.println("Contando bytes 0xFF...");
            long inicio = System.currentTimeMillis();
            long contador = contarByte(archivo, (byte) 0xFF);
            long tiempo = System.currentTimeMillis() - inicio;
            System.out.printf("Encontrados: %d en %d ms%n", contador, tiempo);
        }
    }
}
```

---

## 6.9 I/O Asíncrona y No Bloqueante

### 6.9.1 ¿Qué es I/O asíncrona?

En I/O **síncrona**, el thread se bloquea esperando que la operación termine. En I/O **asíncrona**, el thread continúa inmediatamente y recibe una notificación cuando los datos están disponibles.

> [!NOTE]
> ### 📟 La Llamada en Espera vs. El Localizador Vibratorio de Restaurante (I/O Síncrona vs. Asíncrona)
>
> Imagina que tienes muchísima hambre y vas a pedir comida para llevar:
> - **Enfoque Síncrono (La Llamada en Espera)**: Llegas al mostrador, ordenas tu plato y **te quedas parado frente al cajero con los brazos cruzados esperando a que lo cocinen** (tu hilo de ejecución se bloquea). Durante los 15 minutos que toma preparar la comida, no puedes ir al baño, no puedes contestar llamadas, ni hacer otra cosa. Estás "congelado".
> - **Enfoque Asíncrono (El Localizador Vibratorio - `Future` / `CompletionHandler`)**: Ordenas tu comida y el cajero te entrega un **pequeño localizador vibratorio (un `CompletionHandler` o callback)** y te dice: *"Sigue con tus actividades"*.
>   - Tú te vas a sentar, revisas tus correos, hablas por teléfono o lees un libro (el hilo principal sigue libre haciendo otros trabajos).
>   - En cuanto tu platillo está listo en la cocina, el localizador de tu bolsillo **vibra y parpadea (el callback se dispara)**, indicándote que puedes recoger los datos listos sin haber perdido ni un solo segundo de tu tiempo útil de pie en el mostrador.
>
> **En resumen**: La I/O asíncrona permite que tus aplicaciones inicien operaciones pesadas de lectura o escritura en archivos o redes y continúen haciendo otros trabajos de inmediato, recibiendo una notificación automática de vuelta sólo cuando los bytes han sido transferidos con éxito.

```
┌─────────────────────────────────────────────────────────────────┐
│   I/O SÍNCRONA (bloqueante)                                      │
│   Thread ──▶ read() ──▶ ESPERA ──▶ datos ──▶ continúa          │
│            ╚═══════════ thread bloqueado ═══════════╝            │
├─────────────────────────────────────────────────────────────────┤
│   I/O ASÍNCRONA (no bloqueante)                                  │
│   Thread ──▶ read() ──▶ continúa inmediatamente                  │
│                 │                                                │
│                 └──▶ callback/future cuando los datos estén      │
└─────────────────────────────────────────────────────────────────┘
```

### 6.9.2 `AsynchronousFileChannel` (Java 7+)

Java 7 introdujo `AsynchronousFileChannel` que permite leer y escribir archivos sin bloquear el thread actual:

```java
import java.nio.*;
import java.nio.channels.*;
import java.nio.file.*;
import java.nio.charset.*;
import java.util.concurrent.*;

public class AsyncChannelBasico {
    public static void main(String[] args) throws Exception {
        Path archivo = Paths.get("async-test.txt");
        Files.writeString(archivo, "Hola, I/O Asíncrona en Java!\nSegunda línea\nTercera línea");

        try (AsynchronousFileChannel channel = AsynchronousFileChannel.open(
                archivo, StandardOpenOption.READ)) {

            // ENFOQUE 1: FUTURE
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            Future<Integer> result = channel.read(buffer, 0); // Lee desde posición 0

            System.out.println("Lectura iniciada... mientras tanto, hago otras cosas");
            System.out.println("(el thread principal NO está bloqueado)");

            // Hacer otro trabajo mientras la lectura ocurre en background
            Thread.sleep(10);

            // Ahora sí, esperar el resultado
            Integer bytesRead = result.get();  // bloquea solo si aún no terminó
            buffer.flip();
            System.out.println("Leídos " + bytesRead + " bytes:");
            System.out.println("Contenido:\n" + StandardCharsets.UTF_8.decode(buffer));
        }
    }
}
```

### 6.9.3 `CompletionHandler`: el enfoque con callback

En lugar de esperar con un `Future`, el `CompletionHandler` recibe un callback cuando la operación termina:

```java
import java.nio.*;
import java.nio.channels.*;
import java.nio.file.*;
import java.nio.charset.*;
import java.util.concurrent.CountDownLatch;

public class AsyncChannelCallback {
    public static void main(String[] args) throws Exception {
        Path archivo = Paths.get("async-callback.txt");
        Files.writeString(archivo, "Datos para leer asíncronamente...\nMás datos.");

        CountDownLatch latch = new CountDownLatch(1);

        try (AsynchronousFileChannel channel = AsynchronousFileChannel.open(
                archivo, StandardOpenOption.READ)) {

            ByteBuffer buffer = ByteBuffer.allocate(1024);

            channel.read(buffer, 0, buffer, new CompletionHandler<>() {
                @Override
                public void completed(Integer bytesRead, ByteBuffer attachment) {
                    System.out.println("callback: Lectura completada, " + bytesRead + " bytes.");
                    attachment.flip();
                    String contenido = StandardCharsets.UTF_8.decode(attachment).toString();
                    System.out.println("callback: Contenido = " + contenido.trim());
                    latch.countDown();
                }

                @Override
                public void failed(Throwable exc, ByteBuffer attachment) {
                    System.err.println("callback: Error: " + exc.getMessage());
                    latch.countDown();
                }
            });

            System.out.println("main: Lectura asíncrona disparada. Sigo trabajando...");

            // Simular otro trabajo
            for (int i = 0; i < 5; i++) {
                System.out.println("main: trabajando... " + i);
                Thread.sleep(100);
            }

            latch.await();
            System.out.println("main: Todo terminó.");
        }
    }
}
```

### 6.9.4 Lector asíncrono completo con callback

Un ejemplo más realista que lee un archivo completo asíncronamente, fragmento por fragmento:

```java
import java.nio.*;
import java.nio.channels.*;
import java.nio.file.*;
import java.nio.charset.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class LectorAsincronoCompleto {

    private static class AsyncFileReader {
        private final AsynchronousFileChannel channel;
        private final long fileSize;
        private final StringBuilder contenido = new StringBuilder();
        private final AtomicLong posicion = new AtomicLong(0);
        private final CompletableFuture<String> future = new CompletableFuture<>();
        private static final int BUFFER_SIZE = 4096;

        public AsyncFileReader(Path path) throws Exception {
            this.fileSize = Files.size(path);
            this.channel = AsynchronousFileChannel.open(path, StandardOpenOption.READ);
        }

        public CompletableFuture<String> readAll() {
            leerSiguienteFragmento();
            return future;
        }

        private void leerSiguienteFragmento() {
            ByteBuffer buffer = ByteBuffer.allocate(BUFFER_SIZE);
            long currentPos = posicion.get();

            if (currentPos >= fileSize) {
                future.complete(contenido.toString());
                try { channel.close(); } catch (Exception e) { /* ignore */ }
                return;
            }

            channel.read(buffer, currentPos, buffer, new CompletionHandler<>() {
                @Override
                public void completed(Integer bytesRead, ByteBuffer buf) {
                    buf.flip();
                    contenido.append(StandardCharsets.UTF_8.decode(buf));
                    posicion.addAndGet(bytesRead);
                    leerSiguienteFragmento();  // Leer siguiente fragmento
                }

                @Override
                public void failed(Throwable exc, ByteBuffer buf) {
                    future.completeExceptionally(exc);
                    try { channel.close(); } catch (Exception e) { /* ignore */ }
                }
            });
        }
    }

    public static void main(String[] args) throws Exception {
        Path testFile = Paths.get("async-lector-test.txt");
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 100; i++) {
            sb.append("Línea ").append(i).append(": contenido de prueba\n");
        }
        Files.writeString(testFile, sb.toString());

        AsyncFileReader reader = new AsyncFileReader(testFile);

        System.out.println("Iniciando lectura asíncrona...");
        reader.readAll()
            .thenAccept(contenido -> {
                System.out.println("=== Lectura completada ===");
                System.out.println("Total caracteres leídos: " + contenido.length());
                System.out.println("Primeras 2 líneas:");
                contenido.lines().limit(2).forEach(System.out::println);
            })
            .exceptionally(ex -> {
                System.err.println("Error: " + ex.getMessage());
                return null;
            })
            .get();

        Files.delete(testFile);
    }
}
```

### 6.9.5 Comparación: I/O síncrona vs asíncrona

| | **Síncrona** | **Asíncrona** |
|---|-------------|---------------|
| **Bloqueo** | El thread se bloquea hasta que los datos están disponibles | El thread continúa inmediatamente |
| **Complejidad** | Simple, lineal | Más compleja (callbacks/futures) |
| **Uso de threads** | Un thread por operación | Múltiples operaciones comparten pocos threads |
| **Rendimiento** | Bueno para pocas conexiones | Excelente para muchas conexiones concurrentes |
| **Debugging** | Fácil (stack trace lineal) | Difícil (callbacks anidados) |
| **Cuándo usar** | Aplicaciones simples, scripts | Servidores de alto rendimiento, miles de conexiones |

> **Regla práctica:** usa I/O asíncrona cuando necesites manejar cientos o miles de operaciones concurrentes sin crear un thread por cada una. Para aplicaciones simples, la I/O síncrona es más fácil de escribir y mantener.

---

## 6.10 Compresión y archivos JAR/ZIP

### 6.10.1 `ZipInputStream` / `ZipOutputStream`

```java
import java.io.*;
import java.nio.file.*;
import java.util.zip.*;

public class CompresionZip {

    // Crear un archivo ZIP con varios archivos
    public static void crearZip(Path zipPath, Path... archivos) throws IOException {
        try (ZipOutputStream zos = new ZipOutputStream(
                 new BufferedOutputStream(Files.newOutputStream(zipPath)))) {

            for (Path archivo : archivos) {
                ZipEntry entry = new ZipEntry(archivo.getFileName().toString());
                zos.putNextEntry(entry);
                Files.copy(archivo, zos);
                zos.closeEntry();
                System.out.println("Agregado: " + archivo.getFileName());
            }
        }
    }

    // Leer un archivo ZIP y listar su contenido
    public static void leerZip(Path zipPath) throws IOException {
        try (ZipInputStream zis = new ZipInputStream(
                 new BufferedInputStream(Files.newInputStream(zipPath)))) {

            ZipEntry entry;
            while ((entry = zis.getNextEntry()) != null) {
                String nombre = entry.getName();
                long tamanoComprimido = entry.getCompressedSize();
                long tamanoOriginal = entry.getSize();
                boolean esDirectorio = entry.isDirectory();

                System.out.printf("%s %-40s %10d -> %10d bytes%n",
                    esDirectorio ? "[DIR]" : "[FIL]",
                    nombre,
                    tamanoOriginal > 0 ? tamanoOriginal : 0,
                    tamanoComprimido > 0 ? tamanoComprimido : 0);

                zis.closeEntry();
            }
        }
    }

    // Extraer un archivo ZIP completo
    public static void extraerZip(Path zipPath, Path destinoDir) throws IOException {
        Files.createDirectories(destinoDir);

        try (ZipInputStream zis = new ZipInputStream(
                 new BufferedInputStream(Files.newInputStream(zipPath)))) {

            ZipEntry entry;
            while ((entry = zis.getNextEntry()) != null) {
                Path archivoDestino = destinoDir.resolve(entry.getName());

                if (entry.isDirectory()) {
                    Files.createDirectories(archivoDestino);
                } else {
                    Files.createDirectories(archivoDestino.getParent());
                    Files.copy(zis, archivoDestino, StandardCopyOption.REPLACE_EXISTING);
                    System.out.println("Extraído: " + entry.getName());
                }
                zis.closeEntry();
            }
        }
    }

    // Comprimir un directorio completo recursivamente
    public static void comprimirDirectorio(Path dirOrigen, Path zipDestino) throws IOException {
        try (ZipOutputStream zos = new ZipOutputStream(
                 new BufferedOutputStream(Files.newOutputStream(zipDestino)))) {

            Files.walk(dirOrigen)
                 .filter(path -> !Files.isDirectory(path))
                 .forEach(path -> {
                     try {
                         String nombreEntrada = dirOrigen.relativize(path).toString();
                         zos.putNextEntry(new ZipEntry(nombreEntrada));
                         Files.copy(path, zos);
                         zos.closeEntry();
                     } catch (IOException e) {
                         throw new UncheckedIOException(e);
                     }
                 });
        }
    }

    public static void main(String[] args) throws IOException {
        Path dir = Paths.get("prueba-zip");
        Files.createDirectories(dir);
        Files.writeString(dir.resolve("a.txt"), "Contenido del archivo A");
        Files.writeString(dir.resolve("b.txt"), "Contenido del archivo B");
        Files.writeString(dir.resolve("c.txt"), "Contenido del archivo C");

        crearZip(Paths.get("prueba.zip"),
            dir.resolve("a.txt"), dir.resolve("b.txt"), dir.resolve("c.txt"));

        System.out.println("\n=== Contenido del ZIP ===");
        leerZip(Paths.get("prueba.zip"));

        System.out.println("\n=== Extrayendo ZIP ===");
        extraerZip(Paths.get("prueba.zip"), Paths.get("extraido"));
    }
}
```

### 6.10.2 Zipping con protección contra Zip Bomb

```java
import java.io.*;
import java.nio.file.*;
import java.util.zip.*;

public class ZipAvanzado {

    // Extraer ZIP con protección contra Zip Bomb y Zip Slip
    public static void extraerZipSeguro(Path zipPath, Path destinoDir) throws IOException {
        Files.createDirectories(destinoDir);

        try (ZipInputStream zis = new ZipInputStream(
                 new BufferedInputStream(Files.newInputStream(zipPath)))) {

            ZipEntry entry;
            long totalExtraido = 0;
            long MAX_EXPANDIDO = 100 * 1024 * 1024; // 100 MB máximo

            while ((entry = zis.getNextEntry()) != null) {
                Path archivoDestino = destinoDir.resolve(entry.getName());

                // Protección contra Zip Slip: no permitir rutas que escapen del directorio
                if (!archivoDestino.normalize().startsWith(destinoDir.normalize())) {
                    throw new IOException("Entrada ZIP maliciosa: " + entry.getName());
                }

                if (entry.isDirectory()) {
                    Files.createDirectories(archivoDestino);
                } else {
                    Files.createDirectories(archivoDestino.getParent());

                    totalExtraido += Files.copy(zis, archivoDestino,
                        StandardCopyOption.REPLACE_EXISTING);

                    if (totalExtraido > MAX_EXPANDIDO) {
                        throw new IOException(
                            "Límite de extracción excedido. Posible Zip Bomb.");
                    }
                }
                zis.closeEntry();
            }
        }
    }
}
```

### 6.10.3 `JarFile` y `JarEntry`: leer archivos JAR

Un archivo JAR (Java ARchive) es un ZIP con un `META-INF/MANIFEST.MF` especial:

```java
import java.io.*;
import java.nio.file.*;
import java.util.*;
import java.util.jar.*;

public class LectorJar {

    public static void listarContenidoJar(Path jarPath) throws IOException {
        try (JarFile jar = new JarFile(jarPath.toFile())) {
            System.out.println("=== Contenido de: " + jarPath.getFileName() + " ===");

            // Leer el MANIFEST.MF
            Manifest manifest = jar.getManifest();
            if (manifest != null) {
                System.out.println("\n--- MANIFEST.MF ---");
                Attributes attrs = manifest.getMainAttributes();
                for (Map.Entry<Object, Object> entry : attrs.entrySet()) {
                    System.out.println(entry.getKey() + ": " + entry.getValue());
                }
            }

            // Listar todas las entradas
            System.out.println("\n--- Entradas ---");
            Enumeration<JarEntry> entries = jar.entries();
            while (entries.hasMoreElements()) {
                JarEntry entry = entries.nextElement();
                long tam = entry.getSize();
                long comp = entry.getCompressedSize();

                System.out.printf("%s %-50s %10d -> %10d bytes%n",
                    entry.isDirectory() ? "[DIR]" : "[FIL]",
                    entry.getName(),
                    tam > 0 ? tam : 0,
                    comp > 0 ? comp : 0);
            }
        }
    }

    // Buscar una clase específica dentro de un JAR
    public static void buscarClaseEnJar(Path jarPath, String nombreClase) throws IOException {
        try (JarFile jar = new JarFile(jarPath.toFile())) {
            String busqueda = nombreClase.replace('.', '/') + ".class";

            JarEntry entry = jar.getJarEntry(busqueda);
            if (entry != null) {
                System.out.println("Clase encontrada: " + entry.getName());
                System.out.println("  Tamaño: " + entry.getSize() + " bytes");
                System.out.println("  CRC: " + Long.toHexString(entry.getCrc()));
            } else {
                System.out.println("Clase no encontrada: " + nombreClase);
            }
        }
    }

    // Extraer una entrada específica
    public static void extraerEntrada(Path jarPath, String nombreEntrada,
                                       Path destino) throws IOException {
        try (JarFile jar = new JarFile(jarPath.toFile())) {
            JarEntry entry = jar.getJarEntry(nombreEntrada);

            if (entry != null) {
                Files.copy(jar.getInputStream(entry), destino,
                    StandardCopyOption.REPLACE_EXISTING);
                System.out.println("Extraído: " + nombreEntrada + " -> " + destino);
            }
        }
    }

    public static void main(String[] args) throws IOException {
        Path jarPath = Paths.get(System.getProperty("java.home"),
            "lib", "jrt-fs.jar");
        if (Files.exists(jarPath)) {
            listarContenidoJar(jarPath);
            buscarClaseEnJar(jarPath, "java.lang.String");
        }
    }
}
```

### 6.10.4 `GZIPInputStream` / `GZIPOutputStream`

Para compresión GZIP (un solo archivo, muy común en Unix):

```java
import java.io.*;
import java.nio.file.*;

public class CompresionGzip {

    public static void comprimirGzip(Path origen, Path destinoGz) throws IOException {
        try (GZIPOutputStream gzos = new GZIPOutputStream(
                 Files.newOutputStream(destinoGz));
             FileInputStream fis = new FileInputStream(origen.toFile())) {
            fis.transferTo(gzos);
        }
    }

    public static void descomprimirGzip(Path origenGz, Path destino) throws IOException {
        try (GZIPInputStream gzis = new GZIPInputStream(
                 Files.newInputStream(origenGz));
             FileOutputStream fos = new FileOutputStream(destino.toFile())) {
            gzis.transferTo(fos);
        }
    }

    public static void descomprimirGzipALineas(Path origenGz) throws IOException {
        try (GZIPInputStream gzis = new GZIPInputStream(
                 Files.newInputStream(origenGz));
             BufferedReader reader = new BufferedReader(
                 new InputStreamReader(gzis))) {

            String linea;
            while ((linea = reader.readLine()) != null) {
                System.out.println(linea);
            }
        }
    }
}
```

---

## 6.11 Comunicación de red con Sockets

### 6.11.1 Socket y ServerSocket básicos

```
┌──────────────────────────────────────────────────────────────────┐
│   COMUNICACIÓN CLIENTE-SERVIDOR CON SOCKETS                       │
│                                                                   │
│   ┌────────────┐                           ┌────────────┐        │
│   │  Servidor  │                           │  Cliente   │        │
│   │            │                           │            │        │
│   │ ServerSocket│                          │  Socket    │        │
│   │ accept()   │◀──── conexión TCP ────────│ connect()  │        │
│   │            │                           │            │        │
│   │ InputStream│◀──── datos (bytes) ───────│OutputStream│        │
│   │OutputStream│───── datos (bytes) ──────▶│InputStream │        │
│   └────────────┘                           └────────────┘        │
│                                                                   │
│   Puerto: 8080                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 6.11.2 Servidor HTTP mínimo (devuelve "Hello World" al navegador)

```java
import java.io.*;
import java.net.*;
import java.nio.charset.StandardCharsets;
import java.time.LocalDateTime;

public class ServidorHttpMinimo {
    private static final int PUERTO = 8080;

    public static void main(String[] args) {
        try (ServerSocket serverSocket = new ServerSocket(PUERTO)) {
            System.out.println("Servidor HTTP iniciado en http://localhost:" + PUERTO);
            System.out.println("Abre tu navegador en esa URL. Ctrl+C para detener.");

            while (true) {
                try (Socket cliente = serverSocket.accept()) {
                    manejarPeticion(cliente);
                } catch (IOException e) {
                    System.err.println("Error manejando cliente: " + e.getMessage());
                }
            }
        } catch (IOException e) {
            System.err.println("Error iniciando servidor: " + e.getMessage());
        }
    }

    private static void manejarPeticion(Socket cliente) throws IOException {
        BufferedReader reader = new BufferedReader(
            new InputStreamReader(cliente.getInputStream(), StandardCharsets.UTF_8));
        OutputStream out = cliente.getOutputStream();

        // Leer la primera línea de la petición HTTP
        String requestLine = reader.readLine();
        if (requestLine == null) return;
        System.out.println("[" + LocalDateTime.now() + "] " +
            cliente.getInetAddress().getHostAddress() + " - " + requestLine);

        // Leer headers (ignorarlos para este ejemplo)
        String header;
        while ((header = reader.readLine()) != null && !header.isEmpty()) {
            // System.out.println("  Header: " + header);
        }

        // Construir respuesta HTTP
        String body = """
            <!DOCTYPE html>
            <html>
            <head><meta charset="UTF-8"><title>Servidor Java</title></head>
            <body>
                <h1>¡Hola Mundo desde Java!</h1>
                <p>Servidor HTTP funcionando en el puerto """ + PUERTO + """</p>
                <p>Hora del servidor: """ + LocalDateTime.now() + """</p>
                <pre>Ruta solicitada: """ + requestLine + """</pre>
            </body>
            </html>""";

        String response = "HTTP/1.1 200 OK\r\n" +
            "Content-Type: text/html; charset=UTF-8\r\n" +
            "Content-Length: " + body.getBytes(StandardCharsets.UTF_8).length + "\r\n" +
            "Connection: close\r\n" +
            "\r\n" +
            body;

        out.write(response.getBytes(StandardCharsets.UTF_8));
        out.flush();
    }
}
```

### 6.11.3 Cliente HTTP con `java.net.Socket`

```java
import java.io.*;
import java.net.*;
import java.nio.charset.StandardCharsets;

public class ClienteHttpGet {
    public static void main(String[] args) throws IOException {
        String host = "example.com";
        int port = 80;

        try (Socket socket = new Socket(host, port);
             PrintWriter writer = new PrintWriter(
                 new OutputStreamWriter(socket.getOutputStream(), StandardCharsets.UTF_8), true);
             BufferedReader reader = new BufferedReader(
                 new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8))) {

            // Enviar petición HTTP GET
            writer.println("GET / HTTP/1.1");
            writer.println("Host: " + host);
            writer.println("User-Agent: JavaSocketClient/1.0");
            writer.println("Connection: close");
            writer.println();  // Línea en blanco para terminar headers

            // Leer respuesta
            System.out.println("=== Respuesta ===");
            String line;
            boolean headersDone = false;
            while ((line = reader.readLine()) != null) {
                if (!headersDone && line.isEmpty()) {
                    headersDone = true;
                    System.out.println("--- Body ---");
                    continue;
                }
                System.out.println(line);
            }
        }
    }
}
```

### 6.11.4 Servidor multi-cliente con threads

```java
import java.io.*;
import java.net.*;
import java.nio.charset.StandardCharsets;
import java.util.concurrent.*;

public class ServidorMultiCliente {
    private static final int PUERTO = 9000;
    private static final ExecutorService pool = Executors.newFixedThreadPool(10);

    public static void main(String[] args) throws IOException {
        try (ServerSocket server = new ServerSocket(PUERTO)) {
            System.out.println("Servidor multi-cliente en puerto " + PUERTO);

            while (true) {
                Socket cliente = server.accept();
                pool.submit(() -> manejarCliente(cliente));
            }
        }
    }

    private static void manejarCliente(Socket cliente) {
        try (cliente;
             BufferedReader in = new BufferedReader(
                 new InputStreamReader(cliente.getInputStream(), StandardCharsets.UTF_8));
             PrintWriter out = new PrintWriter(
                 new OutputStreamWriter(cliente.getOutputStream(), StandardCharsets.UTF_8), true)) {

            out.println("Bienvenido al servidor. Escribe 'salir' para desconectarte.");

            String mensaje;
            while ((mensaje = in.readLine()) != null) {
                System.out.println("[" + Thread.currentThread().getName() +
                    "] Recibido: " + mensaje);

                if ("salir".equalsIgnoreCase(mensaje.trim())) {
                    out.println("¡Adiós!");
                    break;
                }

                out.println("ECO: " + mensaje);
            }
        } catch (IOException e) {
            System.err.println("Error con cliente: " + e.getMessage());
        }
    }
}
```

### 6.11.5 El nuevo HTTP Client (Java 11+)

Java 11 introdujo `java.net.http.HttpClient`, una API moderna que reemplaza a `HttpURLConnection`:

```java
import java.net.URI;
import java.net.http.*;
import java.net.http.HttpResponse.BodyHandlers;
import java.io.IOException;
import java.util.concurrent.CompletableFuture;

public class HttpClientModerno {

    // SÍNCRONO: bloquea hasta recibir respuesta
    public static void peticionSincrona() throws Exception {
        HttpClient client = HttpClient.newHttpClient();

        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://jsonplaceholder.typicode.com/posts/1"))
            .header("Accept", "application/json")
            .GET()
            .build();

        HttpResponse<String> response = client.send(request, BodyHandlers.ofString());

        System.out.println("Status: " + response.statusCode());
        System.out.println("Headers: " + response.headers().map());
        System.out.println("Body: " + response.body());
    }

    // ASÍNCRONO: con callbacks vía CompletableFuture
    public static void peticionAsincrona() {
        HttpClient client = HttpClient.newHttpClient();

        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://jsonplaceholder.typicode.com/posts/2"))
            .GET()
            .build();

        CompletableFuture<HttpResponse<String>> future =
            client.sendAsync(request, BodyHandlers.ofString());

        System.out.println("Petición enviada... mientras tanto sigo trabajando");

        future.thenAccept(response -> {
            System.out.println("\n=== Respuesta asíncrona ===");
            System.out.println("Status: " + response.statusCode());
            System.out.println("Body: " + response.body());
        }).join();  // Esperar (o continuar sin bloquear en una app real)
    }

    // POST con cuerpo JSON
    public static void peticionPOST() throws Exception {
        HttpClient client = HttpClient.newHttpClient();

        String jsonBody = """
            {
                "title": "Nuevo post desde Java",
                "body": "Contenido del post",
                "userId": 1
            }""";

        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://jsonplaceholder.typicode.com/posts"))
            .header("Content-Type", "application/json")
            .POST(HttpRequest.BodyPublishers.ofString(jsonBody))
            .build();

        HttpResponse<String> response = client.send(request, BodyHandlers.ofString());

        System.out.println("POST Status: " + response.statusCode());
        System.out.println("POST Body: " + response.body());
    }

    // Configuración avanzada del cliente
    public static void clienteAvanzado() throws Exception {
        HttpClient client = HttpClient.newBuilder()
            .connectTimeout(java.time.Duration.ofSeconds(10))
            .followRedirects(HttpClient.Redirect.NORMAL)
            .version(HttpClient.Version.HTTP_2)  // HTTP/2 si el servidor lo soporta
            .build();

        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://httpbin.org/get"))
            .timeout(java.time.Duration.ofSeconds(30))
            .GET()
            .build();

        HttpResponse<String> response = client.send(request, BodyHandlers.ofString());
        System.out.println("HTTP Version: " + response.version());
        System.out.println("Body: " + response.body());
    }

    public static void main(String[] args) throws Exception {
        peticionSincrona();
        peticionAsincrona();
        peticionPOST();
        clienteAvanzado();
    }
}
```

### 6.11.6 WebSockets básicos con el HTTP Client (Java 11+)

```java
import java.net.URI;
import java.net.http.*;
import java.util.concurrent.*;
import java.nio.charset.StandardCharsets;

public class WebSocketDemo {

    public static void main(String[] args) throws Exception {
        HttpClient client = HttpClient.newHttpClient();

        // Conectar a un WebSocket público de prueba
        CompletableFuture<WebSocket> wsFuture = client.newWebSocketBuilder()
            .buildAsync(URI.create("wss://echo.websocket.org"), new WebSocket.Listener() {

                @Override
                public void onOpen(WebSocket webSocket) {
                    System.out.println("WebSocket conectado!");
                    webSocket.sendText("¡Hola desde Java!", true);
                    WebSocket.Listener.super.onOpen(webSocket);
                }

                @Override
                public CompletionStage<?> onText(WebSocket webSocket,
                        CharSequence data, boolean last) {
                    System.out.println("Mensaje recibido: " + data);
                    // Cerrar tras recibir el eco
                    webSocket.sendClose(WebSocket.NORMAL_CLOSURE, "ok");
                    return WebSocket.Listener.super.onText(webSocket, data, last);
                }

                @Override
                public CompletionStage<?> onClose(WebSocket webSocket,
                        int statusCode, String reason) {
                    System.out.println("WebSocket cerrado: " + statusCode + " " + reason);
                    return WebSocket.Listener.super.onClose(webSocket, statusCode, reason);
                }

                @Override
                public void onError(WebSocket webSocket, Throwable error) {
                    System.err.println("Error: " + error.getMessage());
                }
            });

        WebSocket ws = wsFuture.get(5, TimeUnit.SECONDS);
        // Esperar un poco para recibir el eco
        Thread.sleep(3000);
    }
}
```

---

## 6.12 Archivos temporales

### 6.12.1 `File.createTempFile()` (java.io clásico)

```java
import java.io.File;
import java.io.IOException;

public class ArchivosTemporalesClasico {
    public static void main(String[] args) throws IOException {
        // Crear archivo temporal en el directorio por defecto (/tmp en Unix)
        File tempFile = File.createTempFile("prefijo_", ".tmp");
        System.out.println("Archivo temporal: " + tempFile.getAbsolutePath());

        // Escribir datos
        java.nio.file.Files.writeString(tempFile.toPath(), "Datos temporales...");

        // Leer
        String contenido = java.nio.file.Files.readString(tempFile.toPath());
        System.out.println("Contenido: " + contenido);

        // Marcar para eliminación al salir de la JVM
        tempFile.deleteOnExit();
        System.out.println("Marcado para eliminar al salir (deleteOnExit).");

        // O eliminar manualmente
        // tempFile.delete();
    }
}
```

### 6.12.2 `Files.createTempFile()` (NIO.2)

```java
import java.nio.file.*;
import java.io.IOException;

public class ArchivosTemporalesNIO {

    public static void main(String[] args) throws IOException {
        // Crear archivo temporal en el directorio por defecto
        Path tempFile = Files.createTempFile("miApp_", ".dat");
        System.out.println("Archivo temporal: " + tempFile.toAbsolutePath());

        // Escribir y leer
        Files.writeString(tempFile, "Datos de sesión temporal");
        System.out.println("Contenido: " + Files.readString(tempFile));

        // Crear archivo temporal en un directorio específico
        Path dirPersonalizado = Paths.get("temp-files");
        Files.createDirectories(dirPersonalizado);

        Path tempEnDir = Files.createTempFile(dirPersonalizado, "app_", ".tmp");
        System.out.println("En dir personalizado: " + tempEnDir.toAbsolutePath());

        // Garantizar eliminación con deleteOnExit o try-finally
        tempFile.toFile().deleteOnExit();
        tempEnDir.toFile().deleteOnExit();
    }
}
```

### 6.12.3 Directorios temporales con `Files.createTempDirectory()`

```java
import java.nio.file.*;
import java.io.IOException;

public class DirectoriosTemporales {

    public static void main(String[] args) throws IOException {
        // Crear directorio temporal
        Path tempDir = Files.createTempDirectory("miApp_");
        System.out.println("Directorio temporal: " + tempDir.toAbsolutePath());

        // Crear archivos dentro del directorio temporal
        Path archivo1 = tempDir.resolve("datos.dat");
        Path archivo2 = tempDir.resolve("config.properties");

        Files.writeString(archivo1, "Contenido 1");
        Files.writeString(archivo2, "clave=valor");

        System.out.println("Contenido del directorio temporal:");
        try (var stream = Files.list(tempDir)) {
            stream.forEach(p -> System.out.println("  " + p.getFileName()));
        }

        // Marcar directorio para eliminación recursiva al salir
        // Nota: deleteOnExit NO funciona para directorios no vacíos
        // Solución: cleanup hook con FileVisitor (ver 6.7.6)
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            try {
                borrarRecursivo(tempDir);
                System.out.println("Directorio temporal eliminado: " + tempDir.getFileName());
            } catch (IOException e) {
                System.err.println("Error limpiando temporal: " + e.getMessage());
            }
        }));

        System.out.println("Directorio marcado para limpieza al salir.");
    }

    private static void borrarRecursivo(Path dir) throws IOException {
        if (Files.exists(dir)) {
            Files.walkFileTree(dir, new SimpleFileVisitor<>() {
                @Override
                public FileVisitResult visitFile(Path file, BasicFileAttributes attrs)
                        throws IOException {
                    Files.delete(file);
                    return FileVisitResult.CONTINUE;
                }
                @Override
                public FileVisitResult postVisitDirectory(Path dir, IOException exc)
                        throws IOException {
                    if (exc != null) throw exc;
                    Files.delete(dir);
                    return FileVisitResult.CONTINUE;
                }
            });
        }
    }
}
```

### 6.12.4 Garantizar eliminación: patrones

```java
import java.nio.file.*;
import java.io.IOException;

public class GarantizarEliminacion {

    // Patrón 1: try-finally explícito (más seguro que deleteOnExit)
    public static void conTryFinally() {
        Path tempFile = null;
        try {
            tempFile = Files.createTempFile("app_", ".tmp");
            Files.writeString(tempFile, "Datos de trabajo...");

            // ... procesamiento ...

        } catch (IOException e) {
            System.err.println("Error: " + e.getMessage());
        } finally {
            if (tempFile != null) {
                try {
                    Files.deleteIfExists(tempFile);
                    System.out.println("Archivo temporal eliminado.");
                } catch (IOException e) {
                    System.err.println("No se pudo eliminar: " + e.getMessage());
                }
            }
        }
    }

    // Patrón 2: Shutdown hook (para limpieza garantizada incluso en errores)
    public static void conShutdownHook() throws IOException {
        Path tempDir = Files.createTempDirectory("app_");

        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("Shutdown hook: limpiando " + tempDir);
            // La lógica de eliminación recursiva (ver ejemplo anterior)
        }));

        // Si la app termina normalmente:
        Files.delete(tempDir); // Como es temporal, debería estar vacío
    }

    // Patrón 3: Usar StandardOpenOption.DELETE_ON_CLOSE (Java 7+)
    public static void conDeleteOnClose() throws IOException {
        Path tempFile = Files.createTempFile("app_", ".tmp");

        try (var channel = java.nio.channels.FileChannel.open(tempFile,
                StandardOpenOption.WRITE,
                StandardOpenOption.DELETE_ON_CLOSE)) {
            // El archivo se elimina automáticamente al cerrar el canal
            java.nio.ByteBuffer buf = java.nio.ByteBuffer.wrap("datos".getBytes());
            channel.write(buf);
            System.out.println("Datos escritos en temporal con DELETE_ON_CLOSE");
        }
        // tempFile ya fue eliminado automáticamente
        System.out.println("¿Existe aún? " + Files.exists(tempFile)); // false
    }

    public static void main(String[] args) throws IOException {
        conTryFinally();
        conDeleteOnClose();
    }
}
```

> **Recomendación:** Prefiere `StandardOpenOption.DELETE_ON_CLOSE` para archivos temporales de corta vida. Para directorios temporales, usa un shutdown hook con eliminación recursiva.

---

## 6.13 I/O de alto rendimiento

### 6.13.1 Técnicas de buffering óptimo

El tamaño del buffer es una de las decisiones más importantes para el rendimiento de I/O:

```
┌──────────────────────────────────────────────────────────────────┐
│   TAMAÑO DE BUFFER vs RENDIMIENTO                                 │
│                                                                   │
│   Rendimiento                                                     │
│       ▲                                                           │
│       │          ╭────────────────────────────╮                  │
│       │        ╱                              ╲                  │
│       │      ╱                                  ╲                │
│       │    ╱                                      ╲              │
│       │  ╱                                          ╲            │
│       │╱                                              ╲          │
│       └──────────────────────────────────────────────────▶      │
│        1B   1KB  4KB 8KB 16KB  32KB  64KB 128KB 256KB           │
│                          ▲                                       │
│                          │                                       │
│                  Punto óptimo (8-32 KB para discos)              │
│                                                                   │
│   ⚠ Demasiado pequeño → demasiadas llamadas al SO                │
│   ⚠ Demasiado grande  → presión sobre el GC, menor cache hit    │
└──────────────────────────────────────────────────────────────────┘
```

```java
import java.io.*;
import java.nio.file.*;

public class BufferingOptimo {

    // Benchmark para encontrar el tamaño de buffer óptimo
    public static long benchmarkBuffer(Path archivo, int tamanoBuffer) throws IOException {
        long inicio = System.nanoTime();
        long totalBytes = 0;

        byte[] buffer = new byte[tamanoBuffer];
        try (FileInputStream fis = new FileInputStream(archivo.toFile())) {
            int leidos;
            while ((leidos = fis.read(buffer)) != -1) {
                totalBytes += leidos;
                // Simulamos procesamiento ligero
                for (int i = 0; i < leidos; i++) {
                    // NOP: solo medimos el costo de I/O + acceso al array
                }
            }
        }

        return System.nanoTime() - inicio;
    }

    public static void main(String[] args) throws IOException {
        Path archivo = Paths.get("/tmp/test-buffer.dat");

        // Crear archivo de prueba de 200 MB
        if (!Files.exists(archivo)) {
            System.out.println("Creando archivo de prueba de 200 MB...");
            byte[] bloque = new byte[1024 * 1024];
            try (FileOutputStream fos = new FileOutputStream(archivo.toFile())) {
                for (int i = 0; i < 200; i++) {
                    fos.write(bloque);
                }
            }
        }

        int[] tamanos = { 512, 1024, 2048, 4096, 8192, 16384, 32768, 65536, 131072 };

        System.out.println("Tamaño buffer | Tiempo (ms) | Rendimiento (MB/s)");
        System.out.println("-------------|-------------|-------------------");

        for (int tam : tamanos) {
            // Warmup
            benchmarkBuffer(archivo, tam);

            // Medición real
            long nanos = benchmarkBuffer(archivo, tam);
            double ms = nanos / 1_000_000.0;
            double mbps = (200.0) / (ms / 1000.0);

            System.out.printf("%12d | %10.1f | %17.1f%n", tam, ms, mbps);
        }

        Files.deleteIfExists(archivo);
    }
}
```

### 6.13.2 Transferencia directa con `FileChannel.transferTo/transferFrom`

Java NIO ofrece transferencia directa entre canales usando el DMA del sistema operativo, evitando copias a nivel de usuario:

```
┌──────────────────────────────────────────────────────────────────┐
│   transferTo / transferFrom (zero-copy)                           │
│                                                                   │
│   ┌────────┐              ┌────────┐              ┌────────┐     │
│   │ Disco  │── DMA ──────▶│Kernel  │── DMA ──────▶│ Disco  │     │
│   │origen  │              │Buffer  │              │destino │     │
│   └────────┘              └────────┘              └────────┘     │
│                                                                   │
│   ✓ Sin copia a espacio de usuario                                │
│   ✓ Usa DMA (Direct Memory Access) del hardware                  │
│   ✓ Ideal para copiar archivos grandes                            │
│                                                                   │
│   vs. enfoque tradicional:                                        │
│   Disco → Kernel Buffer → User Buffer → Kernel Buffer → Disco    │
│                        (4 copias!)                                │
└──────────────────────────────────────────────────────────────────┘
```

```java
import java.io.*;
import java.nio.channels.*;

public class TransferenciaDirecta {

    // Copiar archivo usando transferTo (zero-copy cuando es posible)
    public static void copiarZeroCopy(Path origen, Path destino) throws IOException {
        try (FileChannel sourceChannel = FileChannel.open(origen, StandardOpenOption.READ);
             FileChannel destChannel = FileChannel.open(destino,
                 StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {

            long tamano = sourceChannel.size();
            long posicion = 0;
            long transferido = 0;

            // transferTo puede no transferir todo de una vez, por eso iteramos
            while (posicion < tamano) {
                transferido = sourceChannel.transferTo(posicion,
                    tamano - posicion, destChannel);
                posicion += transferido;
            }
        }
    }

    // Copiar archivo usando transferFrom
    public static void copiarTransferFrom(Path origen, Path destino) throws IOException {
        try (FileChannel sourceChannel = FileChannel.open(origen, StandardOpenOption.READ);
             FileChannel destChannel = FileChannel.open(destino,
                 StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {

            long tamano = sourceChannel.size();
            long posicion = 0;
            long transferido = 0;

            while (posicion < tamano) {
                transferido = destChannel.transferFrom(sourceChannel,
                    posicion, tamano - posicion);
                posicion += transferido;
            }
        }
    }

    public static void main(String[] args) throws IOException {
        // Comparar copia tradicional vs zero-copy
        Path origen = Paths.get("/tmp/origen-grande.bin");
        Path destinoTrad = Paths.get("/tmp/destino-trad.bin");
        Path destinoZC = Paths.get("/tmp/destino-zc.bin");

        // Crear archivo de prueba de 1 GB si no existe
        if (!Files.exists(origen)) {
            System.out.println("Creando archivo de 1 GB...");
            byte[] bloque = new byte[1024 * 1024 * 8]; // 8 MB
            try (FileChannel ch = FileChannel.open(origen,
                    StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {
                java.nio.ByteBuffer buf = java.nio.ByteBuffer.wrap(bloque);
                for (int i = 0; i < 128; i++) {
                    buf.rewind();
                    ch.write(buf);
                }
            }
        }

        // Método tradicional
        long t0 = System.currentTimeMillis();
        try (InputStream is = Files.newInputStream(origen);
             OutputStream os = Files.newOutputStream(destinoTrad)) {
            byte[] buffer = new byte[8192];
            int leidos;
            while ((leidos = is.read(buffer)) != -1) {
                os.write(buffer, 0, leidos);
            }
        }
        System.out.printf("Tradicional:        %d ms%n", System.currentTimeMillis() - t0);

        // Zero-copy
        t0 = System.currentTimeMillis();
        copiarZeroCopy(origen, destinoZC);
        System.out.printf("transferTo (zero-copy): %d ms%n", System.currentTimeMillis() - t0);

        Files.deleteIfExists(destinoTrad);
        Files.deleteIfExists(destinoZC);
    }
}
```

### 6.13.3 Comparación de rendimiento: FileInputStream vs FileChannel vs mmap

```java
import java.io.*;
import java.nio.*;
import java.nio.channels.*;

public class ComparacionRendimiento {

    private static final int TAMANO_ARCHIVO_MB = 500;

    public static long medirFileInputStream(Path archivo) throws IOException {
        long inicio = System.nanoTime();
        byte[] buffer = new byte[32768]; // 32 KB
        try (FileInputStream fis = new FileInputStream(archivo.toFile())) {
            while (fis.read(buffer) != -1) { /* solo leer */ }
        }
        return System.nanoTime() - inicio;
    }

    public static long medirFileChannel(Path archivo) throws IOException {
        long inicio = System.nanoTime();
        ByteBuffer buffer = ByteBuffer.allocateDirect(32768);
        try (FileChannel ch = FileChannel.open(archivo, StandardOpenOption.READ)) {
            while (ch.read(buffer) != -1) {
                buffer.clear();
            }
        }
        return System.nanoTime() - inicio;
    }

    public static long medirMMap(Path archivo) throws IOException {
        long inicio = System.nanoTime();
        long tamano = Files.size(archivo);
        long fragmento = 128 * 1024 * 1024; // 128 MB

        try (FileChannel ch = FileChannel.open(archivo, StandardOpenOption.READ)) {
            for (long pos = 0; pos < tamano; pos += fragmento) {
                long actual = Math.min(fragmento, tamano - pos);
                MappedByteBuffer buf = ch.map(FileChannel.MapMode.READ_ONLY, pos, actual);
                while (buf.hasRemaining()) { buf.get(); }
            }
        }
        return System.nanoTime() - inicio;
    }

    public static long medirMmapSequencial(Path archivo) throws IOException {
        long inicio = System.nanoTime();
        try (FileChannel ch = FileChannel.open(archivo, StandardOpenOption.READ)) {
            MappedByteBuffer buf = ch.map(FileChannel.MapMode.READ_ONLY, 0, Files.size(archivo));
            while (buf.hasRemaining()) { buf.get(); }
        }
        return System.nanoTime() - inicio;
    }

    public static void main(String[] args) throws IOException {
        Path archivo = Paths.get("/tmp/perf-test.bin");

        if (!Files.exists(archivo)) {
            System.out.println("Creando archivo de " + TAMANO_ARCHIVO_MB + " MB...");
            byte[] bloque = new byte[1024 * 1024 * 8];
            try (FileChannel ch = FileChannel.open(archivo,
                    StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {
                ByteBuffer buf = ByteBuffer.wrap(bloque);
                for (int i = 0; i < TAMANO_ARCHIVO_MB / 8; i++) {
                    buf.rewind();
                    ch.write(buf);
                }
            }
        }

        System.out.println("\n--- Benchmark de Rendimiento (" + TAMANO_ARCHIVO_MB + " MB) ---\n");

        // Warmup
        medirFileInputStream(archivo);

        System.out.printf("%-30s %10s %15s%n", "Método", "Tiempo(ms)", "Rendimiento(MB/s)");

        for (int i = 0; i < 3; i++) {
            long ns;
            double ms, mbps;

            ns = medirFileInputStream(archivo);
            ms = ns / 1_000_000.0;
            mbps = TAMANO_ARCHIVO_MB / (ms / 1000.0);
            System.out.printf("%-30s %10.1f %15.1f%n", "FileInputStream (32KB buf)", ms, mbps);

            ns = medirFileChannel(archivo);
            ms = ns / 1_000_000.0;
            mbps = TAMANO_ARCHIVO_MB / (ms / 1000.0);
            System.out.printf("%-30s %10.1f %15.1f%n", "FileChannel (DirectBuffer)", ms, mbps);

            ns = medirMMap(archivo);
            ms = ns / 1_000_000.0;
            mbps = TAMANO_ARCHIVO_MB / (ms / 1000.0);
            System.out.printf("%-30s %10.1f %15.1f%n", "mmap (fragmentado 128MB)", ms, mbps);

            if (TAMANO_ARCHIVO_MB <= 2000) {
                ns = medirMmapSequencial(archivo);
                ms = ns / 1_000_000.0;
                mbps = TAMANO_ARCHIVO_MB / (ms / 1000.0);
                System.out.printf("%-30s %10.1f %15.1f%n", "mmap (secuencial)", ms, mbps);
            }

            System.out.println();
        }
    }
}
```

### 6.13.4 Resumen de estrategias de alto rendimiento

```
┌──────────────────────────────────────────────────────────────────┐
│   GUÍA DE RENDIMIENTO DE I/O EN JAVA                              │
│                                                                   │
│   ¿Qué necesitas hacer?                                           │
│   │                                                               │
│   ├── Copiar archivos                                             │
│   │   └── Usa FileChannel.transferTo() (zero-copy)               │
│   │                                                               │
│   ├── Leer archivo secuencialmente                                │
│   │   ├── < 100 MB: Files.readAllLines() o Files.readString()    │
│   │   ├── 100 MB - 1 GB: BufferedInputStream con buffer 32 KB    │
│   │   └── > 1 GB: FileChannel con ByteBuffer directo             │
│   │                                                               │
│   ├── Acceso aleatorio a archivos                                 │
│   │   ├── < 2 GB: MappedByteBuffer (mmap)                        │
│   │   └── > 2 GB: mmap fragmentado o RandomAccessFile            │
│   │                                                               │
│   ├── Muchas operaciones concurrentes (> 1000)                   │
│   │   └── AsynchronousFileChannel                                 │
│   │                                                               │
│   ├── Procesar archivo de texto línea por línea                  │
│   │   └── Files.lines() con Stream (memoria constante)           │
│   │                                                               │
│   ├── Escritura intensiva de logs                                │
│   │   └── BufferedWriter con flush periódico                     │
│   │                                                               │
│   └── Archivos temporales                                        │
│       └── StandardOpenOption.DELETE_ON_CLOSE                      │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 6.13.5 Tamaños de buffer recomendados

| Escenario | Tamaño buffer | Justificación |
|-----------|--------------|---------------|
| Archivos pequeños (< 1 MB) | 4096 (4 KB) | Coincide con el tamaño de página del SO |
| Archivos medianos (1 MB - 1 GB) | 8192-32768 (8-32 KB) | Balance entre llamadas al SO y memoria |
| Archivos grandes (> 1 GB) | 65536-131072 (64-128 KB) | Menos llamadas al SO, buena utilización del bus |
| Red (sockets) | 4096-8192 (4-8 KB) | Coincide con MTU de red y buffers del kernel |
| SSD/NVMe | 32768-131072 (32-128 KB) | Aprovecha el mayor ancho de banda |
| Discos rotacionales (HDD) | 65536 (64 KB) | Reduce seeks, aprovecha lectura secuencial |

---

## 6.14 Comparación: `java.io` clásico vs. `java.nio.file` (NIO.2)

| Característica | `java.io` clásico | `java.nio.file` (NIO.2) |
|---------------|-------------------|--------------------------|
| **Representación de ruta** | `java.io.File` | `java.nio.file.Path` |
| **Creación de ruta** | `new File("ruta")` | `Paths.get("ruta")` |
| **Operaciones con archivos** | Métodos en `File` + streams manuales | Métodos estáticos en `Files` |
| **Lectura de texto** | `BufferedReader` + `FileReader` | `Files.readAllLines()`, `Files.lines()`, `Files.readString()` |
| **Escritura de texto** | `BufferedWriter` + `FileWriter` | `Files.write()`, `Files.writeString()` |
| **Copiar/mover archivos** | No nativo (código manual con streams) | `Files.copy()`, `Files.move()` con opciones |
| **Recorrer directorios** | `File.listFiles()` recursivo manual | `Files.walk()`, `Files.find()`, `FileVisitor` |
| **Manejo de errores** | Retorna `false` o `null` (propenso a olvidos) | Lanza excepción con detalles |
| **Atributos de archivo** | Métodos básicos en `File` | `BasicFileAttributes`, atributos POSIX, DOS, ACL |
| **Enlaces simbólicos** | No soportado | Soporte completo con opciones `LinkOption` |
| **Charset explícito** | No en `FileReader`/`FileWriter` (usa charset por defecto) | Siempre especificado (recomendado usar siempre) |
| **Monitoreo de cambios** | No disponible | `WatchService` |
| **Streams de bytes** | `FileInputStream` / `FileOutputStream` | `Files.newInputStream()` / `Files.newOutputStream()` |
| **Rendimiento** | Adecuado | Optimizado con canales y buffers directos (`ByteBuffer`) |
| **API Stream (Java 8)** | No integrado | `Files.lines()`, `Files.walk()`, `Files.find()`, `Files.list()` |
| **Versión de Java** | Desde 1.0 | Desde Java 7 |

### ¿Cuándo usar cada uno?

- **Usa `java.io.File` y streams clásicos** cuando mantengas código legacy, cuando trabajes con APIs antiguas que aún esperan `File`, o para casos muy simples de lectura/escritura secuencial.
- **Usa NIO.2 (`Path` + `Files`)** en **todo código nuevo**. Es más expresivo, más seguro (excepciones descriptivas), soporta operaciones atómicas, tiene mejor integración con Streams de Java 8 y ofrece funcionalidades que `java.io` simplemente no tiene (monitoreo de archivos, enlaces simbólicos, recorrido eficiente de directorios).
- **Usa NIO (canales y buffers, `java.nio.channels`)** cuando necesites I/O de alto rendimiento, mapeo de archivos en memoria (`MappedByteBuffer`), operaciones no bloqueantes o transferencia directa entre canales. Este tema se aborda en profundidad en el Capítulo 7.

---


## Resumen del capítulo

- Los **streams de bytes** (`InputStream`/`OutputStream`) trabajan con datos binarios crudos; los **streams de caracteres** (`Reader`/`Writer`) interpretan texto con un charset.
- Usa siempre **try-with-resources** para cerrar automáticamente los streams y evitar fugas de recursos.
- **Buffering** (`BufferedInputStream`, `BufferedReader`) es esencial para rendimiento: evita lecturas/escrituras individuales al sistema operativo.
- `DataInputStream`/`DataOutputStream` permiten serializar tipos primitivos en formato binario portable.
- `PrintWriter` simplifica la escritura de texto formateado con `printf`.
- La clase `File` permite manipular rutas y metadatos pero no el contenido de archivos.
- La **serialización** convierte objetos en bytes; usa `Serializable`, cuidado con `transient` y `serialVersionUID`, y nunca deserialices datos no confiables sin filtros.
- **NIO.2** (`Path` + `Files`) es la API moderna para archivos: más segura, expresiva y potente que `java.io.File`.
- `Files.walk()`, `Files.find()` y `FileVisitor` permiten recorrer directorios de forma eficiente.
- `WatchService` proporciona monitoreo de cambios en tiempo real en el sistema de archivos.

---

← [Capítulo anterior](capitulo-05-excepciones.md) | [Inicio](../README.md) | [Capítulo siguiente →](capitulo-07-concurrencia.md)
