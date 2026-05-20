# Capítulo 10: Buenas Prácticas, Arquitectura y Profesionalismo en Java

> *"Cualquier tonto puede escribir código que una máquina entienda. Los buenos programadores escriben código que los humanos puedan entender."* — Martin Fowler

---

Este es el capítulo final. Si has llegado hasta aquí, ya dominas la sintaxis de Java, la orientación a objetos, las colecciones, las excepciones, la concurrencia, los streams, el acceso a bases de datos... Pero el viaje del programador no termina cuando el código compila. Apenas comienza.

Lo que distingue a un programador profesional no es la velocidad al teclear ni la memoria para recordar APIs. Es la capacidad de diseñar sistemas que sobrevivan al paso del tiempo, que puedan ser modificados sin miedo, que fallen de forma predecible y que puedan ser diagnosticados en producción a las 3 de la mañana.

Este capítulo reúne las herramientas intelectuales y prácticas que necesitas para dar ese salto. Desde los principios SOLID hasta la arquitectura hexagonal, desde TDD hasta la observabilidad en producción, desde los patrones de diseño clásicos hasta el ecosistema moderno de microservicios. Y al final, una reflexión sobre lo que significa crecer como programador.

---

## 10.1 Principios SOLID

Los principios SOLID, acuñados por Robert C. Martin (Uncle Bob), son cinco directrices de diseño orientado a objetos que conducen a sistemas más mantenibles, flexibles y testeables. No son reglas dogmáticas: son criterios para pensar mejor el diseño.

### 10.1.1 S — Principio de Responsabilidad Única (Single Responsibility)

> **Una clase debe tener una, y solo una, razón para cambiar.**

Cuando una clase asume múltiples responsabilidades, cualquier cambio en una de ellas puede afectar inadvertidamente a las otras. El acoplamiento innecesario es la raíz de muchos dolores de cabeza en el mantenimiento.

#### Problema: una clase monolítica

```java
public class ReporteEmpleado {
    private String nombre;
    private double salario;

    public ReporteEmpleado(String nombre, double salario) {
        this.nombre = nombre;
        this.salario = salario;
    }

    // Responsabilidad 1: cálculo de nómina
    public double calcularImpuestos() { return salario * 0.19; }
    public double calcularSalarioNeto() { return salario - calcularImpuestos(); }

    // Responsabilidad 2: persistencia
    public void guardarEnBaseDeDatos() {
        System.out.println("INSERT INTO empleados VALUES (...)");
    }

    // Responsabilidad 3: generación de reportes
    public String generarReportePDF() { return "Reporte PDF de " + nombre; }
    public String generarReporteCSV() { return nombre + "," + salario; }
}
```

Esta clase tiene al menos tres razones para cambiar: si cambia la lógica fiscal, si cambia la base de datos, o si cambia el formato de los reportes.

#### Solución: separar responsabilidades

```java
// Responsabilidad única: representar al empleado
public class Empleado {
    private String nombre;
    private double salario;

    public Empleado(String nombre, double salario) {
        this.nombre = nombre; this.salario = salario;
    }
    public String getNombre() { return nombre; }
    public double getSalario() { return salario; }
}

// Responsabilidad única: calcular impuestos y nómina
public class CalculadoraNomina {
    private static final double TASA_IMPUESTO = 0.19;
    public double calcularImpuestos(Empleado empleado) {
        return empleado.getSalario() * TASA_IMPUESTO;
    }
    public double calcularSalarioNeto(Empleado empleado) {
        return empleado.getSalario() - calcularImpuestos(empleado);
    }
}

// Responsabilidad única: persistencia
public class RepositorioEmpleado {
    public void guardar(Empleado empleado) {
        System.out.printf("INSERT INTO empleados (nombre, salario) VALUES ('%s', %.2f)%n",
                empleado.getNombre(), empleado.getSalario());
    }
}

// Responsabilidad única: generación de reportes
public class GeneradorReporteEmpleado {
    public String generarPDF(Empleado empleado) {
        return String.format("Reporte PDF de %s - Salario: %.2f",
                empleado.getNombre(), empleado.getSalario());
    }
    public String generarCSV(Empleado empleado) {
        return String.format("%s,%.2f", empleado.getNombre(), empleado.getSalario());
    }
}
```

**Beneficio:** cada clase evoluciona de forma independiente. Si Hacienda cambia la tasa impositiva, solo modificamos `CalculadoraNomina`. Si migramos de MySQL a PostgreSQL, solo tocamos `RepositorioEmpleado`.

#### Ejemplo de producción real: Servicio de Notificaciones (Email, SMS, Push)

Imagina un sistema que envía notificaciones por email, SMS y push. Una implementación naive que viola SRP:

```java
// VIOLACIÓN: clase monolítica que hace demasiado
public class ServicioNotificaciones {
    private final JavaMailSender mailSender;
    private final TwilioClient smsClient;
    private final FirebaseClient pushClient;

    public void enviarNotificacion(Usuario usuario, String mensaje, String canal) {
        switch (canal) {
            case "EMAIL" -> {
                MimeMessage email = mailSender.createMimeMessage();
                email.setRecipient(usuario.getEmail());
                email.setSubject("Notificacion");
                email.setText(mensaje);
                if (!usuario.getEmail().contains("@"))
                    throw new IllegalArgumentException("Email invalido");
                mailSender.send(email);
                registrarAuditoria(usuario.getId(), "EMAIL", mensaje);
            }
            case "SMS" -> {
                String telefono = usuario.getTelefono();
                if (!telefono.startsWith("+")) telefono = "+34" + telefono;
                smsClient.sendMessage(telefono, mensaje);
                registrarAuditoria(usuario.getId(), "SMS", mensaje);
            }
            case "PUSH" -> {
                if (usuario.getDeviceToken() == null)
                    throw new IllegalArgumentException("Sin token de dispositivo");
                pushClient.sendPush(usuario.getDeviceToken(), mensaje);
                registrarAuditoria(usuario.getId(), "PUSH", mensaje);
            }
        }
    }

    private void registrarAuditoria(long userId, String canal, String mensaje) {
        // INSERT en tabla de auditoria...
    }
}
```

Cada nuevo canal (WhatsApp, Telegram) requiere modificar esta clase gigante. Refactoricemos con SRP:

```java
// Abstraccion: cada canal es una responsabilidad separada
public interface CanalNotificacion {
    void enviar(Usuario usuario, String mensaje);
    boolean soporta(Usuario usuario);
}

public class NotificadorEmail implements CanalNotificacion {
    private final JavaMailSender mailSender;
    private final ValidadorEmail validador;

    @Override
    public void enviar(Usuario usuario, String mensaje) {
        validador.validar(usuario.getEmail());
        MimeMessage email = mailSender.createMimeMessage();
        email.setRecipient(usuario.getEmail());
        email.setText(mensaje);
        mailSender.send(email);
    }

    @Override
    public boolean soporta(Usuario usuario) { return usuario.getEmail() != null; }
}

public class NotificadorSMS implements CanalNotificacion {
    private final TwilioClient smsClient;

    @Override
    public void enviar(Usuario usuario, String mensaje) {
        String telefono = NormalizadorTelefono.normalizar(usuario.getTelefono());
        smsClient.sendMessage(telefono, mensaje);
    }

    @Override
    public boolean soporta(Usuario usuario) { return usuario.getTelefono() != null; }
}

public class NotificadorPush implements CanalNotificacion {
    private final FirebaseClient pushClient;

    @Override
    public void enviar(Usuario usuario, String mensaje) {
        pushClient.sendPush(usuario.getDeviceToken(), mensaje);
    }

    @Override
    public boolean soporta(Usuario usuario) { return usuario.getDeviceToken() != null; }
}

// Orquestador: solo coordina, no conoce los detalles de cada canal
public class ServicioNotificaciones {
    private final List<CanalNotificacion> canales;
    private final AuditoriaService auditoria;

    public ServicioNotificaciones(List<CanalNotificacion> canales, AuditoriaService auditoria) {
        this.canales = canales;
        this.auditoria = auditoria;
    }

    public void notificar(Usuario usuario, String mensaje) {
        for (CanalNotificacion canal : canales) {
            if (canal.soporta(usuario)) {
                canal.enviar(usuario, mensaje);
                auditoria.registrar(usuario.getId(),
                        canal.getClass().getSimpleName(), mensaje);
            }
        }
    }
}
```

Ahora anadir WhatsApp es crear una clase nueva, sin tocar nada existente. Cada clase tiene una sola razon para cambiar.

---

### 10.1.2 O — Principio de Abierto/Cerrado (Open/Closed)

> **Las entidades de software deben estar abiertas para extension, pero cerradas para modificacion.**

Debemos poder anadir nuevo comportamiento sin modificar el codigo existente. Esto se logra mediante abstracciones (interfaces, clases abstractas) y polimorfismo.

#### Problema: codigo que crece con `if` encadenados

```java
public class CalculadoraArea {
    public double calcularArea(Object figura) {
        if (figura instanceof Circulo c)
            return Math.PI * c.getRadio() * c.getRadio();
        else if (figura instanceof Rectangulo r)
            return r.getAncho() * r.getAlto();
        else if (figura instanceof Triangulo t)
            return (t.getBase() * t.getAltura()) / 2.0;
        else throw new IllegalArgumentException("Figura no soportada");
    }
}
```

#### Solucion: abstraer con una interfaz

```java
public interface Figura { double calcularArea(); }

public class Circulo implements Figura {
    private final double radio;
    public Circulo(double radio) { this.radio = radio; }
    @Override public double calcularArea() { return Math.PI * radio * radio; }
}

public class Rectangulo implements Figura {
    private final double ancho, alto;
    public Rectangulo(double ancho, double alto) { this.ancho = ancho; this.alto = alto; }
    @Override public double calcularArea() { return ancho * alto; }
}

public class Triangulo implements Figura {
    private final double base, altura;
    public Triangulo(double base, double altura) { this.base = base; this.altura = altura; }
    @Override public double calcularArea() { return (base * altura) / 2.0; }
}
```

Ahora anadir una nueva figura no requiere tocar `CalculadoraArea`:

```java
public class Pentagono implements Figura {
    private final double lado;
    private static final double FACTOR = 1.7204774;
    public Pentagono(double lado) { this.lado = lado; }
    @Override public double calcularArea() { return FACTOR * lado * lado; }
}
```

#### Enfoque con clase abstracta

```java
public abstract class PoligonoRegular implements Figura {
    protected final double lado;
    protected final int numLados;

    protected PoligonoRegular(double lado, int numLados) {
        this.lado = lado; this.numLados = numLados;
    }
    protected double apotema() {
        return lado / (2.0 * Math.tan(Math.PI / numLados));
    }
    @Override
    public double calcularArea() { return (numLados * lado * apotema()) / 2.0; }
}

public class Hexagono extends PoligonoRegular {
    public Hexagono(double lado) { super(lado, 6); }
}
```

#### Ejemplo de produccion real: Pasarela de Pago (Stripe, PayPal, MercadoPago) con OCP + Strategy

Un sistema de e-commerce necesita aceptar multiples proveedores de pago. Cada uno tiene su propia API, autenticacion y formato de respuesta. Anadir un nuevo proveedor sin tocar el codigo existente es el sueno de OCP.

```java
// Contrato comun para todas las pasarelas de pago
public interface PasarelaPago {
    ResultadoPago procesarPago(Pago pago);
    ResultadoReembolso reembolsar(String idTransaccion);
    EstadoTransaccion consultarEstado(String idTransaccion);
}

// Entidades de dominio compartidas
public record Pago(String idPedido, BigDecimal monto, Moneda moneda,
                   DatosTarjeta tarjeta, String idempotencia) {}

public record ResultadoPago(String idTransaccion, EstadoPago estado,
                            String mensaje, Map<String, String> metadatos) {}

public enum EstadoPago { APROBADO, RECHAZADO, PENDIENTE, ERROR }
```

```java
// Implementacion para Stripe
public class StripePasarela implements PasarelaPago {
    private final StripeClient stripeClient;
    private final String apiKey;

    public StripePasarela(StripeClient stripeClient, String apiKey) {
        this.stripeClient = stripeClient;
        this.apiKey = apiKey;
    }

    @Override
    public ResultadoPago procesarPago(Pago pago) {
        StripeCharge charge = stripeClient.charges().create(
            Map.of(
                "amount", pago.monto().multiply(BigDecimal.valueOf(100)).longValue(),
                "currency", pago.moneda().getCodigo(),
                "source", pago.tarjeta().token(),
                "idempotency_key", pago.idempotencia()
            ), apiKey
        );
        return mapearRespuesta(charge);
    }

    @Override
    public ResultadoReembolso reembolsar(String idTransaccion) {
        StripeRefund refund = stripeClient.refunds().create(idTransaccion, apiKey);
        return new ResultadoReembolso(refund.id(),
                refund.status().equals("succeeded"));
    }

    @Override
    public EstadoTransaccion consultarEstado(String idTransaccion) {
        StripeCharge charge = stripeClient.charges().retrieve(idTransaccion, apiKey);
        return mapearEstado(charge.status());
    }

    private ResultadoPago mapearRespuesta(StripeCharge charge) {
        return new ResultadoPago(charge.id(),
            switch (charge.status()) {
                case "succeeded" -> EstadoPago.APROBADO;
                case "failed" -> EstadoPago.RECHAZADO;
                default -> EstadoPago.PENDIENTE;
            }, charge.description(), Map.of());
    }

    private EstadoTransaccion mapearEstado(String status) {
        return "succeeded".equals(status) ? EstadoTransaccion.COMPLETADA
                : EstadoTransaccion.FALLIDA;
    }
}
```

```java
// Implementacion para PayPal
public class PayPalPasarela implements PasarelaPago {
    private final PayPalClient paypalClient;
    private final String clientId;
    private final String secret;

    public PayPalPasarela(PayPalClient paypalClient, String clientId, String secret) {
        this.paypalClient = paypalClient;
        this.clientId = clientId;
        this.secret = secret;
    }

    @Override
    public ResultadoPago procesarPago(Pago pago) {
        String accessToken = paypalClient.authenticate(clientId, secret);
        PayPalOrder order = paypalClient.orders().create(
            PayPalOrderRequest.builder()
                .amount(pago.monto())
                .currency(pago.moneda().getCodigo())
                .intent("CAPTURE")
                .build(),
            accessToken
        );
        return mapearRespuesta(order);
    }

    @Override
    public ResultadoReembolso reembolsar(String idTransaccion) { /* ... */ return null; }

    @Override
    public EstadoTransaccion consultarEstado(String idTransaccion) { /* ... */ return null; }

    private ResultadoPago mapearRespuesta(PayPalOrder order) { /* ... */ return null; }
}
```

```java
// Implementacion para MercadoPago
public class MercadoPagoPasarela implements PasarelaPago {
    private final MercadoPagoClient mpClient;
    private final String accessToken;

    @Override
    public ResultadoPago procesarPago(Pago pago) {
        MPPayment payment = mpClient.payments().create(
            MPPaymentRequest.builder()
                .transactionAmount(pago.monto())
                .token(pago.tarjeta().token())
                .description("Pedido #" + pago.idPedido())
                .installments(1)
                .paymentMethodId(pago.tarjeta().tipo())
                .build(),
            accessToken
        );
        return mapearRespuesta(payment);
    }

    @Override public ResultadoReembolso reembolsar(String id) { return null; }
    @Override public EstadoTransaccion consultarEstado(String id) { return null; }
    private ResultadoPago mapearRespuesta(MPPayment p) { return null; }
}
```

```java
// El orquestador no conoce ninguna implementacion concreta.
// Anadir un nuevo proveedor es crear una clase, no tocar el orquestador.
public class ProcesadorPagos {
    private final Map<String, PasarelaPago> pasarelas;

    public ProcesadorPagos(Map<String, PasarelaPago> pasarelas) {
        this.pasarelas = pasarelas;
    }

    public ResultadoPago procesar(Pago pago, String proveedor) {
        PasarelaPago pasarela = pasarelas.get(proveedor.toUpperCase());
        if (pasarela == null)
            throw new IllegalArgumentException("Proveedor no soportado: " + proveedor);
        return pasarela.procesarPago(pago);
    }
}

// Configuracion: anadir un nuevo proveedor es una linea
var procesador = new ProcesadorPagos(Map.of(
    "STRIPE", new StripePasarela(stripeClient, apiKey),
    "PAYPAL", new PayPalPasarela(paypalClient, clientId, secret),
    "MERCADOPAGO", new MercadoPagoPasarela(mpClient, accessToken)
));
```

Este diseno esta **abierto para extension** (nuevas pasarelas) pero **cerrado para modificacion** (el `ProcesadorPagos` no cambia).

---

### 10.1.3 L — Principio de Sustitucion de Liskov (Liskov Substitution)

> **Los subtipos deben ser sustituibles por sus tipos base sin alterar el correcto funcionamiento del programa.**

Formulado por Barbara Liskov en 1987, este principio establece que si una clase `S` es un subtipo de `T`, los objetos de tipo `T` pueden reemplazarse por objetos de tipo `S` sin que el programa se comporte incorrectamente.

#### La violacion clasica: Cuadrado extiende Rectangulo

Parece natural: en geometria, un cuadrado *es* un rectangulo. Pero en programacion, la herencia debe respetar el comportamiento, no solo la taxonomia del mundo real.

```java
// VIOLACION DE LSP
public class Rectangulo {
    protected double ancho;
    protected double alto;

    public void setAncho(double ancho) { this.ancho = ancho; }
    public void setAlto(double alto) { this.alto = alto; }
    public double getAncho() { return ancho; }
    public double getAlto() { return alto; }
    public double calcularArea() { return ancho * alto; }
}

public class Cuadrado extends Rectangulo {
    @Override
    public void setAncho(double ancho) {
        super.setAncho(ancho);
        super.setAlto(ancho); // Fuerza la invariante del cuadrado
    }

    @Override
    public void setAlto(double alto) {
        super.setAlto(alto);
        super.setAncho(alto); // Fuerza la invariante del cuadrado
    }
}
```

Ahora, un cliente que usa `Rectangulo` espera que modificar el ancho no afecte al alto:

```java
public class Cliente {
    public void redimensionar(Rectangulo r) {
        r.setAncho(5); r.setAlto(10);
        assert r.calcularArea() == 50 : "Fallo! Area = " + r.calcularArea();
    }

    public static void main(String[] args) {
        Cliente cliente = new Cliente();
        Rectangulo rect = new Rectangulo();
        cliente.redimensionar(rect); // OK: area = 50

        Rectangulo cuad = new Cuadrado(); // Sustitucion
        cliente.redimensionar(cuad);      // FALLO: area = 100
    }
}
```

El problema no es la representacion, sino el **contrato**: `Rectangulo` promete que `setAncho` y `setAlto` son independientes; `Cuadrado` rompe esa promesa.

#### Solucion con interfaz comun y clases inmutables

```java
public interface Figura { double calcularArea(); }

public class Rectangulo implements Figura {
    private final double ancho;
    private final double alto;

    public Rectangulo(double ancho, double alto) {
        this.ancho = ancho; this.alto = alto;
    }
    public double getAncho() { return ancho; }
    public double getAlto() { return alto; }
    @Override public double calcularArea() { return ancho * alto; }
}

public class Cuadrado implements Figura {
    private final double lado;
    public Cuadrado(double lado) { this.lado = lado; }
    public double getLado() { return lado; }
    @Override public double calcularArea() { return lado * lado; }
}
```

Al hacer las clases inmutables (sin setters), el problema desaparece. La inmutabilidad es una gran aliada de LSP.

#### Ejemplo de produccion real: CuentaBancaria vs CuentaAhorro

Consideremos un sistema bancario donde `CuentaAhorro` extiende `CuentaBancaria`:

```java
// VIOLACION: CuentaAhorro extiende CuentaBancaria pero rompe el contrato
public class CuentaBancaria {
    protected BigDecimal saldo;
    protected final String numeroCuenta;

    public CuentaBancaria(String numeroCuenta, BigDecimal saldoInicial) {
        this.numeroCuenta = numeroCuenta;
        this.saldo = saldoInicial;
    }

    public void depositar(BigDecimal monto) {
        if (monto.compareTo(BigDecimal.ZERO) <= 0)
            throw new IllegalArgumentException("Monto debe ser positivo");
        saldo = saldo.add(monto);
    }

    // Contrato: se puede retirar cualquier monto si hay saldo
    public void retirar(BigDecimal monto) {
        if (monto.compareTo(BigDecimal.ZERO) <= 0)
            throw new IllegalArgumentException("Monto debe ser positivo");
        if (monto.compareTo(saldo) > 0)
            throw new SaldoInsuficienteException("Saldo insuficiente");
        saldo = saldo.subtract(monto);
    }

    public BigDecimal getSaldo() { return saldo; }
}
```

```java
// VIOLACION: anade restricciones que rompen el contrato de CuentaBancaria
public class CuentaAhorro extends CuentaBancaria {
    private static final BigDecimal SALDO_MINIMO = new BigDecimal("100");
    private static final int MAX_RETIROS_MENSUALES = 6;
    private int retirosEsteMes = 0;

    public CuentaAhorro(String numeroCuenta, BigDecimal saldoInicial) {
        super(numeroCuenta, saldoInicial);
        if (saldoInicial.compareTo(SALDO_MINIMO) < 0)
            throw new IllegalArgumentException("Saldo inicial inferior al minimo");
    }

    @Override
    public void retirar(BigDecimal monto) {
        // Refuerza la precondicion: restricciones que la clase base no tiene
        if (retirosEsteMes >= MAX_RETIROS_MENSUALES)
            throw new LimiteRetirosExcedidoException("Maximo de retiros alcanzado");
        if (saldo.subtract(monto).compareTo(SALDO_MINIMO) < 0)
            throw new SaldoInsuficienteException("Retiro dejaria saldo bajo el minimo");
        retirosEsteMes++;
        super.retirar(monto);
    }
}
```

El problema: un metodo como `procesarTransferenciaMasiva(List<CuentaBancaria> cuentas)` espera que todas las cuentas se comporten igual. Pero `CuentaAhorro` lanza excepciones inesperadas.

#### Solucion: usar composicion, no herencia

```java
// Interfaz comun que define el contrato real
public interface Cuenta {
    void depositar(BigDecimal monto);
    void retirar(BigDecimal monto);
    BigDecimal getSaldo();
    String getNumeroCuenta();
}

// Cuenta corriente: implementa solo lo que necesita
public class CuentaCorriente implements Cuenta {
    private BigDecimal saldo;
    private final String numeroCuenta;

    public CuentaCorriente(String numeroCuenta, BigDecimal saldoInicial) {
        this.numeroCuenta = numeroCuenta;
        this.saldo = saldoInicial;
    }

    @Override
    public void depositar(BigDecimal monto) {
        validarMontoPositivo(monto);
        saldo = saldo.add(monto);
    }

    @Override
    public void retirar(BigDecimal monto) {
        validarMontoPositivo(monto);
        if (monto.compareTo(saldo) > 0)
            throw new SaldoInsuficienteException("Saldo insuficiente");
        saldo = saldo.subtract(monto);
    }

    @Override public BigDecimal getSaldo() { return saldo; }
    @Override public String getNumeroCuenta() { return numeroCuenta; }

    private void validarMontoPositivo(BigDecimal monto) {
        if (monto.compareTo(BigDecimal.ZERO) <= 0)
            throw new IllegalArgumentException("Monto debe ser positivo");
    }
}

// Cuenta de ahorro: usa composicion, anade reglas propias
public class CuentaAhorro implements Cuenta {
    private final CuentaCorriente cuentaBase;       // Composicion
    private final PoliticaAhorro politica;

    public CuentaAhorro(String numeroCuenta, BigDecimal saldoInicial,
                         PoliticaAhorro politica) {
        this.politica = politica;
        politica.validarSaldoMinimo(saldoInicial);
        this.cuentaBase = new CuentaCorriente(numeroCuenta, saldoInicial);
    }

    @Override
    public void depositar(BigDecimal monto) { cuentaBase.depositar(monto); }

    @Override
    public void retirar(BigDecimal monto) {
        politica.verificarPuedeRetirar();
        politica.validarSaldoPostRetiro(cuentaBase.getSaldo(), monto);
        cuentaBase.retirar(monto);
        politica.registrarRetiro();
    }

    @Override public BigDecimal getSaldo() { return cuentaBase.getSaldo(); }
    @Override public String getNumeroCuenta() { return cuentaBase.getNumeroCuenta(); }
}

// La politica de ahorro es una clase separada, reutilizable y testeable
public class PoliticaAhorro {
    private final BigDecimal saldoMinimo;
    private final int maxRetirosMensuales;
    private int retirosEsteMes = 0;

    public PoliticaAhorro(BigDecimal saldoMinimo, int maxRetirosMensuales) {
        this.saldoMinimo = saldoMinimo;
        this.maxRetirosMensuales = maxRetirosMensuales;
    }

    public void validarSaldoMinimo(BigDecimal saldo) {
        if (saldo.compareTo(saldoMinimo) < 0)
            throw new IllegalArgumentException("Saldo inferior al minimo");
    }

    public void verificarPuedeRetirar() {
        if (retirosEsteMes >= maxRetirosMensuales)
            throw new LimiteRetirosExcedidoException("Maximo de retiros alcanzado");
    }

    public void validarSaldoPostRetiro(BigDecimal saldo, BigDecimal monto) {
        if (saldo.subtract(monto).compareTo(saldoMinimo) < 0)
            throw new SaldoInsuficienteException("Retiro dejaria saldo bajo minimo");
    }

    public void registrarRetiro() { retirosEsteMes++; }
}
```

Ahora `CuentaAhorro` y `CuentaCorriente` implementan la misma interfaz `Cuenta` sin sorpresas. Quien programa contra `Cuenta` no recibe comportamientos inesperados.

**Pautas para cumplir LSP:**
- Un subtipo no debe reforzar las precondiciones de un metodo.
- Un subtipo no debe debilitar las postcondiciones.
- Las excepciones lanzadas por el subtipo deben ser subtipos de las excepciones de la clase base.
- Prefiere composicion sobre herencia cuando la jerarquia no es solida.

---

### 10.1.4 I — Principio de Segregacion de Interfaces (Interface Segregation)

> **Ningun cliente debe verse forzado a depender de metodos que no utiliza.**

Las interfaces "gordas" (con muchos metodos) obligan a las implementaciones a definir metodos vacios o que lanzan `UnsupportedOperationException`. Es preferible tener varias interfaces pequenas y cohesivas.

#### Problema: interfaz demasiado generica

```java
public interface Trabajador {
    void trabajar(); void comer(); void dormir();
    void programar(); void diseniar(); void testear();
}
```

Un `Desarrollador` que implemente `Trabajador` se ve obligado a definir `comer()` y `dormir()`. Un `Robot` tendra que implementar `comer()` aunque no tenga sentido.

#### Solucion: interfaces segregadas

```java
public interface Trabajable { void trabajar(); }
public interface Programable { void programar(); }
public interface Diseniable { void diseniar(); }
public interface Testeable { void testear(); }

public class Desarrollador implements Programable, Testeable {
    @Override
    public void programar() { System.out.println("Escribiendo codigo..."); }
    @Override
    public void testear() { System.out.println("Ejecutando tests..."); }
}

public class DesarrolladorFullStack implements Programable, Diseniable, Testeable {
    @Override public void programar() { System.out.println("Backend + frontend..."); }
    @Override public void diseniar() { System.out.println("Disenando UI..."); }
    @Override public void testear() { System.out.println("Tests de integracion..."); }
}
```

#### Ejemplo de produccion real: Worker con interfaz gorda de 20 metodos

Imagina un sistema de procesamiento de trabajos batch con esta interfaz monstruosa:

```java
// INTERFAZ GORDA: 20+ metodos sin cohesion
public interface Worker {
    void start(); void stop(); void pause(); void resume();
    JobStatus getStatus(); void configure(Properties props);
    void onJobCompleted(Job job); void onJobFailed(Job job, Exception error);
    void onJobStarted(Job job); void retryFailedJobs();
    List<Job> getPendingJobs(); List<Job> getCompletedJobs();
    Map<String, Object> getMetrics(); void exportMetrics(String format);
    void sendHeartbeat(); void shutdown(); void restart();
    void reloadConfiguration(); void validateConfiguration(Properties props);
    boolean isHealthy(); String healthCheck();
}
```

Refactoricemos con ISP en interfaces pequenas y cohesivas:

```java
// Interfaces pequenas y cohesionadas
public interface Lifecycle { void start(); void stop(); boolean isRunning(); }

public interface JobProcessor {
    JobStatus getStatus(); void processJob(Job job);
    void onJobCompleted(Job job); void onJobFailed(Job job, Exception error);
}

public interface JobSchedule {
    List<Job> getPendingJobs(); List<Job> getCompletedJobs();
    void retryFailedJobs(); void cancelJob(String jobId);
}

public interface HealthCheckable {
    boolean isHealthy(); Map<String, Object> healthDetails();
}

public interface Metricable {
    Map<String, Object> getMetrics(); void incrementCounter(String metric, long value);
}

public interface Configurable {
    void configure(Properties props);
    void validateConfiguration(Properties props);
    void reloadConfiguration();
}

// Cada clase implementa solo lo que necesita
public class SimpleJobWorker implements Lifecycle, JobProcessor {
    @Override public void start() { /* ... */ }
    @Override public void stop() { /* ... */ }
    @Override public boolean isRunning() { /* ... */ }
    @Override public JobStatus getStatus() { /* ... */ }
    @Override public void processJob(Job job) { /* ... */ }
    @Override public void onJobCompleted(Job job) { /* ... */ }
    @Override public void onJobFailed(Job job, Exception error) { /* ... */ }
}

// Un worker mas completo puede implementar mas interfaces
public class FullFeaturedWorker implements Lifecycle, JobProcessor,
        JobSchedule, HealthCheckable, Metricable, Configurable {
    // Todas las implementaciones relevantes
}

// Cliente que solo necesita health check: depende de la interfaz minima
public class MonitoringService {
    private final List<HealthCheckable> servicesToMonitor;

    public MonitoringService(List<HealthCheckable> services) {
        this.servicesToMonitor = services;
    }

    public void checkAll() {
        for (HealthCheckable svc : servicesToMonitor) {
            if (!svc.isHealthy()) {
                alert(svc.healthDetails());
            }
        }
    }

    private void alert(Map<String, Object> details) { /* ... */ }
}
```

**Beneficio:** cada clase implementa solo lo que necesita. El codigo es mas claro y el acoplamiento se reduce. En Java, las interfaces con un solo metodo se conocen como **interfaces funcionales**:

```java
@FunctionalInterface
public interface Filtrable<T> {
    boolean filtrar(T elemento);
}

List<String> nombres = List.of("Ana", "Carlos", "Beatriz");
Filtrable<String> empiezaConA = s -> s.startsWith("A");
nombres.stream().filter(empiezaConA::filtrar).forEach(System.out::println);
```

---

### 10.1.5 D — Principio de Inversion de Dependencias (Dependency Inversion)

> **Los modulos de alto nivel no deben depender de modulos de bajo nivel. Ambos deben depender de abstracciones.**

En Java, esto se materializa mediante **inyeccion de dependencias** y el uso de interfaces para desacoplar.

#### Problema: dependencia directa de una implementacion concreta

```java
// Modulo de bajo nivel: implementacion concreta
public class ServicioNotificacionEmail {
    public void enviar(String destinatario, String mensaje) {
        System.out.printf("Enviando email a %s: %s%n", destinatario, mensaje);
    }
}

// Modulo de alto nivel: depende directamente del modulo de bajo nivel
public class ProcesadorPedidos {
    private ServicioNotificacionEmail notificador = new ServicioNotificacionEmail();

    public void procesarPedido(Pedido pedido) {
        notificador.enviar(pedido.getClienteEmail(),
                "Su pedido #" + pedido.getId() + " ha sido procesado.");
    }
}
```

#### Solucion: depender de una abstraccion (Inyeccion Manual)

```java
public interface ServicioNotificacion {
    void enviar(String destinatario, String mensaje);
}

public class ServicioNotificacionEmail implements ServicioNotificacion {
    @Override
    public void enviar(String destinatario, String mensaje) {
        System.out.printf("[EMAIL] Para: %s | Mensaje: %s%n", destinatario, mensaje);
    }
}

public class ServicioNotificacionSMS implements ServicioNotificacion {
    @Override
    public void enviar(String destinatario, String mensaje) {
        System.out.printf("[SMS] Para: %s | Mensaje: %s%n", destinatario, mensaje);
    }
}

public class ServicioNotificacionWhatsApp implements ServicioNotificacion {
    @Override
    public void enviar(String destinatario, String mensaje) {
        System.out.printf("[WhatsApp] Para: %s | Mensaje: %s%n", destinatario, mensaje);
    }
}

// Modulo de alto nivel: ahora depende de la abstraccion
public class ProcesadorPedidos {
    private final ServicioNotificacion notificador;

    public ProcesadorPedidos(ServicioNotificacion notificador) {
        this.notificador = notificador;
    }

    public void procesarPedido(Pedido pedido) {
        notificador.enviar(pedido.getClienteEmail(),
                "Su pedido #" + pedido.getId() + " ha sido procesado.");
    }
}

// Uso: las dependencias se inyectan desde fuera
public class Main {
    public static void main(String[] args) {
        ServicioNotificacion emailService = new ServicioNotificacionEmail();
        ProcesadorPedidos procesador = new ProcesadorPedidos(emailService);
        procesador.procesarPedido(new Pedido(1, "cliente@email.com"));

        // Cambiar a SMS sin modificar ProcesadorPedidos
        ServicioNotificacion smsService = new ServicioNotificacionSMS();
        ProcesadorPedidos procesador2 = new ProcesadorPedidos(smsService);
        procesador2.procesarPedido(new Pedido(2, "+56912345678"));
    }
}
```

#### Inyeccion de Dependencias con Spring Framework

En aplicaciones reales, la inyeccion manual no escala. Ahi entran los contenedores de DI como Spring:

```java
// Opcion 1: Inyeccion por constructor (RECOMENDADA)
@Service
public class ProcesadorPedidos {

    private final ServicioNotificacion notificador;
    private final RepositorioPedidos repositorio;
    private final CalculadoraImpuestos calculadora;

    // @Autowired es opcional desde Spring 4.3 si hay un solo constructor
    public ProcesadorPedidos(ServicioNotificacion notificador,
                              RepositorioPedidos repositorio,
                              CalculadoraImpuestos calculadora) {
        this.notificador = notificador;
        this.repositorio = repositorio;
        this.calculadora = calculadora;
    }

    public void procesarPedido(Pedido pedido) {
        BigDecimal impuestos = calculadora.calcular(pedido);
        pedido.setImpuestos(impuestos);
        repositorio.guardar(pedido);
        notificador.enviar(pedido.getClienteEmail(),
                "Pedido #" + pedido.getId() + " procesado. Total: "
                + pedido.getTotal());
    }
}

// Configuracion: Spring decide que implementacion concreta inyectar
@Configuration
public class AppConfig {

    @Bean
    @Profile("!prod")  // En desarrollo usamos email simulado
    public ServicioNotificacion servicioNotificacionDev() {
        return new ServicioNotificacionEmail();
    }

    @Bean
    @Profile("prod")   // En produccion usamos SMS real
    public ServicioNotificacion servicioNotificacionProd() {
        return new ServicioNotificacionSMS(credencialesTwilio());
    }

    @Bean
    @Primary  // Si hay ambiguedad, esta implementacion tiene prioridad
    public RepositorioPedidos repositorioPedidos(DataSource ds) {
        return new JdbcRepositorioPedidos(ds);
    }
}
```

```java
// Opcion 2: Inyeccion por campo (NO recomendada: dificulta testing)
@Service
public class ProcesadorPedidosMalo {
    @Autowired
    private ServicioNotificacion notificador; // Dificil de testear sin Spring

    @Autowired
    private RepositorioPedidos repositorio;   // Oculto, no obvio que depende
}

// Opcion 3: Inyeccion por setter (util para dependencias opcionales)
@Service
public class ProcesadorPedidosConOpcionales {
    private final RepositorioPedidos repositorio; // Obligatorio: constructor
    private ServicioAuditoria auditoria;           // Opcional: setter

    public ProcesadorPedidosConOpcionales(RepositorioPedidos repositorio) {
        this.repositorio = repositorio;
    }

    @Autowired(required = false)
    public void setAuditoria(ServicioAuditoria auditoria) {
        this.auditoria = auditoria;
    }
}
```

**Por que la inyeccion por constructor es superior:**

1. **Inmutabilidad:** las dependencias se asignan a campos `final`.
2. **Obligatoriedad:** el constructor exige todas las dependencias. Si falta una, no compila.
3. **Testeabilidad:** en un test unitario, puedes pasar mocks al constructor sin Spring.
4. **Visibilidad:** una mirada al constructor revela todas las dependencias.

#### Comparacion: DI Manual vs DI con Framework

| Aspecto | DI Manual | DI con Spring |
|---------|-----------|---------------|
| Configuracion | Explicita en `main()` o fabricas | Anotaciones + configuracion |
| Flexibilidad | Maxima: control total | Alta: perfiles, `@Conditional` |
| Complejidad | Crece con el tamano | Spring gestiona el grafo |
| Arranque | Instantaneo | ~2-5 segundos (contexto Spring) |
| Testing | Muy simple | `@SpringBootTest` o unitarios sin Spring |
| Curva de aprendizaje | Baja (es `new`) | Alta (ciclo de vida, scopes) |
| Cuando usarlo | Proyectos pequenos, librerias | Apps empresariales, microservicios |


---

## 10.2 Patrones de Diseno

Los patrones de diseno son soluciones probadas a problemas recurrentes en el desarrollo de software. No son algoritmos concretos, sino plantillas que puedes adaptar. Conocerlos te da un vocabulario comun con otros desarrolladores: decir "aqui usaria un Observer" es mas preciso que describir todo el mecanismo.

### 10.2.1 Patrones Creacionales

Los patrones creacionales abstraen el proceso de instanciacion de objetos, haciendo el sistema independiente de como se crean, componen y representan sus objetos.

#### Singleton

**Implementacion con Enum (RECOMENDADA por Joshua Bloch en *Effective Java*):**

```java
public enum ConfiguracionEnum {
    INSTANCIA;

    ConfiguracionEnum() { System.out.println("ConfiguracionEnum inicializada"); }
    public String getValor(String clave) { return "valor para " + clave; }
}
// Uso: ConfiguracionEnum.INSTANCIA.getValor("timeout");
```

Los enums son thread-safe por definicion, protegidos contra serializacion y reflexion.

**Eager (inicializacion temprana):**

```java
public class ConfiguracionEager {
    private static final ConfiguracionEager INSTANCIA = new ConfiguracionEager();

    private ConfiguracionEager() {
        System.out.println("ConfiguracionEager inicializada");
    }

    public static ConfiguracionEager getInstancia() { return INSTANCIA; }
    public String getValor(String clave) { return "valor para " + clave; }
}
```

Thread-safe sin sincronizacion. Desventaja: la instancia se crea aunque nunca se use.

**Lazy con Double-Checked Locking:**

```java
public class ConfiguracionLazy {
    private static volatile ConfiguracionLazy instancia;

    private ConfiguracionLazy() {
        System.out.println("ConfiguracionLazy inicializada");
    }

    public static ConfiguracionLazy getInstancia() {
        if (instancia == null) {
            synchronized (ConfiguracionLazy.class) {
                if (instancia == null) {
                    instancia = new ConfiguracionLazy();
                }
            }
        }
        return instancia;
    }
}
```

`volatile` garantiza visibilidad entre hilos desde Java 5.

**Cuando usar Singleton?** Cuando exactamente un recurso debe ser compartido (pool de conexiones, cache, gestor de configuracion).

**Cuando evitarlo?** Cuando introduce estado global mutable (dificulta el testing) o cuando el requisito "unico" puede cambiar.

#### Factory Method

```java
public interface Transporte { void entregar(); }

public class Camion implements Transporte {
    @Override public void entregar() { System.out.println("Entregando por tierra en camion."); }
}
public class Barco implements Transporte {
    @Override public void entregar() { System.out.println("Entregando por mar en barco."); }
}
public class Avion implements Transporte {
    @Override public void entregar() { System.out.println("Entregando por aire en avion."); }
}

// Creador abstracto
public abstract class Logistica {
    protected abstract Transporte crearTransporte();

    public void planificarEntrega() {
        Transporte transporte = crearTransporte();
        System.out.println("Planificando ruta...");
        transporte.entregar();
    }
}

public class LogisticaTerrestre extends Logistica {
    @Override protected Transporte crearTransporte() { return new Camion(); }
}
public class LogisticaMaritima extends Logistica {
    @Override protected Transporte crearTransporte() { return new Barco(); }
}
public class LogisticaAerea extends Logistica {
    @Override protected Transporte crearTransporte() { return new Avion(); }
}

// Factory Method parametrizado (fabrica simple)
public class LogisticaFactory {
    public static Transporte crearTransporte(TipoTransporte tipo) {
        return switch (tipo) {
            case CAMION -> new Camion();
            case BARCO  -> new Barco();
            case AVION  -> new Avion();
        };
    }
    public enum TipoTransporte { CAMION, BARCO, AVION }
}
```

#### Abstract Factory

Crea **familias** de objetos relacionados. Diferencia clave con Factory Method: FM crea un solo producto, AF crea familias completas.

```java
public interface Boton { void renderizar(); void onClick(); }
public interface Checkbox { void renderizar(); boolean estaMarcado(); }

public interface GUIFactory { Boton crearBoton(); Checkbox crearCheckbox(); }

public class WindowsFactory implements GUIFactory {
    @Override public Boton crearBoton() { return new BotonWindows(); }
    @Override public Checkbox crearCheckbox() { return new CheckboxWindows(); }
}

public class MacOSFactory implements GUIFactory {
    @Override public Boton crearBoton() { return new BotonMacOS(); }
    @Override public Checkbox crearCheckbox() { return new CheckboxMacOS(); }
}

// Cliente: no sabe que fabrica concreta usa
public class Aplicacion {
    private final Boton boton;
    private final Checkbox checkbox;

    public Aplicacion(GUIFactory factory) {
        this.boton = factory.crearBoton();
        this.checkbox = factory.crearCheckbox();
    }

    public void renderizar() { boton.renderizar(); checkbox.renderizar(); }

    public static void main(String[] args) {
        GUIFactory factory = System.getProperty("os.name").contains("Mac")
                ? new MacOSFactory() : new WindowsFactory();
        Aplicacion app = new Aplicacion(factory);
        app.renderizar();
    }
}
```

#### Builder

Separa la construccion de un objeto complejo de su representacion. Ideal para objetos con muchos parametros opcionales.

```java
public class Pizza {
    private final String tamanio;
    private final String masa;
    private final boolean quesoExtra;
    private final boolean bordeRelleno;
    private final String queso;
    private final String proteina;
    private final String vegetal1;
    private final String vegetal2;

    private Pizza(Builder builder) {
        this.tamanio = builder.tamanio;
        this.masa = builder.masa;
        this.quesoExtra = builder.quesoExtra;
        this.bordeRelleno = builder.bordeRelleno;
        this.queso = builder.queso;
        this.proteina = builder.proteina;
        this.vegetal1 = builder.vegetal1;
        this.vegetal2 = builder.vegetal2;
    }

    public static class Builder {
        private final String tamanio;
        private final String masa;
        private boolean quesoExtra = false;
        private boolean bordeRelleno = false;
        private String queso = "Mozzarella";
        private String proteina = null;
        private String vegetal1 = null;
        private String vegetal2 = null;

        public Builder(String tamanio, String masa) {
            this.tamanio = tamanio;
            this.masa = masa;
        }

        public Builder quesoExtra(boolean val) { quesoExtra = val; return this; }
        public Builder bordeRelleno(boolean val) { bordeRelleno = val; return this; }
        public Builder queso(String val) { queso = val; return this; }
        public Builder proteina(String val) { proteina = val; return this; }
        public Builder vegetal1(String val) { vegetal1 = val; return this; }
        public Builder vegetal2(String val) { vegetal2 = val; return this; }

        public Pizza build() {
            if (tamanio == null || masa == null)
                throw new IllegalStateException("Tamano y masa son obligatorios");
            return new Pizza(this);
        }
    }

    @Override
    public String toString() {
        return String.format("Pizza [%s, %s, queso=%s, proteina=%s]",
                tamanio, masa, queso, proteina);
    }
}

// Uso: lectura clara, sin ambiguedad de parametros posicionales
Pizza pizza = new Pizza.Builder("Grande", "Integral")
        .quesoExtra(true)
        .proteina("Pepperoni")
        .vegetal1("Champinones")
        .vegetal2("Pimientos")
        .build();
```

**Builder en el JDK:** `StringBuilder`, `StringBuffer`, `Stream.Builder<T>`.

#### Prototype

Permite crear nuevos objetos clonando una instancia prototipica. Prefiere el Copy Constructor sobre el roto `Cloneable`:

```java
public class Documento {
    private String titulo;
    private String contenido;
    private List<String> autores;

    public Documento(String titulo, String contenido, List<String> autores) {
        this.titulo = titulo;
        this.contenido = contenido;
        this.autores = new ArrayList<>(autores);
    }

    // Copy constructor: explicito, no depende de magia
    public Documento(Documento original) {
        this.titulo = original.titulo;
        this.contenido = original.contenido;
        this.autores = new ArrayList<>(original.autores);
    }
}
```

**Problema con `Cloneable`:** es una interfaz rota. No declara `clone()` (esta en `Object`), y `clone()` es `protected`. Ademas, la clonacion superficial de `Object.clone()` copia referencias, no objetos. El copy constructor es explicito, seguro y no requiere casting.

---

### 10.2.2 Patrones Estructurales

Los patrones estructurales se ocupan de como componer clases y objetos para formar estructuras mas grandes, manteniendo la flexibilidad.

#### Adapter

Convierte la interfaz de una clase en otra que el cliente espera.

**Ejemplo: conectar una API externa de meteorologia:**

```java
// Interfaz que nuestra aplicacion espera
public interface ServicioClima {
    double obtenerTemperaturaCelsius(String ciudad);
    String obtenerCondicion(String ciudad);
}

// API externa (no podemos modificarla)
public class WeatherAPIExterna {
    public double getTemperatureF(String city) { return 75.0; } // Fahrenheit
    public int getWeatherCode(String city) { return 2; }        // 2=Nublado
}

// Object Adapter (composicion — preferido en Java)
public class WeatherAPIAdapter implements ServicioClima {
    private final WeatherAPIExterna apiExterna;

    public WeatherAPIAdapter(WeatherAPIExterna apiExterna) {
        this.apiExterna = apiExterna;
    }

    @Override
    public double obtenerTemperaturaCelsius(String ciudad) {
        double fahrenheit = apiExterna.getTemperatureF(ciudad);
        return (fahrenheit - 32) * 5.0 / 9.0;
    }

    @Override
    public String obtenerCondicion(String ciudad) {
        return switch (apiExterna.getWeatherCode(ciudad)) {
            case 1 -> "Soleado";
            case 2 -> "Nublado";
            case 3 -> "Lluvia";
            default -> "Desconocido";
        };
    }
}
```

**Object Adapter vs Class Adapter:**
- **Object Adapter** usa composicion. Es mas flexible porque puede adaptar una clase y todas sus subclases.
- **Class Adapter** usa herencia multiple. En Java solo funciona si lo adaptado es una interfaz.

#### Decorator

Anade responsabilidades adicionales a un objeto dinamicamente, proporcionando una alternativa flexible a la herencia.

```java
public interface Notificador { void enviar(String mensaje); }

public class NotificadorBasico implements Notificador {
    @Override public void enviar(String m) { System.out.println("Enviando: " + m); }
}

public abstract class NotificadorDecorador implements Notificador {
    protected final Notificador wrappee;
    protected NotificadorDecorador(Notificador w) { this.wrappee = w; }
    @Override public void enviar(String m) { wrappee.enviar(m); }
}

public class NotificadorEmail extends NotificadorDecorador {
    public NotificadorEmail(Notificador w) { super(w); }
    @Override public void enviar(String m) {
        super.enviar(m); System.out.println("[Email] " + m);
    }
}

public class NotificadorSMS extends NotificadorDecorador {
    public NotificadorSMS(Notificador w) { super(w); }
    @Override public void enviar(String m) {
        super.enviar(m); System.out.println("[SMS] " + m);
    }
}

public class NotificadorSlack extends NotificadorDecorador {
    public NotificadorSlack(Notificador w) { super(w); }
    @Override public void enviar(String m) {
        super.enviar(m); System.out.println("[Slack] " + m);
    }
}

// Uso: apilar decoradores en tiempo de ejecucion
Notificador completo = new NotificadorSlack(
        new NotificadorSMS(
                new NotificadorEmail(
                        new NotificadorBasico())));
completo.enviar("Alerta importante");
```

**Analogia con Java I/O:**

```java
InputStream file = new FileInputStream("datos.txt");
InputStream buffered = new BufferedInputStream(file);
InputStream gzip = new GZIPInputStream(buffered);
DataInputStream data = new DataInputStream(gzip);
int valor = data.readInt(); // Todas las capacidades combinadas
```

#### Facade

Proporciona una interfaz simplificada a un conjunto de interfaces en un subsistema complejo.

```java
// Subsistema complejo
public class CPU {
    public void congelar() { System.out.println("CPU: congelando..."); }
    public void ejecutar() { System.out.println("CPU: ejecutando..."); }
    public void saltar(long pos) { System.out.println("CPU: saltando a " + pos); }
}
public class Memoria {
    public void cargar(long pos, byte[] datos) {
        System.out.println("Memoria: cargando en posicion " + pos);
    }
}
public class DiscoDuro {
    public byte[] leer(long sector, int tam) {
        System.out.println("DiscoDuro: leyendo sector " + sector);
        return new byte[tam];
    }
}

// Facade: interfaz simple para el cliente
public class FachadaComputadora {
    private final CPU cpu = new CPU();
    private final Memoria memoria = new Memoria();
    private final DiscoDuro disco = new DiscoDuro();

    public void encender() {
        System.out.println("=== Iniciando computadora ===");
        cpu.congelar();
        byte[] boot = disco.leer(0, 1024);
        memoria.cargar(0, boot);
        cpu.saltar(0); cpu.ejecutar();
        System.out.println("=== Computadora lista ===");
    }
}
```

#### Proxy

Proporciona un sustituto o marcador de posicion para otro objeto, controlando el acceso a el.

**Virtual Proxy (carga perezosa de objetos costosos):**

```java
public interface Imagen { void mostrar(); }

public class ImagenReal implements Imagen {
    private final String nombreArchivo;

    public ImagenReal(String nombreArchivo) {
        this.nombreArchivo = nombreArchivo;
        cargarDesdeDisco(); // Costoso
    }

    private void cargarDesdeDisco() {
        System.out.println("Cargando " + nombreArchivo + " desde disco...");
    }

    @Override public void mostrar() { System.out.println("Mostrando: " + nombreArchivo); }
}

public class ImagenProxy implements Imagen {
    private final String nombreArchivo;
    private ImagenReal imagenReal;

    public ImagenProxy(String nombreArchivo) { this.nombreArchivo = nombreArchivo; }

    @Override
    public void mostrar() {
        if (imagenReal == null) {
            imagenReal = new ImagenReal(nombreArchivo);
        }
        imagenReal.mostrar();
    }
}
```

**Protection Proxy (control de acceso):**

```java
public interface Documento { void leer(); void escribir(String contenido); }

public class DocumentoProxy implements Documento {
    private final DocumentoReal documento;
    private final String rolUsuario;

    public DocumentoProxy(DocumentoReal documento, String rolUsuario) {
        this.documento = documento; this.rolUsuario = rolUsuario;
    }

    @Override public void leer() { documento.leer(); }

    @Override
    public void escribir(String contenido) {
        if ("ADMIN".equals(rolUsuario)) documento.escribir(contenido);
        else throw new SecurityException("Solo ADMIN puede escribir");
    }
}
```

---

### 10.2.3 Patrones de Comportamiento

Los patrones de comportamiento se centran en la comunicacion entre objetos.

#### Observer

Define una dependencia uno-a-muchos: cuando un objeto cambia de estado, todos sus dependientes son notificados.

```java
public interface Observador { void actualizar(String noticia); }

public class AgenciaNoticias {
    private final List<Observador> observadores = new ArrayList<>();
    private String ultimaNoticia;

    public void suscribir(Observador obs) { observadores.add(obs); }
    public void desuscribir(Observador obs) { observadores.remove(obs); }

    public void publicarNoticia(String noticia) {
        this.ultimaNoticia = noticia;
        for (Observador obs : observadores) {
            obs.actualizar(ultimaNoticia);
        }
    }
}

public class CanalTelevision implements Observador {
    private final String nombre;
    public CanalTelevision(String nombre) { this.nombre = nombre; }

    @Override public void actualizar(String noticia) {
        System.out.printf("[TV %s] Ultima hora: %s%n", nombre, noticia);
    }
}

public class Periodico implements Observador {
    private final String nombre;
    public Periodico(String nombre) { this.nombre = nombre; }

    @Override public void actualizar(String noticia) {
        System.out.printf("[%s] Titular: %s%n", nombre, noticia);
    }
}
```

**Usando `PropertyChangeListener` de Java:**

```java
import java.beans.PropertyChangeListener;
import java.beans.PropertyChangeSupport;

public class ModeloDatos {
    private final PropertyChangeSupport soporte = new PropertyChangeSupport(this);
    private String valor;

    public void addObserver(PropertyChangeListener listener) {
        soporte.addPropertyChangeListener(listener);
    }

    public void setValor(String nuevoValor) {
        String anterior = this.valor;
        this.valor = nuevoValor;
        soporte.firePropertyChange("valor", anterior, nuevoValor);
    }
}

ModeloDatos modelo = new ModeloDatos();
modelo.addObserver(evt -> {
    System.out.printf("Cambio: '%s' -> '%s'%n",
            evt.getOldValue(), evt.getNewValue());
});
modelo.setValor("Hola");
modelo.setValor("Mundo");
```

#### Strategy

Define una familia de algoritmos, encapsula cada uno y los hace intercambiables.

```java
public interface EstrategiaEnvio {
    double calcularCosto(double pesoKg, double distanciaKm);
}

public class EnvioTerrestre implements EstrategiaEnvio {
    @Override public double calcularCosto(double p, double d) { return p * 0.5 + d * 0.1; }
}
public class EnvioAereo implements EstrategiaEnvio {
    @Override public double calcularCosto(double p, double d) { return p * 2.0 + d * 0.5; }
}
public class EnvioMaritimo implements EstrategiaEnvio {
    @Override public double calcularCosto(double p, double d) { return p * 0.3 + d * 0.05; }
}

// Contexto: usa la estrategia
public class CalculadoraEnvio {
    private EstrategiaEnvio estrategia;

    public void setEstrategia(EstrategiaEnvio estrategia) {
        this.estrategia = estrategia;
    }

    public double calcular(double peso, double distancia) {
        if (estrategia == null) throw new IllegalStateException("Estrategia no definida");
        return estrategia.calcularCosto(peso, distancia);
    }
}
```

**Strategy en el JDK: `Comparator<T>` es el ejemplo canonico:**

```java
// Estrategia 1: por precio ascendente
productos.sort(Comparator.comparingDouble(Producto::getPrecio));

// Estrategia 2: por popularidad descendente
productos.sort(Comparator.comparingInt(Producto::getPopularidad).reversed());

// Estrategia compuesta: por popularidad desc, luego precio asc
productos.sort(Comparator.comparingInt(Producto::getPopularidad).reversed()
        .thenComparingDouble(Producto::getPrecio));
```

#### Command

Encapsula una peticion como un objeto. `Runnable` es el ejemplo mas puro de Command en Java.

```java
public interface Comando { void ejecutar(); void deshacer(); }

public class EditorTexto {
    private final StringBuilder contenido = new StringBuilder();

    public void insertar(int pos, String texto) { contenido.insert(pos, texto); }
    public void eliminar(int pos, int longi) { contenido.delete(pos, pos + longi); }
    public String getTexto() { return contenido.toString(); }
    public String getFragmento(int pos, int longi) {
        return contenido.substring(pos, pos + longi);
    }
}

public class ComandoInsertar implements Comando {
    private final EditorTexto editor;
    private final int posicion;
    private final String texto;

    public ComandoInsertar(EditorTexto editor, int posicion, String texto) {
        this.editor = editor; this.posicion = posicion; this.texto = texto;
    }
    @Override public void ejecutar() { editor.insertar(posicion, texto); }
    @Override public void deshacer() { editor.eliminar(posicion, texto.length()); }
}

public class ComandoEliminar implements Comando {
    private final EditorTexto editor;
    private final int posicion, longitud;
    private String textoEliminado;

    public ComandoEliminar(EditorTexto editor, int posicion, int longitud) {
        this.editor = editor; this.posicion = posicion; this.longitud = longitud;
    }

    @Override public void ejecutar() {
        textoEliminado = editor.getFragmento(posicion, longitud);
        editor.eliminar(posicion, longitud);
    }
    @Override public void deshacer() { editor.insertar(posicion, textoEliminado); }
}

// Historial con deshacer
public class HistorialEditor {
    private final Deque<Comando> historial = new ArrayDeque<>();
    public void ejecutarComando(Comando comando) {
        comando.ejecutar();
        historial.push(comando);
    }
    public void deshacer() {
        if (!historial.isEmpty()) historial.pop().deshacer();
    }
}
```

**Runnable como Command en el JDK:**

```java
Runnable tarea1 = () -> System.out.println("Ejecutando tarea 1");
Runnable tarea2 = () -> System.out.println("Ejecutando tarea 2");
ExecutorService executor = Executors.newFixedThreadPool(2);
executor.submit(tarea1); executor.submit(tarea2);
executor.shutdown();
```

#### Template Method

Define el esqueleto de un algoritmo, difiriendo algunos pasos a las subclases.

```java
public abstract class ProcesadorDatos {

    // Template Method: define el esqueleto del algoritmo
    public final void procesar(String origen) {
        String datos = leerDatos(origen);
        String datosValidados = validar(datos);
        String datosTransformados = transformar(datosValidados);
        guardarResultado(datosTransformados);
    }

    protected abstract String leerDatos(String origen);

    protected String validar(String datos) {
        if (datos == null || datos.isBlank())
            throw new IllegalArgumentException("Datos vacios o nulos");
        return datos;
    }

    protected abstract String transformar(String datos);

    // Hook: opcional, las subclases pueden sobrescribir
    protected void antesDeGuardar(String datos) { }

    private void guardarResultado(String datos) {
        antesDeGuardar(datos);
        System.out.println("Resultado guardado: " + datos);
    }
}

public class ProcesadorCSV extends ProcesadorDatos {
    @Override protected String leerDatos(String o) {
        return "nombre,edad,ciudad\\nAna,28,Madrid";
    }
    @Override protected String transformar(String d) {
        return "{nombre: 'Ana', edad: 28}";
    }
}

public class ProcesadorXML extends ProcesadorDatos {
    @Override protected String leerDatos(String o) {
        return "<usuario><nombre>Ana</nombre></usuario>";
    }
    @Override protected String transformar(String d) {
        return "{nombre: 'Ana'}";
    }
    @Override protected void antesDeGuardar(String datos) {
        System.out.println("[METRICAS] Procesando " + datos.length() + " caracteres");
    }
}
```

**Template Method en el JDK:** `java.util.AbstractList`, `java.io.InputStream`, `javax.servlet.http.HttpServlet` (define `doGet()`, `doPost()` como pasos; `service()` es el template method).

---
### 10.2.4 MVC / MVP / MVVM — El patron mas importante para aplicaciones

El patron Modelo-Vista-Controlador (y sus variantes MVP y MVVM) es la columna vertebral de casi toda aplicacion con interfaz de usuario. Aunque surgio en Smalltalk-80, su uso en Java es ubicuo: JavaFX, Swing, Spring MVC, JSF, Android.

```
     +----------+           +-----------+
     |  Modelo  |<--------->|  Control  |
     +----------+           +-----------+
          ^                      |
          |                      |
          v                      v
     +---------------------------------+
     |            Vista                 |
     +---------------------------------+
```

- **Modelo:** datos y logica de negocio. No sabe nada de la vista.
- **Vista:** representacion visual. Observa al modelo para actualizarse.
- **Controlador:** interpreta la entrada del usuario, actualiza el modelo.

#### Ejemplo en JavaFX

```java
// === MODELO ===
public class Tarea {
    private final StringProperty titulo = new SimpleStringProperty();
    private final BooleanProperty completada = new SimpleBooleanProperty();

    public Tarea(String titulo) {
        this.titulo.set(titulo);
        this.completada.set(false);
    }

    public StringProperty tituloProperty() { return titulo; }
    public BooleanProperty completadaProperty() { return completada; }
    public String getTitulo() { return titulo.get(); }
    public void setTitulo(String t) { titulo.set(t); }
    public boolean isCompletada() { return completada.get(); }
    public void setCompletada(boolean c) { completada.set(c); }
}

public class ListaTareasModel {
    private final ObservableList<Tarea> tareas = FXCollections.observableArrayList();

    public ObservableList<Tarea> getTareas() { return tareas; }
    public void agregarTarea(String titulo) { tareas.add(new Tarea(titulo)); }
    public void eliminarTarea(Tarea tarea) { tareas.remove(tarea); }
    public int getTotal() { return tareas.size(); }
    public long getCompletadas() {
        return tareas.stream().filter(Tarea::isCompletada).count();
    }
}
```

```java
// === VISTA ===
public class ListaTareasView {
    private final ListView<Tarea> listaView = new ListView<>();
    private final TextField campoTexto = new TextField();
    private final Button btnAgregar = new Button("Agregar");
    private final Button btnEliminar = new Button("Eliminar");
    private final Label lblEstadisticas = new Label("Total: 0, Completadas: 0");

    public ListView<Tarea> getListaView() { return listaView; }
    public TextField getCampoTexto() { return campoTexto; }
    public Button getBtnAgregar() { return btnAgregar; }
    public Button getBtnEliminar() { return btnEliminar; }
    public Label getLblEstadisticas() { return lblEstadisticas; }
}
```

```java
// === CONTROLADOR ===
public class ListaTareasController {
    private final ListaTareasModel modelo;
    private final ListaTareasView vista;

    public ListaTareasController(ListaTareasModel modelo, ListaTareasView vista) {
        this.modelo = modelo; this.vista = vista;
        configurarEventos(); bindearVistaModelo();
    }

    private void configurarEventos() {
        vista.getBtnAgregar().setOnAction(e -> {
            String texto = vista.getCampoTexto().getText().trim();
            if (!texto.isEmpty()) {
                modelo.agregarTarea(texto);
                vista.getCampoTexto().clear();
            }
        });

        vista.getBtnEliminar().setOnAction(e -> {
            Tarea seleccionada = vista.getListaView()
                    .getSelectionModel().getSelectedItem();
            if (seleccionada != null) modelo.eliminarTarea(seleccionada);
        });
    }

    private void bindearVistaModelo() {
        vista.getListaView().setItems(modelo.getTareas());
        modelo.getTareas().addListener((ListChangeListener<Tarea>) cambio ->
            vista.getLblEstadisticas().setText(
                String.format("Total: %d, Completadas: %d",
                    modelo.getTotal(), modelo.getCompletadas())));
    }
}
```

#### MVP (Model-View-Presenter)

En MVP, la Vista es pasiva: no conoce al modelo. El Presenter actua como intermediario total, haciendo la vista extremadamente testeable.

```java
// Contrato: define la interaccion entre Vista y Presenter
public interface TareasContract {
    interface View {
        void mostrarTareas(List<Tarea> tareas);
        void mostrarEstadisticas(int total, int completadas);
        void mostrarError(String mensaje);
        String getTextoNuevaTarea();
    }

    interface Presenter {
        void cargarTareas();
        void agregarTarea();
        void eliminarTarea(Tarea tarea);
        void marcarCompletada(Tarea tarea, boolean completada);
    }
}

public class TareasPresenter implements TareasContract.Presenter {
    private final TareasContract.View view;
    private final TareaRepository repository;

    public TareasPresenter(TareasContract.View view, TareaRepository repository) {
        this.view = view;
        this.repository = repository;
    }

    @Override
    public void cargarTareas() {
        List<Tarea> tareas = repository.obtenerTodas();
        view.mostrarTareas(tareas);
        view.mostrarEstadisticas(tareas.size(),
            (int) tareas.stream().filter(Tarea::isCompletada).count());
    }

    @Override
    public void agregarTarea() {
        String texto = view.getTextoNuevaTarea();
        if (texto == null || texto.trim().isEmpty()) {
            view.mostrarError("El titulo no puede estar vacio");
            return;
        }
        repository.guardar(new Tarea(texto));
        cargarTareas();
    }

    @Override
    public void eliminarTarea(Tarea tarea) {
        repository.eliminar(tarea);
        cargarTareas();
    }

    @Override
    public void marcarCompletada(Tarea tarea, boolean completada) {
        tarea.setCompletada(completada);
        repository.actualizar(tarea);
        cargarTareas();
    }
}
```

**Cuando usar cada variante:**
- **MVC:** aplicaciones web tradicionales, donde el controlador recibe HTTP y devuelve vistas.
- **MVP:** aplicaciones de escritorio y Android, donde la vista es pasiva y el presenter contiene toda la logica de presentacion (altamente testeable).
- **MVVM:** aplicaciones con data binding bidireccional (JavaFX con propiedades, Android con LiveData). La vista se actualiza automaticamente.

---

### 10.2.5 Repository Pattern

El patron Repository media entre el dominio y la capa de persistencia, actuando como una coleccion en memoria de objetos de dominio. Separa la logica de negocio de los detalles de almacenamiento.

```java
// Interfaz del repositorio (en el dominio, sin dependencias tecnicas)
public interface RepositorioPedidos {
    Optional<Pedido> buscarPorId(PedidoId id);
    List<Pedido> buscarPorCliente(ClienteId clienteId);
    List<Pedido> buscarPendientes();
    void guardar(Pedido pedido);
    void eliminar(PedidoId id);
    long contarPorEstado(EstadoPedido estado);
}

// Implementacion JPA (en infraestructura)
@Repository
public class JpaRepositorioPedidos implements RepositorioPedidos {
    private final JpaPedidoDao jpaDao;

    public JpaRepositorioPedidos(JpaPedidoDao jpaDao) { this.jpaDao = jpaDao; }

    @Override
    public Optional<Pedido> buscarPorId(PedidoId id) {
        return jpaDao.findById(id.getValor()).map(this::toDomain);
    }

    @Override
    public List<Pedido> buscarPorCliente(ClienteId clienteId) {
        return jpaDao.findByClienteId(clienteId.getValor())
                .stream().map(this::toDomain).toList();
    }

    @Override
    public void guardar(Pedido pedido) {
        PedidoEntity entity = toEntity(pedido);
        jpaDao.save(entity);
    }

    @Override
    public void eliminar(PedidoId id) { jpaDao.deleteById(id.getValor()); }

    private Pedido toDomain(PedidoEntity entity) { /* mapeo */ return null; }
    private PedidoEntity toEntity(Pedido pedido) { /* mapeo */ return null; }
}

// Implementacion en memoria (para tests unitarios)
public class EnMemoriaRepositorioPedidos implements RepositorioPedidos {
    private final Map<Long, Pedido> almacen = new ConcurrentHashMap<>();

    @Override
    public Optional<Pedido> buscarPorId(PedidoId id) {
        return Optional.ofNullable(almacen.get(id.getValor()));
    }

    @Override
    public List<Pedido> buscarPorCliente(ClienteId clienteId) {
        return almacen.values().stream()
                .filter(p -> p.getClienteId().equals(clienteId)).toList();
    }

    @Override
    public void guardar(Pedido pedido) {
        almacen.put(pedido.getId().getValor(), pedido);
    }

    @Override
    public void eliminar(PedidoId id) { almacen.remove(id.getValor()); }

    @Override
    public List<Pedido> buscarPendientes() {
        return almacen.values().stream()
                .filter(p -> p.getEstado() == EstadoPedido.PENDIENTE).toList();
    }

    @Override
    public long contarPorEstado(EstadoPedido estado) {
        return almacen.values().stream()
                .filter(p -> p.getEstado() == estado).count();
    }
}
```

**Beneficios del Repository Pattern:**
- El dominio no conoce JPA, JDBC ni MongoDB. Solo conoce la interfaz `RepositorioPedidos`.
- Podemos cambiar la implementacion de persistencia sin tocar la logica de negocio.
- Los tests unitarios del servicio usan la implementacion en memoria; los de integracion usan la real con Testcontainers.

---

### 10.2.6 Service Layer — Servicio de Dominio vs Servicio de Aplicacion

La capa de servicio define el limite de la aplicacion y coordina el flujo de trabajo. Es importante distinguir dos tipos:

| | Servicio de Dominio | Servicio de Aplicacion |
|---|---|---|
| Que contiene | Logica de negocio pura | Orquestacion de casos de uso |
| Transacciones | No | Si (`@Transactional`) |
| Accede a repositorios | Solo a los necesarios | Coordina multiples repositorios |
| Ejemplo | `CalculadoraPrecios`, `ValidadorPedido` | `ServicioCrearPedido`, `ServicioFacturacion` |

```java
// Servicio de Dominio: logica de negocio pura, sin dependencias externas
public class CalculadoraPrecios {
    private final PoliticaDescuentos politicaDescuentos;

    public CalculadoraPrecios(PoliticaDescuentos politicaDescuentos) {
        this.politicaDescuentos = politicaDescuentos;
    }

    public Precio calcularPrecioFinal(Pedido pedido) {
        BigDecimal subtotal = pedido.getLineas().stream()
                .map(l -> l.getPrecioUnitario()
                        .multiply(BigDecimal.valueOf(l.getCantidad())))
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal descuento = politicaDescuentos.calcular(pedido);
        BigDecimal impuestos = calcularImpuestos(subtotal.subtract(descuento));

        return new Precio(subtotal, descuento, impuestos);
    }

    private BigDecimal calcularImpuestos(BigDecimal base) {
        return base.multiply(new BigDecimal("0.21"));
    }
}
```

```java
// Servicio de Aplicacion: orquesta el caso de uso completo
@Service
@Transactional
public class ServicioCrearPedido {

    private final RepositorioPedidos repositorioPedidos;
    private final RepositorioClientes repositorioClientes;
    private final RepositorioProductos repositorioProductos;
    private final CalculadoraPrecios calculadoraPrecios;
    private final ServicioNotificacion notificador;
    private final PublicadorEventos publicadorEventos;

    public ServicioCrearPedido(RepositorioPedidos repositorioPedidos,
                                RepositorioClientes repositorioClientes,
                                RepositorioProductos repositorioProductos,
                                CalculadoraPrecios calculadoraPrecios,
                                ServicioNotificacion notificador,
                                PublicadorEventos publicadorEventos) {
        this.repositorioPedidos = repositorioPedidos;
        this.repositorioClientes = repositorioClientes;
        this.repositorioProductos = repositorioProductos;
        this.calculadoraPrecios = calculadoraPrecios;
        this.notificador = notificador;
        this.publicadorEventos = publicadorEventos;
    }

    public PedidoId ejecutar(CrearPedidoComando comando) {
        // 1. Validar existencia del cliente
        Cliente cliente = repositorioClientes.buscarPorId(comando.clienteId())
                .orElseThrow(() ->
                    new ClienteNoEncontradoException(comando.clienteId()));

        // 2. Construir lineas de pedido (validando disponibilidad)
        List<LineaPedido> lineas = comando.lineas().stream()
                .map(l -> {
                    Producto producto = repositorioProductos
                            .buscarPorId(l.productoId())
                            .orElseThrow(() ->
                                new ProductoNoEncontradoException(l.productoId()));
                    if (producto.getStock() < l.cantidad()) {
                        throw new StockInsuficienteException(
                                producto.getId(), l.cantidad());
                    }
                    return new LineaPedido(producto, l.cantidad(),
                            producto.getPrecioUnitario());
                }).toList();

        // 3. Crear el pedido
        Pedido pedido = new Pedido(cliente.getId(), lineas);

        // 4. Calcular precios (servicio de dominio)
        pedido.asignarPrecio(calculadoraPrecios.calcularPrecioFinal(pedido));

        // 5. Guardar
        repositorioPedidos.guardar(pedido);

        // 6. Notificar y publicar eventos
        notificador.enviar(cliente.getEmail(),
                "Pedido #" + pedido.getId() + " creado. Total: "
                + pedido.getTotal());
        publicadorEventos.publicar(new PedidoCreadoEvento(pedido));

        return pedido.getId();
    }
}
```

La diferencia es clara: el servicio de aplicacion **orquesta** y maneja transacciones. El servicio de dominio **calcula** y contiene las reglas de negocio puras. Esta separacion hace que las reglas de negocio sean testeables sin base de datos ni frameworks.

---

### 10.2.7 DTO vs Entity

Una fuente constante de confusion y codigo boilerplate. La distincion es simple pero crucial:

| | Entity | DTO |
|---|---|---|
| Proposito | Representa una fila en la BD | Transportar datos entre capas |
| Ciclo de vida | Gestionado por JPA/Hibernate | Efimero: se crea, se envia, se descarta |
| Identidad | Tiene identidad (`@Id`) | No tiene identidad propia |
| Relaciones | `@OneToMany`, `@ManyToOne` con lazy loading | Datos planos, posiblemente anidados |
| Mutabilidad | Mutable (JPA necesita setters) | Preferiblemente inmutable (records) |
| Capa | Capa de persistencia | Capa de presentacion / API |

```java
// === ENTITY: vive en la capa de persistencia ===
@Entity
@Table(name = "usuarios")
public class UsuarioEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String nombre;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(name = "password_hash", nullable = false)
    private String passwordHash;

    @Enumerated(EnumType.STRING)
    private RolUsuario rol;

    @OneToMany(mappedBy = "usuario", fetch = FetchType.LAZY)
    private List<PedidoEntity> pedidos = new ArrayList<>();

    @CreationTimestamp
    private LocalDateTime fechaCreacion;

    // Getters y setters (JPA los necesita)
}
```

```java
// === DTOs: viven en la capa de presentacion ===
public record UsuarioDTO(
    Long id, String nombre, String email,
    String rol, LocalDateTime fechaCreacion
) {}

public record CrearUsuarioRequest(
    @NotBlank String nombre,
    @Email @NotBlank String email,
    @Size(min = 8) String password,
    RolUsuario rol
) {}

public record UsuarioResponse(
    Long id, String nombre, String email,
    String rol, LocalDateTime fechaCreacion
) {}
```

```java
// === ENSAMBLADOR ===

// Opcion 1: Manual (mas control, mas codigo)
public class UsuarioMapper {
    public UsuarioResponse toResponse(UsuarioEntity entity) {
        return new UsuarioResponse(
            entity.getId(), entity.getNombre(), entity.getEmail(),
            entity.getRol().name(), entity.getFechaCreacion());
    }

    public UsuarioEntity toEntity(CrearUsuarioRequest request) {
        UsuarioEntity entity = new UsuarioEntity();
        entity.setNombre(request.nombre());
        entity.setEmail(request.email());
        entity.setPasswordHash(hashPassword(request.password()));
        entity.setRol(request.rol());
        return entity;
    }

    private String hashPassword(String raw) {
        return BCrypt.hashpw(raw, BCrypt.gensalt());
    }
}

// Opcion 2: MapStruct (genera codigo en compilacion, sin reflection)
@Mapper(componentModel = "spring")
public interface UsuarioMapStructMapper {

    UsuarioResponse toResponse(UsuarioEntity entity);

    @Mapping(target = "passwordHash",
             expression = "java(hashPassword(request.password()))")
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "pedidos", ignore = true)
    @Mapping(target = "fechaCreacion", ignore = true)
    UsuarioEntity toEntity(CrearUsuarioRequest request);

    default String hashPassword(String raw) {
        return BCrypt.hashpw(raw, BCrypt.gensalt());
    }
}
```

**Recomendacion:** usa metodos manuales para casos simples (menos dependencias), y MapStruct para proyectos grandes (type-safe, sin overhead de reflection, compila a codigo Java puro). Evita ModelMapper en proyectos serios: los errores de mapeo se detectan en tiempo de ejecucion, no en compilacion.

---

### 10.2.8 Chain of Responsibility — Middleware y Filtros

Evita acoplar el emisor de una peticion a su receptor, dando a mas de un objeto la oportunidad de manejar la peticion. Es el patron detras de los filtros de servlets, los middleware de Spring, y los interceptores.

```java
// Pipeline de validacion para crear un pedido
public abstract class ValidadorPedido {
    protected ValidadorPedido siguiente;

    public ValidadorPedido encadenar(ValidadorPedido siguiente) {
        this.siguiente = siguiente;
        return siguiente;
    }

    public abstract ResultadoValidacion validar(Pedido pedido);

    protected ResultadoValidacion validarSiguiente(Pedido pedido) {
        if (siguiente == null) return ResultadoValidacion.valido();
        return siguiente.validar(pedido);
    }
}

public class ValidadorStock extends ValidadorPedido {
    private final RepositorioProductos productos;

    public ValidadorStock(RepositorioProductos productos) {
        this.productos = productos;
    }

    @Override
    public ResultadoValidacion validar(Pedido pedido) {
        for (LineaPedido linea : pedido.getLineas()) {
            Producto producto = productos
                    .buscarPorId(linea.getProductoId()).orElse(null);
            if (producto == null)
                return ResultadoValidacion.error(
                    "Producto no encontrado: " + linea.getProductoId());
            if (producto.getStock() < linea.getCantidad())
                return ResultadoValidacion.error(
                    "Stock insuficiente para: " + producto.getNombre());
        }
        return validarSiguiente(pedido);
    }
}

public class ValidadorLimiteCredito extends ValidadorPedido {
    private final RepositorioClientes clientes;
    private static final BigDecimal LIMITE_CREDITO = new BigDecimal("5000");

    public ValidadorLimiteCredito(RepositorioClientes clientes) {
        this.clientes = clientes;
    }

    @Override
    public ResultadoValidacion validar(Pedido pedido) {
        Cliente cliente = clientes.buscarPorId(pedido.getClienteId())
                .orElse(null);
        if (cliente == null)
            return ResultadoValidacion.error("Cliente no encontrado");
        if (pedido.getTotal().compareTo(LIMITE_CREDITO) > 0
                && !cliente.isVerificado())
            return ResultadoValidacion.error(
                "Cliente no verificado excede limite de credito");
        return validarSiguiente(pedido);
    }
}

public class ValidadorFraude extends ValidadorPedido {
    private final ServicioAntifraude antifraude;

    public ValidadorFraude(ServicioAntifraude antifraude) {
        this.antifraude = antifraude;
    }

    @Override
    public ResultadoValidacion validar(Pedido pedido) {
        if (antifraude.esSospechoso(pedido))
            return ResultadoValidacion.error(
                "Pedido marcado como sospechoso por antifraude");
        return validarSiguiente(pedido);
    }
}

// Construccion de la cadena
ValidadorPedido pipeline = new ValidadorStock(productos);
pipeline.encadenar(new ValidadorLimiteCredito(clientes))
        .encadenar(new ValidadorFraude(antifraude));

// Uso
ResultadoValidacion resultado = pipeline.validar(pedido);
if (!resultado.esValido()) {
    throw new PedidoInvalidoException(resultado.getMensajesError());
}
```

**Chain of Responsibility en el JDK y frameworks:**
- `javax.servlet.Filter` y `FilterChain`
- Spring Security `SecurityFilterChain`
- `java.util.logging.Logger` con sus handlers en cascada
- Netty pipeline de handlers

---

### 10.2.9 CQRS Basico — Separar Lecturas de Escrituras

**CQRS (Command Query Responsibility Segregation)** propone usar modelos diferentes para leer informacion y para actualizarla. En su forma mas simple, significa separar los servicios de consulta de los de comando.

```
         ┌─────────────────────┐
         │      API / UI       │
         └──────┬──────┬───────┘
                │      │
        Comandos│      │Consultas
                │      │
         ┌──────▼──┐ ┌─▼───────────┐
         │ Escritura│ │  Lectura    │
         │ (Command)│ │  (Query)    │
         └────┬─────┘ └─┬───────────┘
              │          │
         ┌────▼──┐  ┌───▼──────┐
         │  BD   │  │BD Lectura│
         │Maestra│  │(cache/idx)│
         └───────┘  └──────────┘
```

```java
// === MODELO DE ESCRITURA (Commands) ===

// Comando: intencion de cambiar el estado
public record CrearPedidoCommand(
    Long clienteId, List<LineaPedidoCommand> lineas
) {
    public CrearPedidoCommand {
        Objects.requireNonNull(clienteId);
        if (lineas == null || lineas.isEmpty())
            throw new IllegalArgumentException("Pedido sin lineas");
    }
}

public record LineaPedidoCommand(Long productoId, int cantidad) {}

// Handler de comando
@Service
@Transactional
public class CrearPedidoCommandHandler {
    private final RepositorioPedidos repositorioPedidos;
    private final RepositorioProductos repositorioProductos;
    private final CalculadoraPrecios calculadoraPrecios;

    public PedidoId handle(CrearPedidoCommand command) {
        List<LineaPedido> lineas = command.lineas().stream()
                .map(l -> {
                    Producto p = repositorioProductos
                            .buscarPorId(l.productoId())
                            .orElseThrow(() ->
                                new ProductoNoEncontradoException(l.productoId()));
                    p.reservarStock(l.cantidad());
                    repositorioProductos.guardar(p);
                    return new LineaPedido(p.getId(), l.cantidad(),
                            p.getPrecioUnitario());
                }).toList();

        Pedido pedido = new Pedido(
                new ClienteId(command.clienteId()), lineas);
        Precio precio = calculadoraPrecios.calcularPrecioFinal(pedido);
        pedido.asignarPrecio(precio);
        repositorioPedidos.guardar(pedido);
        return pedido.getId();
    }
}
```

```java
// === MODELO DE LECTURA (Queries) ===

// Query: solicitud de informacion
public record BuscarPedidosClienteQuery(Long clienteId) {}

public record PedidoResumenDTO(
    Long id, String estado, BigDecimal total,
    LocalDateTime fecha, int numeroLineas
) {}

// Handler de query (puede usar BD de lectura, Elasticsearch, etc.)
@Service
public class BuscarPedidosClienteQueryHandler {
    private final PedidoReadRepository readRepository;

    public List<PedidoResumenDTO> handle(BuscarPedidosClienteQuery query) {
        return readRepository.findResumenByClienteId(query.clienteId());
    }
}

// Repositorio de solo lectura optimizado para consultas
public interface PedidoReadRepository {
    List<PedidoResumenDTO> findResumenByClienteId(Long clienteId);
    Optional<PedidoResumenDTO> findResumenById(Long id);
    long countByEstadoAndClienteId(String estado, Long clienteId);
}

@Repository
public class JdbcPedidoReadRepository implements PedidoReadRepository {
    private final JdbcTemplate jdbc;

    @Override
    public List<PedidoResumenDTO> findResumenByClienteId(Long clienteId) {
        return jdbc.query(
            "SELECT p.id, p.estado, p.total, p.fecha_creacion, " +
            "       (SELECT COUNT(*) FROM lineas_pedido lp " +
            "        WHERE lp.pedido_id = p.id) as lineas " +
            "FROM pedidos p " +
            "WHERE p.cliente_id = ? " +
            "ORDER BY p.fecha_creacion DESC",
            (rs, rowNum) -> new PedidoResumenDTO(
                rs.getLong("id"), rs.getString("estado"),
                rs.getBigDecimal("total"),
                rs.getTimestamp("fecha_creacion").toLocalDateTime(),
                rs.getInt("lineas")),
            clienteId);
    }

    @Override
    public Optional<PedidoResumenDTO> findResumenById(Long id) {
        List<PedidoResumenDTO> results = jdbc.query(
            "SELECT p.id, p.estado, p.total, p.fecha_creacion, " +
            "       (SELECT COUNT(*) FROM lineas_pedido lp " +
            "        WHERE lp.pedido_id = p.id) as lineas " +
            "FROM pedidos p WHERE p.id = ?",
            (rs, n) -> new PedidoResumenDTO(
                rs.getLong("id"), rs.getString("estado"),
                rs.getBigDecimal("total"),
                rs.getTimestamp("fecha_creacion").toLocalDateTime(),
                rs.getInt("lineas")), id);
        return results.isEmpty() ? Optional.empty()
                : Optional.of(results.get(0));
    }
}
```

**Por que separar?**
- Las consultas pueden usar SQL directo, vistas materializadas, o incluso Elasticsearch, sin afectar el modelo de dominio.
- Las escrituras usan el modelo de dominio rico con validaciones, invariantes y eventos.
- Facilita la optimizacion independiente: escalar lecturas con replicas de solo lectura, caches, etc.
- En su forma completa, CQRS con Event Sourcing permite reconstruir el estado desde el historial de eventos.

**No necesitas CQRS completo desde el dia uno.** Empieza separando servicios de comando y query. Introduce event sourcing o BD de lectura separada solo cuando la necesidad de escalado lo justifique.

## 10.3 Arquitectura de Software

Los patrones de diseno resuelven problemas a nivel de clases. La arquitectura resuelve problemas a nivel de sistema: como organizamos los modulos? Como fluyen las dependencias? Que pasa si cambiamos la base de datos?

### 10.3.1 Arquitectura en Capas

La arquitectura mas tradicional y punto de partida para entender las demas. Organiza el codigo en capas horizontales, donde cada capa solo depende de la capa inmediatamente inferior.

```
+----------------------------------------------------------+
|           PRESENTACION (Controllers)                      |
|  - REST Controllers, GraphQL resolvers                    |
|  - DTOs de entrada/salida                                 |
|  - Validacion de entrada                                  |
+----------------------------------------------------------+
|           APLICACION (Use Cases)                          |
|  - Servicios de aplicacion                                |
|  - Orquestacion, transacciones                            |
|  - Puertos (interfaces) hacia infraestructura             |
+----------------------------------------------------------+
|              DOMINIO (Core)                               |
|  - Entidades, Value Objects, Agregados                    |
|  - Servicios de dominio                                   |
|  - Interfaces de repositorio (puertos)                    |
|  - Reglas de negocio                                      |
+----------------------------------------------------------+
|         INFRAESTRUCTURA (Technical)                        |
|  - JPA/Hibernate, JDBC                                    |
|  - Clientes REST, mensajeria                              |
|  - Configuracion de Spring, logging                       |
|  - Implementaciones de puertos                            |
+----------------------------------------------------------+
```

**Regla fundamental:** las capas superiores dependen de las inferiores. El dominio no depende de nada externo. La infraestructura depende del dominio (implementa sus interfaces).

```java
// DOMINIO: no conoce Spring, JPA, ni detalles tecnicos
public class Pedido {
    private PedidoId id;
    private ClienteId clienteId;
    private List<LineaPedido> lineas;
    private EstadoPedido estado;
    private Precio precio;

    public void confirmar() {
        if (estado != EstadoPedido.PENDIENTE)
            throw new IllegalStateException(
                "Solo pedidos pendientes pueden confirmarse");
        this.estado = EstadoPedido.CONFIRMADO;
    }

    public BigDecimal getTotal() { return precio.total(); }
}

// APLICACION: orquesta el caso de uso
@Service
public class ConfirmarPedidoUseCase {
    private final RepositorioPedidos pedidos; // Interfaz del dominio

    @Transactional
    public void ejecutar(PedidoId id) {
        Pedido pedido = pedidos.buscarPorId(id)
                .orElseThrow(() -> new PedidoNoEncontradoException(id));
        pedido.confirmar(); // Logica de dominio
        pedidos.guardar(pedido);
    }
}

// INFRAESTRUCTURA: implementa la interfaz del dominio
@Repository
public class JpaRepositorioPedidos implements RepositorioPedidos {
    @PersistenceContext private EntityManager em;
    // ... implementaciones JPA
}

// PRESENTACION: expone la API
@RestController
@RequestMapping("/api/pedidos")
public class PedidoController {
    private final ConfirmarPedidoUseCase confirmarPedido;

    @PostMapping("/{id}/confirmar")
    public ResponseEntity<Void> confirmar(@PathVariable Long id) {
        confirmarPedido.ejecutar(new PedidoId(id));
        return ResponseEntity.ok().build();
    }
}
```

**Ventajas:** simple de entender, excelente para proyectos pequenos y medianos, bien soportada por frameworks.

**Desventajas:** puede volverse "lasagna code" con demasiadas capas, la dependencia hacia abajo no evita que la logica de dominio se filtre a infraestructura.

---

### 10.3.2 Arquitectura Hexagonal (Ports & Adapters)

Propuesta por Alistair Cockburn, esta arquitectura coloca el dominio en el centro y todo lo externo (BD, APIs, UI, colas de mensajes) como "adaptadores" que se conectan a traves de "puertos" (interfaces).

```
            +--------------------------+
            |     Puerto (interfaz)    |   <-- Define QUE, no COMO
            +-----------+--------------+
                        |
        +---------------+---------------+
        |               |               |
  +-----v------+ +------v------+ +------v------+
  | Adaptador  | |  Adaptador  | |  Adaptador  |
  |   BD       | |   REST      | |  Mensajeria |
  | (JPA)      | |  (Client)   | |  (Kafka)    |
  +------------+ +-------------+ +-------------+
```

**Puertos primarios (driving):** como se USA el sistema desde fuera (REST, CLI, UI).

**Puertos secundarios (driven):** que necesita el dominio del exterior (BD, email, mensajeria).

#### Ejemplo completo: Servicio de Pedidos

```java
// ============ PUERTOS (interfaces en el dominio) ============

// Puerto primario (driving): define como se invoca el sistema
public interface PedidoService {
    PedidoId crearPedido(CrearPedidoRequest request);
    void confirmarPedido(PedidoId id);
    Optional<Pedido> consultarPedido(PedidoId id);
}

// Puertos secundarios (driven): que necesita el dominio
public interface PedidoRepository {
    Optional<Pedido> findById(PedidoId id);
    void save(Pedido pedido);
    List<Pedido> findByClienteId(ClienteId clienteId);
}

public interface NotificationPort {
    void sendOrderConfirmation(ClienteId clienteId, PedidoId pedidoId);
}

public interface EventPublisher {
    void publish(DomainEvent event);
}
```

```java
// ============ DOMINIO ============

public class PedidoServiceImpl implements PedidoService {
    private final PedidoRepository repository;
    private final NotificationPort notificationPort;
    private final EventPublisher eventPublisher;

    public PedidoServiceImpl(PedidoRepository repository,
                              NotificationPort notificationPort,
                              EventPublisher eventPublisher) {
        this.repository = repository;
        this.notificationPort = notificationPort;
        this.eventPublisher = eventPublisher;
    }

    @Override
    public PedidoId crearPedido(CrearPedidoRequest request) {
        Pedido pedido = Pedido.crear(request.clienteId(), request.lineas());
        repository.save(pedido);
        eventPublisher.publish(new PedidoCreadoEvent(pedido));
        return pedido.getId();
    }

    @Override
    public void confirmarPedido(PedidoId id) {
        Pedido pedido = repository.findById(id)
                .orElseThrow(() -> new PedidoNoEncontradoException(id));
        pedido.confirmar();
        repository.save(pedido);
        notificationPort.sendOrderConfirmation(
                pedido.getClienteId(), pedido.getId());
        eventPublisher.publish(new PedidoConfirmadoEvent(pedido));
    }

    @Override
    public Optional<Pedido> consultarPedido(PedidoId id) {
        return repository.findById(id);
    }
}
```

```java
// ============ ADAPTADORES ============

// Adaptador de BD (PostgreSQL via JPA)
@Repository
public class JpaPedidoRepository implements PedidoRepository {
    @PersistenceContext private EntityManager em;

    @Override
    public Optional<Pedido> findById(PedidoId id) {
        PedidoJpaEntity entity = em.find(
                PedidoJpaEntity.class, id.value());
        return Optional.ofNullable(entity).map(PedidoMapper::toDomain);
    }

    @Override
    public void save(Pedido pedido) {
        PedidoJpaEntity entity = PedidoMapper.toEntity(pedido);
        em.merge(entity);
    }
}

// Adaptador de notificaciones (Email)
@Service
public class EmailNotificationPort implements NotificationPort {
    private final JavaMailSender mailSender;
    private final ClienteRepository clienteRepository;

    @Override
    public void sendOrderConfirmation(ClienteId clienteId,
                                       PedidoId pedidoId) {
        Cliente cliente = clienteRepository.findById(clienteId)
                .orElseThrow();
        SimpleMailMessage email = new SimpleMailMessage();
        email.setTo(cliente.getEmail());
        email.setSubject("Pedido #" + pedidoId.value() + " confirmado");
        email.setText("Tu pedido ha sido confirmado.");
        mailSender.send(email);
    }
}

// Adaptador de mensajeria (Kafka)
@Service
public class KafkaEventPublisher implements EventPublisher {
    private final KafkaTemplate<String, DomainEvent> kafka;

    @Override
    public void publish(DomainEvent event) {
        kafka.send(event.getTopic(), event.getAggregateId(), event);
    }
}

// Adaptador primario: REST API
@RestController
@RequestMapping("/api/pedidos")
public class PedidoRestController {
    private final PedidoService pedidoService;

    public PedidoRestController(PedidoService pedidoService) {
        this.pedidoService = pedidoService;
    }

    @PostMapping
    public ResponseEntity<PedidoId> crear(
            @RequestBody CrearPedidoRequest request) {
        PedidoId id = pedidoService.crearPedido(request);
        return ResponseEntity.status(201).body(id);
    }

    @PostMapping("/{id}/confirmar")
    public ResponseEntity<Void> confirmar(@PathVariable Long id) {
        pedidoService.confirmarPedido(new PedidoId(id));
        return ResponseEntity.ok().build();
    }
}
```

**La belleza de la arquitectura hexagonal:** el dominio no sabe si los datos vienen de PostgreSQL, de un archivo CSV, o de una API REST. No sabe si las notificaciones se envian por email, SMS o Slack. Cambiar cualquiera de estos detalles no toca una sola linea del dominio.

---

### 10.3.3 Domain-Driven Design (DDD) Basico

DDD no es una arquitectura, sino un enfoque de modelado que pone el foco en el dominio del negocio y su logica. Se complementa perfectamente con la arquitectura hexagonal.

#### Conceptos fundamentales

**Entity (Entidad):** objeto con identidad continua a lo largo del tiempo. Dos entidades con los mismos atributos son diferentes si tienen distinta identidad.

**Value Object (Objeto de valor):** objeto sin identidad. Dos value objects con los mismos atributos son intercambiables. Son inmutables por definicion.

**Aggregate (Agregado):** cluster de entidades y value objects que se tratan como una unidad. Tiene una raiz (Aggregate Root) que es el unico punto de acceso desde fuera.

**Repository (Repositorio):** abstraccion de una coleccion de agregados. Solo hay repositorios para Aggregate Roots.

**Domain Service (Servicio de dominio):** logica que no pertenece naturalmente a una entidad o value object concreto.

**Domain Event (Evento de dominio):** algo importante que ocurrio en el dominio. Otros componentes pueden reaccionar a el.

**Bounded Context (Contexto acotado):** frontera semantica donde un modelo de dominio tiene un significado consistente. "Cliente" significa cosas distintas en Ventas y en Soporte.

#### Ejemplo: Modelar un Carrito de Compras con DDD

```
+-----------------------------------------------------------------------+
|  Bounded Context: Ventas                                               |
|                                                                       |
|  +--------------------------------------------------------------+     |
|  | Aggregate: Carrito                                             |     |
|  |                                                                 |     |
|  |  Carrito (Aggregate Root)                                       |     |
|  |    |-- CarritoId (Value Object)                                 |     |
|  |    |-- ClienteId (Value Object)                                 |     |
|  |    |-- List<LineaCarrito> (Entity)                              |     |
|  |    |    |-- ProductoId (Value Object)                           |     |
|  |    |    |-- Cantidad (Value Object)                             |     |
|  |    |    |-- PrecioUnitario (Value Object)                       |     |
|  |    |-- EstadoCarrito (Value Object)                             |     |
|  |    |-- FechaCreacion (Value Object)                             |     |
|  |    |-- CuponAplicado? (Value Object, opcional)                  |     |
|  +--------------------------------------------------------------+     |
|                                                                       |
|  Eventos de dominio: CarritoCreado, ProductoAgregado,                 |
|  ProductoEliminado, CuponAplicado, CarritoConvertido                  |
+-----------------------------------------------------------------------+
```

```java
// === VALUE OBJECTS ===

public record CarritoId(UUID value) {
    public CarritoId {
        Objects.requireNonNull(value);
    }
    public static CarritoId generar() {
        return new CarritoId(UUID.randomUUID());
    }
}

public record ClienteId(Long value) {
    public ClienteId {
        if (value == null || value <= 0)
            throw new IllegalArgumentException("ID de cliente invalido");
    }
}

public record ProductoId(Long value) {
    public ProductoId {
        if (value == null || value <= 0)
            throw new IllegalArgumentException("ID de producto invalido");
    }
}

public record Cantidad(int value) {
    public Cantidad {
        if (value < 1) throw new IllegalArgumentException(
                "Cantidad minima: 1");
        if (value > 99) throw new IllegalArgumentException(
                "Cantidad maxima: 99");
    }
    public Cantidad incrementar() { return new Cantidad(value + 1); }
    public Cantidad decrementar() { return new Cantidad(value - 1); }
}

public record Precio(BigDecimal amount, Moneda moneda) {
    public Precio {
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException(
                    "Precio no puede ser negativo");
    }
    public Precio multiplicar(int factor) {
        return new Precio(
                amount.multiply(BigDecimal.valueOf(factor)), moneda);
    }
}

public record Cupon(String codigo, BigDecimal porcentajeDescuento) {
    public Cupon {
        if (porcentajeDescuento.compareTo(BigDecimal.ZERO) <= 0
                || porcentajeDescuento.compareTo(
                        BigDecimal.valueOf(100)) > 0)
            throw new IllegalArgumentException("Descuento: 1-100%");
    }
    public Precio aplicar(Precio precio) {
        BigDecimal factor = BigDecimal.ONE.subtract(
                porcentajeDescuento.divide(BigDecimal.valueOf(100),
                        MathContext.DECIMAL32));
        return new Precio(
                precio.amount().multiply(factor), precio.moneda());
    }
}
```

```java
// === ENTIDAD (dentro del agregado) ===

public class LineaCarrito {
    private ProductoId productoId;
    private Cantidad cantidad;
    private Precio precioUnitario;
    private String nombreProducto;

    LineaCarrito(ProductoId productoId, Cantidad cantidad,
                  Precio precioUnitario, String nombre) {
        this.productoId = productoId;
        this.cantidad = cantidad;
        this.precioUnitario = precioUnitario;
        this.nombreProducto = nombre;
    }

    public void incrementarCantidad() {
        this.cantidad = cantidad.incrementar();
    }

    public void decrementarCantidad() {
        this.cantidad = cantidad.decrementar();
    }

    public Precio calcularSubtotal() {
        return precioUnitario.multiplicar(cantidad.value());
    }

    public ProductoId getProductoId() { return productoId; }
    public Cantidad getCantidad() { return cantidad; }
    public Precio getPrecioUnitario() { return precioUnitario; }
    public String getNombreProducto() { return nombreProducto; }
}

// === AGGREGATE ROOT ===

public class Carrito {
    private CarritoId id;
    private ClienteId clienteId;
    private List<LineaCarrito> lineas;
    private EstadoCarrito estado;
    private Cupon cupon;
    private LocalDateTime fechaCreacion;
    private List<DomainEvent> eventosPendientes;

    private Carrito(ClienteId clienteId) {
        this.id = CarritoId.generar();
        this.clienteId = clienteId;
        this.lineas = new ArrayList<>();
        this.estado = EstadoCarrito.ACTIVO;
        this.fechaCreacion = LocalDateTime.now();
        this.eventosPendientes = new ArrayList<>();
        this.eventosPendientes.add(
                new CarritoCreadoEvent(this.id, this.clienteId));
    }

    // Factory method
    public static Carrito crear(ClienteId clienteId) {
        if (clienteId == null)
            throw new IllegalArgumentException("Cliente requerido");
        return new Carrito(clienteId);
    }

    // Comandos del agregado (modifican estado)

    public void agregarProducto(ProductoId productoId, Cantidad cantidad,
                                 Precio precioUnitario, String nombre) {
        validarCarritoActivo();
        lineas.stream()
                .filter(l -> l.getProductoId().equals(productoId))
                .findFirst()
                .ifPresentOrElse(
                        LineaCarrito::incrementarCantidad,
                        () -> lineas.add(new LineaCarrito(
                                productoId, cantidad,
                                precioUnitario, nombre)));
        eventosPendientes.add(new ProductoAgregadoEvent(
                this.id, productoId, cantidad));
    }

    public void eliminarProducto(ProductoId productoId) {
        validarCarritoActivo();
        LineaCarrito linea = lineas.stream()
                .filter(l -> l.getProductoId().equals(productoId))
                .findFirst()
                .orElseThrow(() ->
                        new ProductoNoEncontradoException(productoId));
        lineas.remove(linea);
        eventosPendientes.add(
                new ProductoEliminadoEvent(this.id, productoId));
    }

    public void aplicarCupon(Cupon cupon) {
        validarCarritoActivo();
        if (this.cupon != null) throw new CuponYaAplicadoException();
        this.cupon = cupon;
        eventosPendientes.add(
                new CuponAplicadoEvent(this.id, cupon.codigo()));
    }

    public Pedido convertirAPedido() {
        validarCarritoActivo();
        if (lineas.isEmpty()) throw new CarritoVacioException();
        this.estado = EstadoCarrito.CONVERTIDO;
        eventosPendientes.add(
                new CarritoConvertidoEvent(this.id, this.clienteId));
        return Pedido.crearDesdeCarrito(this);
    }

    // Consultas

    public Precio calcularTotal() {
        Precio subtotal = lineas.stream()
                .map(LineaCarrito::calcularSubtotal)
                .reduce(new Precio(BigDecimal.ZERO,
                        lineas.get(0).getPrecioUnitario().moneda()),
                        (a, b) -> new Precio(
                                a.amount().add(b.amount()),
                                a.moneda()));
        return cupon != null ? cupon.aplicar(subtotal) : subtotal;
    }

    public int getNumeroArticulos() {
        return lineas.stream()
                .mapToInt(l -> l.getCantidad().value()).sum();
    }

    // Eventos pendientes para que la capa de aplicacion los publique
    public List<DomainEvent> getEventosPendientes() {
        return List.copyOf(eventosPendientes);
    }

    public void limpiarEventos() { eventosPendientes.clear(); }

    // Invariantes
    private void validarCarritoActivo() {
        if (estado != EstadoCarrito.ACTIVO)
            throw new CarritoNoActivoException(
                    "Carrito en estado: " + estado);
    }

    // Getters
    public CarritoId getId() { return id; }
    public ClienteId getClienteId() { return clienteId; }
    public List<LineaCarrito> getLineas() {
        return List.copyOf(lineas);
    }
    public EstadoCarrito getEstado() { return estado; }
    public Optional<Cupon> getCupon() {
        return Optional.ofNullable(cupon);
    }
}
```

```java
// === DOMAIN EVENTS ===

public interface DomainEvent {
    String getTopic();
    String getAggregateId();
    LocalDateTime getOccurredAt();
}

public record CarritoCreadoEvent(CarritoId carritoId,
        ClienteId clienteId) implements DomainEvent {
    @Override public String getTopic() {
        return "ventas.carrito.creado";
    }
    @Override public String getAggregateId() {
        return carritoId.value().toString();
    }
    @Override public LocalDateTime getOccurredAt() {
        return LocalDateTime.now();
    }
}

public record ProductoAgregadoEvent(CarritoId carritoId,
        ProductoId productoId, Cantidad cantidad)
        implements DomainEvent {
    @Override public String getTopic() {
        return "ventas.carrito.producto-agregado";
    }
    @Override public String getAggregateId() {
        return carritoId.value().toString();
    }
    @Override public LocalDateTime getOccurredAt() {
        return LocalDateTime.now();
    }
}

// === SERVICIO DE APLICACION ===

@Service
@Transactional
public class CarritoApplicationService {
    private final CarritoRepository carritoRepository;
    private final ProductoRepository productoRepository;
    private final EventPublisher eventPublisher;

    public CarritoId agregarProducto(ClienteId clienteId,
            ProductoId productoId, int cantidad) {
        Carrito carrito = carritoRepository
                .findActivoByClienteId(clienteId)
                .orElseGet(() -> Carrito.crear(clienteId));

        Producto producto = productoRepository.findById(productoId)
                .orElseThrow(() ->
                        new ProductoNoEncontradoException(productoId));

        carrito.agregarProducto(productoId,
                new Cantidad(cantidad),
                producto.getPrecio(), producto.getNombre());
        carritoRepository.save(carrito);

        carrito.getEventosPendientes()
                .forEach(eventPublisher::publish);
        carrito.limpiarEventos();
        return carrito.getId();
    }
}
```

#### Bounded Contexts — El mapa del dominio

```
+--------------+    +--------------+    +--------------+
|   Ventas     |--->|  Inventario  |<---|   Compras    |
|              |    |              |    |              |
| Cliente como |    | Producto como|    | Producto como|
| comprador    |    | stock y SKU  |    | item de orden|
+------+-------+    +--------------+    +--------------+
       |
       | PedidoConfirmadoEvent
       v
+--------------+    +--------------+
|   Envios     |    |  Facturacion |
| Domicilio,   |    | Impuestos,   |
| transportista|    | factura PDF  |
+--------------+    +--------------+
```

Cada bounded context tiene su propio modelo, su propio lenguaje ubicuo (terminos que el negocio usa), y se comunica con otros contextos mediante eventos de integracion. "Cliente" en Ventas es quien compra; "Cliente" en Envios es quien recibe el paquete. Son conceptos distintos aunque usen la misma palabra.

---

### 10.3.4 Event-Driven Architecture

En una arquitectura orientada a eventos, los componentes se comunican publicando y consumiendo eventos. No se llaman directamente. Esto desacopla servicios y permite que cada uno evolucione independientemente.

```
+----------+     Evento      +--------------+
| Servicio |  PedidoCreado   |   Message     |
|  Pedidos | --------------->|   Broker      |
+----------+                 | (RabbitMQ/   |
                              |  Kafka)      |
+----------+                 +---T---T---T---+
| Servicio |   Suscripcion      |   |   |
|  Email   |<-------------------+   |   |
+----------+                        |   |
+----------+                        |   |
| Servicio |<-----------------------+   |
|  Stock   |                            |
+----------+                            |
+----------+                            |
| Servicio |<---------------------------+
| Analisis |
+----------+
```

#### Ejemplo: Spring + Kafka

```java
// === Integration Event (contrato entre servicios) ===
@Builder
public record PedidoCreadoIntegrationEvent(
    Long pedidoId, Long clienteId, String clienteEmail,
    BigDecimal total, List<LineaPedido> lineas, Instant timestamp
) {
    public record LineaPedido(Long productoId, int cantidad) {}
}

// === Publicador (en el servicio de Pedidos) ===
@Service
public class PedidoEventPublisher {
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public void pedidoCreado(Pedido pedido) {
        PedidoCreadoIntegrationEvent event =
                PedidoCreadoIntegrationEvent.builder()
                .pedidoId(pedido.getId().getValor())
                .clienteId(pedido.getClienteId().getValor())
                .clienteEmail(pedido.getClienteEmail())
                .total(pedido.getTotal())
                .lineas(pedido.getLineas().stream()
                        .map(l -> new PedidoCreadoIntegrationEvent
                                .LineaPedido(l.getProductoId()
                                        .getValor(),
                                l.getCantidad().getValue()))
                        .toList())
                .timestamp(Instant.now())
                .build();

        kafkaTemplate.send("pedidos.creados", event);
    }
}
```

```java
// === Consumidor: Servicio de Stock (descuenta inventario) ===
@Service
public class StockEventHandler {
    private final ProductoRepository productoRepository;

    @KafkaListener(topics = "pedidos.creados",
                   groupId = "stock-service")
    public void handlePedidoCreado(
            PedidoCreadoIntegrationEvent event) {
        log.info("Descontando stock para pedido {}",
                event.pedidoId());
        for (var linea : event.lineas()) {
            Producto producto = productoRepository
                    .findById(new ProductoId(linea.productoId()))
                    .orElseThrow();
            producto.reservarStock(linea.cantidad());
            productoRepository.save(producto);
        }
    }
}

// === Consumidor: Servicio de Email ===
@Service
public class EmailEventHandler {
    private final EmailService emailService;

    @KafkaListener(topics = "pedidos.creados",
                   groupId = "email-service")
    public void handlePedidoCreado(
            PedidoCreadoIntegrationEvent event) {
        emailService.enviarConfirmacion(
                event.clienteEmail(),
                event.pedidoId(), event.total());
    }
}

// === Consumidor: Servicio de Analisis ===
@Service
public class AnalyticsEventHandler {

    @KafkaListener(topics = "pedidos.creados",
                   groupId = "analytics-service")
    public void handlePedidoCreado(
            PedidoCreadoIntegrationEvent event) {
        analyticsRepository.save(new PedidoAnalytics(
                event.pedidoId(), event.clienteId(),
                event.total(), event.timestamp(),
                event.lineas().size()));
    }
}
```

**Eventos de dominio vs Eventos de integracion:**
- **Eventos de dominio:** internos al bounded context. Pueden ser ricos en objetos de dominio.
- **Eventos de integracion:** cruzan bounded contexts. Son DTOs planos, estables, con esquema versionado.

**Garantias de entrega:**
- **At-most-once:** se envia una vez, sin reintentos (puede perderse).
- **At-least-once:** se reenvia hasta confirmacion (puede duplicarse, requiere idempotencia).
- **Exactly-once:** se entrega exactamente una vez (Kafka lo soporta con transacciones).

---

### 10.3.5 Clean Architecture

Propuesta por Robert C. Martin, la Clean Architecture lleva la inversion de dependencias al extremo: las dependencias siempre apuntan hacia adentro. El centro es el dominio puro, sin ningun framework.

```
 +------------------------------------------------------+
 |           Frameworks & Drivers (Web, DB)              |
 |  +--------------------------------------------------+ |
 |  |        Interface Adapters (Controllers,          | |
 |  |        Gateways, Presenters)                     | |
 |  |  +---------------------------------------------+ | |
 |  |  |      Application (Use Cases)                | | |
 |  |  |  +---------------------------------------+  | | |
 |  |  |  |          Domain (Entities)            |  | | |
 |  |  |  +---------------------------------------+  | | |
 |  |  +---------------------------------------------+ | |
 |  +--------------------------------------------------+ |
 +------------------------------------------------------+
```

**La regla de dependencia:** el codigo en un circulo interior no puede saber nada de un circulo exterior.

```java
// === DOMINIO: Entidades puras, sin anotaciones ===
public class Usuario {
    private UsuarioId id;
    private String nombre;
    private Email email;
    private PasswordHash passwordHash;

    public void cambiarNombre(String nuevoNombre) {
        if (nuevoNombre == null || nuevoNombre.trim().isEmpty())
            throw new IllegalArgumentException(
                    "Nombre no puede estar vacio");
        this.nombre = nuevoNombre.trim();
    }
}

public record UsuarioId(UUID value) {}
public record Email(String value) {
    public Email {
        if (!value.contains("@"))
            throw new IllegalArgumentException("Email invalido");
    }
}
```

```java
// === CASO DE USO (Interactor): Java puro, sin frameworks ===
public class RegistrarUsuarioUseCase {

    public interface RegistroOutput {
        void usuarioRegistrado(UsuarioId id);
        void emailYaExiste(Email email);
    }

    private final UsuarioRepository repository;
    private final PasswordEncoder passwordEncoder;

    public UsuarioId ejecutar(RegistrarUsuarioRequest request,
                               RegistroOutput output) {
        if (repository.existePorEmail(request.email())) {
            output.emailYaExiste(request.email());
            return null;
        }

        Usuario usuario = new Usuario(
            UsuarioId.generar(), request.nombre(),
            request.email(),
            passwordEncoder.encode(request.password()));
        repository.guardar(usuario);
        output.usuarioRegistrado(usuario.getId());
        return usuario.getId();
    }
}

public record RegistrarUsuarioRequest(
        String nombre, Email email, String password) {}
```

```java
// === ADAPTADOR: Controller REST ===
@RestController
@RequestMapping("/api/usuarios")
public class RegistrarUsuarioController {
    private final RegistrarUsuarioUseCase useCase;

    @PostMapping
    public ResponseEntity<?> registrar(
            @RequestBody RegistrarUsuarioRequest request) {
        try {
            UsuarioId id = useCase.ejecutar(request,
                new RegistrarUsuarioUseCase.RegistroOutput() {
                    @Override
                    public void usuarioRegistrado(UsuarioId id) {}
                    @Override
                    public void emailYaExiste(Email email) {
                        throw new EmailYaExisteException(
                                email.value());
                    }
                });
            return ResponseEntity.status(201)
                    .body(Map.of("id", id.value()));
        } catch (EmailYaExisteException e) {
            return ResponseEntity.status(409)
                    .body(Map.of("error", e.getMessage()));
        }
    }
}
```

La gracia de Clean Architecture: el caso de uso `RegistrarUsuarioUseCase` no importa Spring, no conoce HTTP, no depende de JPA. Es Java puro. Puedes testearlo con un repositorio en memoria sin levantar ningun contenedor.

**Cuando usar Clean Architecture:** sistemas con logica de negocio compleja, equipos grandes, cuando la testabilidad extrema es un requisito.

**Cuando NO usarla:** CRUDs simples (sobre-ingenieria duele), prototipos y MVPs, microservicios triviales con poca logica de dominio.

## 10.4 Testing Profesional

Probar no es opcional. Es lo que te permite dormir tranquilo el viernes despues del deploy. Un buen suite de tests es la documentacion mas fiable de tu codigo: si los tests pasan, el codigo funciona como se espera.

### 10.4.1 La Piramide de Testing

```
        +------+
        | E2E  |  <-- Pocos, lentos, validan flujos completos
       +--------+
       | Integr.|  <-- Verifican integracion entre componentes
      +----------+
      | Unitarios | <-- Muchos, rapidos, aislan unidades de codigo
      +------------+
```

**Principio fundamental:** escribe muchos tests unitarios, menos de integracion, y pocos end-to-end. La rapidez de los unitarios permite ejecutarlos constantemente durante el desarrollo. Un test que tarda mas de unos segundos en ejecutarse deja de ser util como feedback inmediato.

### 10.4.2 Test-Driven Development (TDD): Red-Green-Refactor

TDD es una disciplina de diseno, no solo de testing. El ciclo es:

1. **RED:** escribe un test que falle (porque el codigo no existe aun).
2. **GREEN:** escribe el minimo codigo para que el test pase.
3. **REFACTOR:** mejora el codigo sin cambiar su comportamiento (los tests siguen en verde).

#### Ejemplo practico completo: Calculadora de Descuentos con TDD

**Requisito:** los clientes VIP tienen 20% de descuento, los regulares 5%, y el descuento maximo es de $500.

**Iteracion 1: RED — Test que falla**

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CalculadoraDescuentosTest {

    @Test
    void clienteVipObtiene20PorCientoDescuento() {
        CalculadoraDescuentos calc = new CalculadoraDescuentos();
        BigDecimal resultado = calc.calcular(
                new BigDecimal("1000"), TipoCliente.VIP);
        assertEquals(new BigDecimal("200.00"), resultado);
    }
}
```

Este test no compila: `CalculadoraDescuentos` y `TipoCliente` no existen aun.

**Iteracion 1: GREEN**

```java
public enum TipoCliente { REGULAR, VIP }

public class CalculadoraDescuentos {
    public BigDecimal calcular(BigDecimal monto, TipoCliente tipo) {
        if (tipo == TipoCliente.VIP) return new BigDecimal("200.00");
        return BigDecimal.ZERO;
    }
}
```

Pasa. Es el codigo mas simple posible.

**Iteracion 2: RED — Anadir cliente regular**

```java
@Test
void clienteRegularObtiene5PorCientoDescuento() {
    CalculadoraDescuentos calc = new CalculadoraDescuentos();
    BigDecimal resultado = calc.calcular(
            new BigDecimal("1000"), TipoCliente.REGULAR);
    assertEquals(new BigDecimal("50.00"), resultado);
}
```

Falla porque devuelve `0` para REGULAR.

**Iteracion 2: GREEN**

```java
public BigDecimal calcular(BigDecimal monto, TipoCliente tipo) {
    if (tipo == TipoCliente.VIP)
        return monto.multiply(new BigDecimal("0.20"));
    return monto.multiply(new BigDecimal("0.05"));
}
```

**Iteracion 3: RED — Descuento maximo de $500**

```java
@Test
void descuentoNoSuperaMaximoDe500() {
    CalculadoraDescuentos calc = new CalculadoraDescuentos();
    BigDecimal resultado = calc.calcular(
            new BigDecimal("10000"), TipoCliente.VIP);
    // 20% de 10000 = 2000, pero maximo es 500
    assertEquals(new BigDecimal("500.00"), resultado);
}
```

**Iteracion 3: GREEN**

```java
private static final BigDecimal MAX_DESCUENTO = new BigDecimal("500.00");

public BigDecimal calcular(BigDecimal monto, TipoCliente tipo) {
    BigDecimal descuento;
    if (tipo == TipoCliente.VIP)
        descuento = monto.multiply(new BigDecimal("0.20"));
    else
        descuento = monto.multiply(new BigDecimal("0.05"));
    return descuento.min(MAX_DESCUENTO);
}
```

**Iteracion 4: RED — Monto cero devuelve cero**

```java
@Test
void montoCeroDevuelveCero() {
    CalculadoraDescuentos calc = new CalculadoraDescuentos();
    assertEquals(BigDecimal.ZERO,
            calc.calcular(BigDecimal.ZERO, TipoCliente.VIP));
}
```

**Iteracion 4: GREEN**

```java
public BigDecimal calcular(BigDecimal monto, TipoCliente tipo) {
    if (monto.compareTo(BigDecimal.ZERO) <= 0) return BigDecimal.ZERO;
    BigDecimal descuento;
    if (tipo == TipoCliente.VIP)
        descuento = monto.multiply(new BigDecimal("0.20"));
    else
        descuento = monto.multiply(new BigDecimal("0.05"));
    return descuento.min(MAX_DESCUENTO);
}
```

**Iteracion 5: REFACTOR — Codigo mas expresivo**

```java
public class CalculadoraDescuentos {
    private static final BigDecimal MAX_DESCUENTO = new BigDecimal("500.00");
    private static final BigDecimal TASA_VIP = new BigDecimal("0.20");
    private static final BigDecimal TASA_REGULAR = new BigDecimal("0.05");

    public BigDecimal calcular(BigDecimal monto, TipoCliente tipo) {
        if (monto == null || monto.compareTo(BigDecimal.ZERO) <= 0)
            return BigDecimal.ZERO;

        BigDecimal tasa = switch (tipo) {
            case VIP -> TASA_VIP;
            case REGULAR -> TASA_REGULAR;
        };

        BigDecimal descuentoCalculado = monto.multiply(tasa);
        return descuentoCalculado.min(MAX_DESCUENTO);
    }
}
```

TDD nos dio: (1) codigo que funciona, (2) tests que documentan el comportamiento, (3) diseno simple guiado por necesidades reales, no por especulacion.

### 10.4.3 JUnit 5 y Mockito: Fundamentos

#### JUnit 5 esencial

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculadoraTest {

    private Calculadora calc;

    @BeforeAll
    static void inicializarTodo() {
        System.out.println("@BeforeAll: Configuracion global");
    }

    @BeforeEach
    void inicializar() { calc = new Calculadora(); }

    @Test
    @DisplayName("Suma de dos numeros positivos")
    void sumaBasica() {
        assertEquals(5, calc.sumar(2, 3), "2 + 3 debe ser 5");
    }

    @Test
    @DisplayName("Division por cero lanza excepcion")
    void divisionPorCeroLanzaExcepcion() {
        ArithmeticException ex = assertThrows(
                ArithmeticException.class,
                () -> calc.dividir(10, 0));
        assertEquals("/ by zero", ex.getMessage());
    }

    @Test
    @DisplayName("Multiples verificaciones a la vez")
    void operacionesMultiples() {
        assertAll("Calculadora",
                () -> assertEquals(5, calc.sumar(2, 3)),
                () -> assertEquals(6, calc.multiplicar(2, 3)),
                () -> assertTrue(calc.sumar(1, 1) > 0),
                () -> assertFalse(calc.restar(1, 5) > 0));
    }

    @Test
    void timeoutEnOperacionLenta() {
        assertTimeout(Duration.ofMillis(100), () -> {
            assertEquals(4, calc.sumar(2, 2));
        });
    }

    @AfterEach void limpiar() { calc = null; }

    @AfterAll
    static void limpiarTodo() {
        System.out.println("@AfterAll: Limpieza global");
    }
}
```

#### Tests Parametrizados

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;

class CalculadoraParametrizadaTest {
    private final Calculadora calc = new Calculadora();

    @ParameterizedTest(name = "{0} + {1} = {2}")
    @CsvSource({
        "1, 1, 2",
        "2, 3, 5",
        "10, -5, 5",
        "0, 0, 0"
    })
    void sumaParametrizada(int a, int b, int esperado) {
        assertEquals(esperado, calc.sumar(a, b));
    }

    @ParameterizedTest
    @ValueSource(strings = {"", " ", "   "})
    void cadenasVacias(String entrada) {
        assertTrue(entrada.isBlank());
    }

    @ParameterizedTest
    @MethodSource("proveerDatosDivision")
    void divisionParametrizada(int a, int b, int esperado) {
        assertEquals(esperado, calc.dividir(a, b));
    }

    static Stream<Arguments> proveerDatosDivision() {
        return Stream.of(
            Arguments.of(10, 2, 5),
            Arguments.of(100, 25, 4),
            Arguments.of(9, 3, 3));
    }
}
```

#### Mockito: Fundamentos

```java
@ExtendWith(MockitoExtension.class)
class ServicioUsuariosTest {

    @Mock
    private RepositorioUsuarios repositorio;

    @InjectMocks
    private ServicioUsuarios servicio;

    @Test
    void obtenerUsuario_existente() {
        Usuario esperado = new Usuario(1L, "Ana", "ana@email.com");
        when(repositorio.buscarPorId(1L))
                .thenReturn(Optional.of(esperado));

        Usuario resultado = servicio.obtenerUsuario(1L);

        assertEquals("Ana", resultado.getNombre());
        verify(repositorio).buscarPorId(1L);
    }

    @Test
    void obtenerUsuario_noExistente_lanzaExcepcion() {
        when(repositorio.buscarPorId(99L))
                .thenReturn(Optional.empty());

        assertThrows(IllegalArgumentException.class,
                () -> servicio.obtenerUsuario(99L));
    }

    @Test
    void crearUsuario_emailDuplicado_lanzaExcepcion() {
        Usuario existente = new Usuario(null, "Ana", "ana@email.com");
        when(repositorio.existePorEmail("ana@email.com"))
                .thenReturn(true);

        assertThrows(IllegalStateException.class,
                () -> servicio.crearUsuario(existente));
        verify(repositorio, never()).guardar(any());
    }

    @Test
    void crearUsuario_emailNuevo_guardaCorrectamente() {
        Usuario nuevo = new Usuario(null, "Carlos", "carlos@email.com");
        when(repositorio.existePorEmail("carlos@email.com"))
                .thenReturn(false);

        servicio.crearUsuario(nuevo);

        verify(repositorio).guardar(nuevo);
    }
}
```

### 10.4.4 Mockito Avanzado

#### @Spy: mock parcial (el objeto real con algunos metodos simulados)

```java
@Spy
private ServicioNotificacionEmail notificador;

@Test
void spyEjemplo() {
    // El metodo enviar() usa la implementacion real...
    notificador.configurarServidor("smtp.example.com");

    // ...pero podemos simular metodos especificos
    doReturn(true).when(notificador).conexionDisponible();

    notificador.enviar("user@email.com", "Hola");
    verify(notificador).enviar("user@email.com", "Hola");
}
```

#### @Captor y ArgumentCaptor

```java
@Captor
private ArgumentCaptor<Pedido> pedidoCaptor;

@Test
void captorEjemplo() {
    Pedido pedido = new Pedido(/* ... */);
    servicio.procesar(pedido);

    verify(repositorioPedidos).guardar(pedidoCaptor.capture());
    Pedido pedidoGuardado = pedidoCaptor.getValue();

    assertEquals(pedido.getClienteId(),
            pedidoGuardado.getClienteId());
    assertTrue(pedidoGuardado.getEstado() == EstadoPedido.PROCESADO);
}

// Capturar multiples invocaciones
@Test
void capturarTodosLosPagos() {
    servicio.pagar(new Pago(1, new BigDecimal("100")));
    servicio.pagar(new Pago(2, new BigDecimal("200")));

    verify(procesadorPagos, times(2))
            .procesar(pagoCaptor.capture());

    List<Pago> pagosProcesados = pagoCaptor.getAllValues();
    assertEquals(2, pagosProcesados.size());
    assertEquals(new BigDecimal("200"), pagosProcesados.get(1).monto());
}
```

#### InOrder: verificar orden de invocaciones

```java
@Test
void verificarOrdenDeInvocaciones() {
    servicio.procesarPedido(pedido);

    InOrder inOrder = inOrder(repositorioPedidos, notificador);

    // Primero se guarda, luego se notifica
    inOrder.verify(repositorioPedidos).guardar(pedido);
    inOrder.verify(notificador).enviar(anyString(), anyString());
}
```

#### doAnswer: comportamiento dinamico

```java
@Test
void doAnswerEjemplo() {
    List<String> notificacionesEnviadas = new ArrayList<>();

    doAnswer(invocation -> {
        String destinatario = invocation.getArgument(0);
        String mensaje = invocation.getArgument(1);
        notificacionesEnviadas.add(destinatario + ": " + mensaje);
        return true; // valor de retorno
    }).when(notificador).enviar(anyString(), anyString());

    servicio.notificarTodos("Mensaje");

    assertEquals(3, notificacionesEnviadas.size());
    assertTrue(notificacionesEnviadas.get(0)
            .contains("Mensaje"));
}
```

### 10.4.5 Property-Based Testing con jqwik

En lugar de escribir casos de prueba especificos, defines propiedades que deben cumplirse para cualquier valor de entrada valido. La libreria genera inputs aleatorios.

```java
import net.jqwik.api.*;
import static org.assertj.core.api.Assertions.*;

class OrdenarPropiedadTest {

    @Property
    boolean ordenarMasInvertirIgualOrdenarDescendente(
            @ForAll List<Integer> lista) {

        List<Integer> ordenada = lista.stream()
                .sorted().toList();
        List<Integer> invertida = new ArrayList<>(ordenada);
        Collections.reverse(invertida);

        // Propiedad: ordenar ascendente + invertir = ordenar descendente
        List<Integer> descendente = lista.stream()
                .sorted(Comparator.reverseOrder()).toList();

        return invertida.equals(descendente);
    }

    @Property
    boolean longitudDeListaNoCambiaAlFiltrar(
            @ForAll List<String> lista,
            @ForAll String prefijo) {

        List<String> filtrada = lista.stream()
                .filter(s -> s.startsWith(prefijo)).toList();

        // Propiedad: filtrar nunca aumenta el tamano
        return filtrada.size() <= lista.size();
    }

    @Property
    boolean sumaEsConmutativa(
            @ForAll @IntRange(min = -1000, max = 1000) int a,
            @ForAll @IntRange(min = -1000, max = 1000) int b) {

        return a + b == b + a;
    }
}
```

### 10.4.6 Mutation Testing con PITest

El mutation testing introduce pequenos cambios (mutaciones) en tu codigo y verifica si tus tests los detectan. Si una mutacion no hace fallar ningun test, tienes codigo no cubierto adecuadamente.

```java
// Codigo original
public boolean esMayorDeEdad(int edad) {
    return edad >= 18;
}

// Mutacion que PITest podria generar (cambia >= por >)
public boolean esMayorDeEdad(int edad) {
    return edad > 18;
}
```

Si tus tests no detectan este cambio (falta un test con edad=18), PITest te lo senala. Es como medir la cobertura, pero de verdad: mide si tus tests realmente verifican el comportamiento, no solo si pasan por las lineas.

**Configuracion Maven:**

```xml
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.16.1</version>
    <configuration>
        <targetClasses>
            <param>com.miapp.dominio.*</param>
        </targetClasses>
        <targetTests>
            <param>com.miapp.dominio.*</param>
        </targetTests>
        <mutators>
            <mutator>STRONGER</mutator>
        </mutators>
    </configuration>
</plugin>
```

```bash
mvn org.pitest:pitest-maven:mutationCoverage
# Genera reporte en target/pit-reports/
```

### 10.4.7 Test Fixtures: Object Mother y Test Data Builder

Crear objetos de prueba complejos puede ser tedioso. Dos patrones ayudan:

#### Object Mother

```java
public class UsuarioMother {
    public static Usuario usuarioValido() {
        return new Usuario(1L, "Maria Garcia",
                "maria@email.com", RolUsuario.CLIENTE);
    }

    public static Usuario usuarioVip() {
        return new Usuario(2L, "Carlos Lopez",
                "carlos@email.com", RolUsuario.VIP);
    }

    public static Usuario usuarioConEmail(String email) {
        return new Usuario(null, "Test User", email,
                RolUsuario.CLIENTE);
    }
}

// Uso en tests
@Test
void crearPedido_clienteVip() {
    Usuario cliente = UsuarioMother.usuarioVip();
    Pedido pedido = servicio.crearPedido(cliente,
            PedidoMother.pedidoMinimo());
    assertTrue(pedido.getTotal().esPositivo());
}
```

#### Test Data Builder

```java
public class PedidoBuilder {
    private ClienteId clienteId = new ClienteId(1L);
    private List<LineaPedido> lineas = List.of(
            new LineaPedido(new ProductoId(1L), 1,
                    new BigDecimal("100.00")));
    private EstadoPedido estado = EstadoPedido.PENDIENTE;

    public PedidoBuilder conCliente(ClienteId id) {
        this.clienteId = id; return this;
    }

    public PedidoBuilder conLineas(List<LineaPedido> lineas) {
        this.lineas = lineas; return this;
    }

    public PedidoBuilder enEstado(EstadoPedido estado) {
        this.estado = estado; return this;
    }

    public Pedido build() {
        Pedido pedido = new Pedido(clienteId, lineas);
        return pedido;
    }
}

// Uso fluido en tests
Pedido pedido = new PedidoBuilder()
        .conCliente(new ClienteId(5L))
        .enEstado(EstadoPedido.CONFIRMADO)
        .build();
```

### 10.4.8 Contract Testing con Pact

En arquitecturas de microservicios, los contract tests aseguran que el proveedor de una API y el consumidor cumplen el contrato acordado.

```java
// Test del lado CONSUMIDOR (el que llama a la API de Pedidos)
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "pedidos-service",
             port = "8888")
public class PedidosConsumerPactTest {

    @Pact(consumer = "carrito-service")
    public V4Pact crearPedidoPact(PactDslWithProvider builder) {
        return builder
            .given("existe un producto con ID 1")
            .uponReceiving("peticion para crear pedido")
                .path("/api/pedidos")
                .method("POST")
                .headers("Content-Type", "application/json")
                .body("""
                    {"clienteId": 1, "lineas": [
                        {"productoId": 1, "cantidad": 2}]}""")
            .willRespondWith()
                .status(201)
                .headers(Map.of("Content-Type", "application/json"))
                .body("""
                    {"id": 100, "estado": "PENDIENTE",
                     "total": 200.00}""")
            .toPact(V4Pact.class);
    }

    @Test
    @PactTestFor(pactMethod = "crearPedidoPact")
    void testCrearPedido(MockServer mockServer) {
        CarritoClient client = new CarritoClient(
                mockServer.getUrl());
        PedidoResponse response = client.crearPedido(
                new CrearPedidoRequest(1L, List.of(
                        new LineaPedidoRequest(1L, 2))));
        assertEquals(100L, response.id());
        assertEquals("PENDIENTE", response.estado());
    }
}
```

### 10.4.9 Integration Tests con Spring Boot

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class UsuarioControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private TestRestTemplate restTemplate;

    @MockBean
    private ServicioNotificacion notificacion;

    @Test
    void crearUsuario_retorna201() throws Exception {
        String requestJson = """
            {"nombre": "Test", "email": "test@email.com",
             "password": "password123"}""";

        MvcResult result = mockMvc.perform(
                post("/api/usuarios")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(requestJson))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").exists())
                .andExpect(jsonPath("$.nombre").value("Test"))
                .andReturn();
    }

    @Test
    void crearUsuario_emailDuplicado_retorna409() throws Exception {
        // Primero creamos uno
        ResponseEntity<UsuarioResponse> created = restTemplate
                .postForEntity("/api/usuarios",
                        new CrearUsuarioRequest("Dupe",
                                "dupe@email.com", "password123", null),
                        UsuarioResponse.class);
        assertEquals(201, created.getStatusCodeValue());

        // Luego intentamos crear otro con el mismo email
        ResponseEntity<Map> conflict = restTemplate
                .postForEntity("/api/usuarios",
                        new CrearUsuarioRequest("Otro",
                                "dupe@email.com", "password456", null),
                        Map.class);
        assertEquals(409, conflict.getStatusCodeValue());
    }
}
```

#### WireMock: simular APIs externas en tests de integracion

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@WireMockTest(httpPort = 8089)
class ServicioClimaIntegrationTest {

    @Autowired
    private ServicioClimaExterno servicioClima;

    @Test
    void obtenerClima_retornaDatosFormateados() {
        // Simular la API externa
        stubFor(get(urlPathEqualTo("/weather"))
                .withQueryParam("city", equalTo("Madrid"))
                .willReturn(aResponse()
                        .withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                            {"temp": 22.5, "condition": "Sunny"}""")));

        ClimaResult result = servicioClima.consultar("Madrid");

        assertEquals(22.5, result.getTemperatura());
        assertEquals("Soleado", result.getCondicion());

        // Verificar que efectivamente se llamo a la API simulada
        verify(getRequestedFor(urlPathEqualTo("/weather"))
                .withQueryParam("city", equalTo("Madrid")));
    }
}
```

### 10.4.10 Bateria de Tests Completa para un Servicio de Usuarios

Una estrategia de testing completa para un servicio incluye:

```
src/test/java/com/miapp/usuarios/
  domain/
    UsuarioTest.java                 // Tests unitarios de la entidad
    EmailTest.java                   // Tests de value objects
    ValidadorPasswordTest.java       // Tests de servicios de dominio
  application/
    ServicioUsuariosTest.java        // Tests unitarios del servicio (con mocks)
    RegistrarUsuarioUseCaseTest.java // Tests del caso de uso
  infrastructure/
    JpaRepositorioUsuariosTest.java  // Tests de integracion con BD
  web/
    UsuarioControllerTest.java       // Tests de controlador (MockMvc standalone)
    UsuarioControllerIntegrationTest.java // Tests de integracion completos
  contract/
    UsuarioApiContractTest.java      // Contract tests con Pact
```

Un test unitario rapido de logica de dominio:

```java
class UsuarioTest {
    @Test
    void cambiarEmail_validaFormato() {
        Usuario usuario = new Usuario("test", new Email("old@test.com"));
        usuario.cambiarEmail(new Email("new@test.com"));
        assertEquals(new Email("new@test.com"), usuario.getEmail());
    }

    @Test
    void emailInvalido_lanzaExcepcion() {
        assertThrows(IllegalArgumentException.class,
                () -> new Email("esto-no-es-email"));
    }
}
```

### 10.4.11 Principios FIRST para buenos tests

- **F**ast (rapidos): los tests deben ejecutarse en milisegundos. Evita llamadas a red, archivos o BD reales.
- **I**ndependent (independientes): un test no debe depender del estado dejado por otro.
- **R**epeatable (repetibles): mismo resultado en cualquier entorno, sin dependencias no controladas.
- **S**elf-validating (autovalidables): el test devuelve verde o rojo, sin requerir interpretacion humana.
- **T**horough (exhaustivos): cubrir casos felices, de error, casos limite y valores frontera.
- **T**imely (oportunos): escribir los tests antes del codigo (TDD) o inmediatamente despues.

### 10.4.12 Convenciones de nomenclatura

```java
// Estilo 1: given_when_then (mas descriptivo)
@Test
void givenUsuarioValido_whenCrearUsuario_thenSeGuardaEnRepositorio() { ... }

// Estilo 2: should_when (conciso)
@Test
void shouldThrowException_whenEmailYaExiste() { ... }

// Estilo 3: metodo que describe el comportamiento
@Test
void crearUsuarioConEmailDuplicadoLanzaExcepcion() { ... }
```

Elige un estilo y se consistente en todo el proyecto.

## 10.5 Observabilidad

No puedes arreglar lo que no puedes ver. La observabilidad va mas alla del logging tradicional: es la capacidad de entender el estado interno de un sistema a partir de sus salidas externas. Sus tres pilares son **logs, metricas y trazas**.

### 10.5.1 Logging Avanzado: SLF4J + Logback

**Logs estructurados (JSON):** en produccion, los logs en JSON permiten ser indexados por herramientas como ELK (Elasticsearch, Logstash, Kibana) o Loki.

```xml
<!-- logback.xml con salida JSON -->
<configuration>
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.JsonEncoder">
            <jsonGeneratorDecorator
                class="ch.qos.logback.contrib.json.classic.JsonGeneratorDecorator"/>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
        </encoder>
    </appender>

    <!-- Niveles por paquete -->
    <logger name="com.miapp.servicio" level="DEBUG"/>
    <logger name="org.springframework" level="WARN"/>
    <logger name="org.hibernate.SQL" level="WARN"/>
    <logger name="com.zaxxer.hikari" level="INFO"/>

    <root level="INFO">
        <appender-ref ref="JSON"/>
    </root>
</configuration>
```

**MDC (Mapped Diagnostic Context):** anade contexto a todas las entradas de log del hilo actual. Invaluable para correlacionar logs en sistemas concurrentes.

```java
import org.slf4j.MDC;
import java.util.UUID;

// En un filtro HTTP o interceptor
@Component
public class TraceFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) {

        String traceId = Optional.ofNullable(
                request.getHeader("X-Trace-Id"))
                .orElse(UUID.randomUUID().toString());
        String userId = request.getHeader("X-User-Id");

        MDC.put("traceId", traceId);
        MDC.put("userId", userId);
        MDC.put("endpoint", request.getRequestURI());

        response.setHeader("X-Trace-Id", traceId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear(); // CRITICO: evitar fugas
        }
    }
}
```

**Buenas practicas de logging:**

1. Usa el nivel adecuado: `TRACE` (debugging profundo), `DEBUG` (desarrollo), `INFO` (hitos importantes), `WARN` (situaciones potencialmente problematicas), `ERROR` (errores que impiden una operacion).
2. No registres informacion sensible: contrasenas, tokens, tarjetas de credito.
3. Parametriza siempre: `logger.info("Hola {}", nombre)` es mas eficiente que `logger.info("Hola " + nombre)`.
4. Loggea excepciones completas como ultimo argumento: `logger.error("Fallo al guardar", e)`.
5. Incluye contexto suficiente: "Error al procesar pedido ID={} para usuario={}" no "Error".
6. Usa MDC en aplicaciones web: `requestId`, `sessionId`, `userId` siempre.
7. Configura rotacion de archivos con `TimeBasedRollingPolicy`.

### 10.5.2 Metricas con Micrometer

Micrometer es la fachada de metricas para Spring Boot. Proporciona una API unificada para Prometheus, Datadog, New Relic, CloudWatch, etc.

```java
// Tipos de metricas

// Counter: solo incrementa (ej: numero de pedidos creados)
@Service
public class PedidoService {
    private final Counter pedidosCreados;

    public PedidoService(MeterRegistry registry) {
        this.pedidosCreados = Counter.builder("pedidos.creados")
                .description("Numero total de pedidos creados")
                .tag("version", "v1")
                .register(registry);
    }

    public Pedido crearPedido(CrearPedidoRequest request) {
        Pedido pedido = /* ... */;
        pedidosCreados.increment();
        return pedido;
    }
}

// Gauge: valor que sube y baja (ej: usuarios activos, memoria usada)
AtomicInteger usuariosActivos = new AtomicInteger(0);
Gauge.builder("usuarios.activos", usuariosActivos,
        AtomicInteger::get)
        .description("Usuarios activos en este momento")
        .register(registry);

// Timer: mide duracion y cuenta invocaciones (ej: latencia de endpoint)
@RestController
public class PedidoController {
    private final Timer buscarPedidosTimer;

    public PedidoController(MeterRegistry registry) {
        this.buscarPedidosTimer = Timer.builder("api.pedidos.buscar")
                .description("Latencia de busqueda de pedidos")
                .publishPercentiles(0.5, 0.95, 0.99)
                .register(registry);
    }

    @GetMapping("/api/pedidos/{id}")
    public PedidoResponse buscarPedido(@PathVariable Long id) {
        return buscarPedidosTimer.record(() -> {
            // Operacion medida
            return pedidoService.buscarPorId(new PedidoId(id))
                    .map(PedidoMapper::toResponse)
                    .orElseThrow(() -> new ResponseStatusException(
                            HttpStatus.NOT_FOUND));
        });
    }
}

// DistributionSummary: mide distribucion de eventos (ej: tamano de payloads)
DistributionSummary.builder("api.payload.size")
        .description("Tamano de payloads HTTP")
        .baseUnit("bytes")
        .register(registry);
```

**Metricas personalizadas para un endpoint REST:**

```java
@RestController
@RequestMapping("/api/pedidos")
public class PedidoController {

    private final Counter pedidosCreados;
    private final Counter pedidosConError;
    private final Timer tiempoCreacion;

    @PostMapping
    public ResponseEntity<PedidoResponse> crear(
            @Valid @RequestBody CrearPedidoRequest request) {
        return tiempoCreacion.record(() -> {
            try {
                Pedido pedido = pedidoService.crear(request);
                pedidosCreados.increment();
                return ResponseEntity.status(201)
                        .body(PedidoMapper.toResponse(pedido));
            } catch (Exception e) {
                pedidosConError.increment();
                throw e;
            }
        });
    }
}
```

### 10.5.3 Tracing: OpenTelemetry

En un sistema distribuido, una peticion HTTP puede atravesar 5 microservicios. Sin tracing, es imposible saber donde esta el cuello de botella.

```java
// Con Spring Boot + Micrometer Tracing + OpenTelemetry

// En application.yml:
// management.tracing.sampling.probability=1.0

// El traceId y spanId se propagan automaticamente
// entre servicios via headers HTTP W3C TraceContext.

// Ejemplo de log con MDC que incluye traceId:
@RestController
public class PedidoController {
    private static final Logger log =
            LoggerFactory.getLogger(PedidoController.class);

    @PostMapping("/api/pedidos")
    public PedidoResponse crear(@RequestBody CrearPedidoRequest r) {
        // log incluye automaticamente traceId si esta configurado
        log.info("Creando pedido para cliente {}", r.clienteId());

        // Llamada a otro microservicio: el trace se propaga
        ProductoResponse producto =
                productoClient.obtenerProducto(r.productoId());

        log.info("Producto obtenido: {}", producto.nombre());
        return pedidoService.crear(r, producto);
    }
}
```

**Propagacion manual de contexto de tracing cuando no hay headers HTTP:**

```java
// Si necesitas pasar el traceId a un hilo nuevo o a una cola de mensajes
@Async
public CompletableFuture<Void> procesarAsync(Pedido pedido) {
    // Spring propaga el contexto de tracing automaticamente con @Async
    return CompletableFuture.runAsync(() -> {
        log.info("Procesando pedido {} asincronamente",
                pedido.getId());
    });
}
```

### 10.5.4 Health Checks: Spring Boot Actuator + Kubernetes

En Kubernetes, la diferencia entre listo y vivo es critica.

```java
// Liveness: "Estoy vivo?" Si falla, K8s reinicia el pod
@Component
public class LivenessHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        // Liveness debe ser ligero: solo verificar que la app no esta muerta
        if (Runtime.getRuntime().freeMemory() < 10 * 1024 * 1024) {
            return Health.down()
                    .withDetail("error", "Memoria criticamente baja")
                    .build();
        }
        return Health.up().build();
    }
}

// Readiness: "Puedo recibir trafico?" Si falla, K8s deja de enviar peticiones
@Component
public class ReadinessHealthIndicator implements HealthIndicator {
    private final DataSource dataSource;
    private final KafkaAdmin kafkaAdmin;

    @Override
    public Health health() {
        // Verificar componentes externos necesarios para operar
        try (Connection conn = dataSource.getConnection()) {
            if (!conn.isValid(5)) {
                return Health.down()
                        .withDetail("database", "Conexion invalida")
                        .build();
            }
        } catch (Exception e) {
            return Health.down()
                    .withDetail("database", e.getMessage())
                    .build();
        }
        return Health.up().build();
    }
}
```

**Configuracion en application.yml:**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true
      show-details: always
      group:
        readiness:
          include: readinessState,db,kafka
        liveness:
          include: livenessState,ping
```

**Kubernetes deployment usa estos endpoints:**

```yaml
# fragmento de Kubernetes Deployment
spec:
  containers:
    - name: mi-app
      image: mi-app:latest
      livenessProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        initialDelaySeconds: 30
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        initialDelaySeconds: 15
        periodSeconds: 5
```

---

## 10.6 Rendimiento y Optimizacion JVM

El rendimiento no es magia. Es ciencia. La JVM ofrece herramientas excepcionales para entender y optimizar el comportamiento de tu aplicacion en tiempo real.

### 10.6.1 Perfiles de Rendimiento

- **CPU profiling:** que metodos consumen mas tiempo de CPU?
- **Memory profiling:** donde se asignan mas objetos?
- **Allocation profiling:** que tipo de objetos se crean mas frecuentemente?
- **Lock profiling:** donde hay contencion de hilos?

### 10.6.2 Java Flight Recorder (JFR)

JFR registra eventos de la JVM con un overhead menor al 1%. Puede usarse en produccion.

```bash
# Habilitar JFR al arrancar
java -XX:StartFlightRecording:filename=recording.jfr,duration=60s \
     -jar mi-aplicacion.jar

# O conectarse a un proceso en ejecucion
jcmd <pid> JFR.start name=profile duration=120s \
     filename=/tmp/profile.jfr
```

Desde Java 11 (Oracle JDK) o cualquier JDK 11+ (OpenJDK desde JDK 8u262), JFR esta disponible sin licencia. Para ver los resultados, abre el archivo `.jfr` con JDK Mission Control (JMC) o `jfr` en IntelliJ.

```java
// Eventos personalizados en JFR
import jdk.jfr.*;

@Name("com.miapp.PedidoCreado")
@Label("Pedido Creado")
@Description("Evento emitido cuando se crea un pedido")
public class PedidoCreadoEvent extends Event {

    @Label("ID del Pedido")
    private long pedidoId;

    @Label("Monto Total")
    private double montoTotal;

    public PedidoCreadoEvent(long pedidoId, double montoTotal) {
        this.pedidoId = pedidoId;
        this.montoTotal = montoTotal;
    }
}

// En el servicio
public void crearPedido(Pedido pedido) {
    PedidoCreadoEvent event = new PedidoCreadoEvent(
            pedido.getId(), pedido.getTotal().doubleValue());
    event.commit(); // Registra el evento en JFR
}
```

### 10.6.3 Flags de JVM Importantes

```bash
java \
  -Xms512m \                          # Heap inicial
  -Xmx2g \                            # Heap maximo
  -XX:MaxMetaspaceSize=256m \         # Metadatos de clases
  -XX:+UseG1GC \                      # G1 Garbage Collector (default desde Java 9)
  -XX:MaxGCPauseMillis=200 \          # Objetivo de pausa para G1
  -XX:+PrintGCDetails \               # Log de GC
  -XX:+HeapDumpOnOutOfMemoryError \   # Heap dump en OOM
  -XX:HeapDumpPath=/var/log/app \     # Directorio del heap dump
  -XX:ErrorFile=/var/log/app/hs_err_pid%p.log \ # Crash logs
  -jar mi-aplicacion.jar
```

**Shenandoah GC** (bajas pausas, OpenJDK): `-XX:+UseShenandoahGC`

**ZGC** (pausas sub-milisegundo, desde Java 15): `-XX:+UseZGC`

### 10.6.4 Optimizaciones Comunes

```java
// MAL: Creacion excesiva de objetos en bucle caliente
for (int i = 0; i < 1000000; i++) {
    String key = "user:" + i;          // String nuevo cada iteracion
    BigDecimal total = BigDecimal.ZERO.add( // BigDecimal inmutable
            new BigDecimal(amount));   // Nuevo BigDecimal cada vez
    cache.put(key, total);
}

// MEJOR: Reutilizar y preferir primitivas
StringBuilder prefix = new StringBuilder("user:");
for (int i = 0; i < 1000000; i++) {
    prefix.setLength(5);               // Reutilizar StringBuilder
    String key = prefix.append(i).toString();
    // Para calculos financieros, usar BigDecimal.
    // Para calculos no financieros, long o double son 100x mas rapidos.
    cache.put(key, (long)(amount * 100)); // Usar long internamente
}

// MAL: Autoboxing innecesario
List<Integer> numbers = new ArrayList<>();
for (int i = 0; i < 1000000; i++) {
    numbers.add(i); // Autoboxing: int -> Integer
}
int sum = 0;
for (Integer n : numbers) {
    sum += n; // Unboxing: Integer -> int
}

// MEJOR: Usar primitivas directamente
int[] numbers = new int[1000000];
for (int i = 0; i < 1000000; i++) {
    numbers[i] = i;
}
int sum = 0;
for (int n : numbers) sum += n;
```

### 10.6.5 JMH (Java Microbenchmark Harness) en Profundidad

Escribir benchmarks correctos es sorprendentemente dificil. La JVM aplica optimizaciones (JIT compilation, dead code elimination, constant folding) que pueden hacer que tu benchmark no mida nada util.

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Benchmark)
@Fork(value = 1, warmups = 1)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
public class ConcatenacionBenchmark {

    @Param({"10", "100", "1000"})
    private int length;

    private String base;

    @Setup
    public void setup() {
        base = "a".repeat(length);
    }

    @Benchmark
    public String concatenacionConMas() {
        return base + "b" + "c" + "d"; // Crea multiples strings intermedios
    }

    @Benchmark
    public String stringBuilder() {
        return new StringBuilder(base)
                .append("b").append("c").append("d").toString();
    }

    // Blackhole: evita que la JVM elimine codigo "muerto"
    @Benchmark
    public void evitarDeadCodeElimination(Blackhole bh) {
        String result = concatenacionConMas();
        bh.consume(result); // JVM no puede eliminar esto
    }
}

// Ejecutar: java -jar benchmarks.jar
```

**Reglas de oro para JMH:**
- Usa `Blackhole` para evitar dead code elimination.
- No midas en la primera iteracion (warmup necesario).
- Usa `@Fork` para aislar benchmarks (JVM independiente).
- Evita benchmarks que midan operaciones demasiado rapidas (menos de 1us).
- Los resultados de JMH son indicativos, no absolutos. Prueba en tu entorno objetivo.

### 10.6.6 GraalVM Native Image

Compila tu aplicacion Java a un binario nativo. Ventajas: arranque instantaneo (ms en vez de segundos) y menor consumo de memoria.

```bash
# Instalar GraalVM y native-image
gu install native-image

# Compilar a binario nativo
native-image -jar mi-aplicacion.jar --no-fallback -H:Name=miapp

# Ejecutar el binario nativo
./miapp
```

**Con Spring Boot:**

```xml
<!-- plugin en pom.xml -->
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
    <version>0.10.2</version>
</plugin>
```

```bash
mvn -Pnative native:compile
# Genera el binario nativo en target/
```

**Limitaciones de Native Image:**
- La reflection, el dynamic class loading, y los proxies dinamicos deben declararse en archivos de configuracion.
- No todos los frameworks son compatibles (Spring Boot tiene soporte casi completo desde 3.0, Quarkus es nativo-first).
- Tiempo de compilacion mucho mayor (minutos en lugar de segundos).
- Ideal para serverless, microservicios con arranque rapido, CLI tools.

```java
// Configuracion de reflection para Native Image
// src/main/resources/META-INF/native-image/reflect-config.json
{
  "name": "com.miapp.modelo.Usuario",
  "allDeclaredConstructors": true,
  "allPublicConstructors": true,
  "allDeclaredMethods": true,
  "allPublicMethods": true
}
```

**Cuando usar GraalVM Native Image:** serverless (AWS Lambda, Google Cloud Run), microservicios que escalan desde cero, CLI tools, entornos con recursos limitados.

**Cuando NO usarlo:** aplicaciones que dependen fuertemente de reflection en runtime, aplicaciones con plugins dinamicos, cuando el tiempo de compilacion extra no es aceptable.

## 10.7 El Ecosistema Java Moderno

Java no es solo un lenguaje. Es un ecosistema vibrante de frameworks, herramientas, plataformas y comunidades.

### 10.7.1 Spring Boot — El Framework Dominante

Spring Boot simplifica la creacion de aplicaciones Spring con configuracion automatica, servidor embebido, y dependencias starter.

```java
@SpringBootApplication
public class MiAplicacion {
    public static void main(String[] args) {
        SpringApplication.run(MiAplicacion.class, args);
    }
}

// @RestController = @Controller + @ResponseBody
@RestController
@RequestMapping("/api/productos")
public class ProductoController {

    private final ProductoService service;

    // Inyeccion por constructor (sin @Autowired, Spring lo infiere)
    public ProductoController(ProductoService service) {
        this.service = service;
    }

    @GetMapping
    public List<ProductoResponse> listar(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return service.listar(PageRequest.of(page, size));
    }

    @GetMapping("/{id}")
    public ProductoResponse obtener(@PathVariable Long id) {
        return service.buscarPorId(id)
                .orElseThrow(() -> new ResponseStatusException(
                        HttpStatus.NOT_FOUND));
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ProductoResponse crear(
            @Valid @RequestBody CrearProductoRequest request) {
        return service.crear(request);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(
            MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
                .forEach(e -> errors.put(e.getField(),
                        e.getDefaultMessage()));
        return ResponseEntity.badRequest().body(errors);
    }
}

// @Service: logica de negocio
@Service
@Transactional(readOnly = true)
public class ProductoService {

    private final ProductoRepository repository;
    private final ProductoMapper mapper;

    public ProductoService(ProductoRepository repository,
                            ProductoMapper mapper) {
        this.repository = repository;
        this.mapper = mapper;
    }

    public List<ProductoResponse> listar(Pageable pageable) {
        return repository.findAll(pageable)
                .map(mapper::toResponse).toList();
    }

    @Transactional
    public ProductoResponse crear(CrearProductoRequest request) {
        if (repository.existsBySku(request.sku())) {
            throw new IllegalArgumentException("SKU duplicado");
        }
        Producto entity = mapper.toEntity(request);
        return mapper.toResponse(repository.save(entity));
    }
}

// @Repository: acceso a datos
@Repository
public interface ProductoRepository
        extends JpaRepository<Producto, Long> {
    boolean existsBySku(String sku);
    List<Producto> findByCategoria(String categoria);
}
```

**Perfiles de Spring:** `@Profile("dev")`, `@Profile("prod")` permiten tener configuraciones diferentes segun el entorno.

```yaml
# application.yml
spring:
  profiles:
    active: dev
---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:testdb
---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://db-prod:5432/miapp
    hikari:
      maximum-pool-size: 20
```

### 10.7.2 Alternativas Ligeras: Quarkus y Micronaut

**Quarkus:** optimizado para GraalVM y HotSpot. Arranque en milisegundos, disenado para contenedores y serverless. Su paradigma es "compile-time boot": hace en compilacion lo que Spring hace en runtime.

```java
// Quarkus REST endpoint
@Path("/api/productos")
public class ProductoResource {

    @Inject
    ProductoService service;

    @GET
    public List<Producto> listar() {
        return service.listar();
    }

    @POST
    @Transactional
    public Response crear(Producto producto) {
        service.crear(producto);
        return Response.status(201).build();
    }
}
```

**Micronaut:** similar a Quarkus, con compilacion ahead-of-time (AOT), ideal para microservicios y funciones serverless. Su inyeccion de dependencias se resuelve en compilacion, eliminando el overhead de reflection.

**Cuando elegir cada uno:**

| | Spring Boot | Quarkus | Micronaut |
|---|---|---|---|
| Madurez | Maxima | Alta | Creciente |
| Arranque | ~2-5s | ~0.5-1s | ~0.5-1s |
| Memoria | ~150-300MB | ~30-80MB | ~30-80MB |
| Ecosistema | Enorme | Creciente | Mediano |
| Native Image | Soportado | Nativo-first | Nativo-first |
| Curva aprendizaje | Media | Media-Alta | Media |
| Ideal para | Empresarial | Cloud-native | Microservicios |

### 10.7.3 Build Tools: Maven vs Gradle

**Maven:** XML declarativo, opinionado, estructura de directorios estandar, ciclo de vida fijo. Ideal para proyectos donde la estandarizacion es mas importante que la flexibilidad.

```xml
<!-- pom.xml -->
<project>
    <groupId>com.miempresa</groupId>
    <artifactId>miapp</artifactId>
    <version>1.0.0</version>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
</project>
```

**Gradle:** Groovy o Kotlin DSL, cache incremental, mas rapido en builds incrementales. Ideal para proyectos con necesidades de construccion complejas y Android.

```groovy
// build.gradle.kts (Kotlin DSL)
plugins {
    id("org.springframework.boot") version "3.3.0"
    id("io.spring.dependency-management") version "1.1.5"
    java
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**Comandos esenciales:**

```bash
# Maven
mvn clean compile      # Limpia y compila
mvn test               # Ejecuta tests
mvn package            # Genera JAR/WAR

# Gradle
gradle build           # Compila, testea y empaqueta
gradle test            # Solo tests
gradle clean           # Limpia artefactos
```

**Cuando elegir:** Maven para equipos grandes donde la estandarizacion es clave. Gradle cuando necesitas builds rapidos, personalizacion, o desarrollas Android.

### 10.7.4 Contenedores: Dockerfile para Java

```dockerfile
# Multi-stage build: compilacion y ejecucion separadas
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN ./mvnw dependency:resolve
COPY src ./src
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup --system javauser && adduser -S -G javauser javauser
USER javauser
COPY --from=builder /app/target/*.jar app.jar

# Flags JVM optimizadas para contenedores
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-jar", "app.jar"]
```

### 10.7.5 Kubernetes: Despliegue de Microservicios

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pedidos-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pedidos-service
  template:
    metadata:
      labels:
        app: pedidos-service
    spec:
      containers:
        - name: app
          image: mi-registry/pedidos-service:1.2.0
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: password
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: pedidos-service
spec:
  selector:
    app: pedidos-service
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: pedidos-config
data:
  database-url: "jdbc:postgresql://postgres:5432/pedidos"
  kafka-brokers: "kafka:9092"
```

### 10.7.6 CI/CD: GitHub Actions para Proyecto Java

```yaml
name: Java CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Configurar JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'

      - name: Compilar y testear
        run: mvn --batch-mode verify

      - name: Analisis de calidad (SonarQube)
        run: mvn sonar:sonar
          -Dsonar.projectKey=miapp
          -Dsonar.host.url=${{ secrets.SONAR_URL }}
          -Dsonar.login=${{ secrets.SONAR_TOKEN }}

      - name: Construir imagen Docker
        run: docker build -t mi-registry/miapp:latest .

      - name: Publicar imagen
        if: github.ref == 'refs/heads/main'
        run: |
          docker tag mi-registry/miapp:latest \
            mi-registry/miapp:${{ github.sha }}
          docker push mi-registry/miapp:${{ github.sha }}

      - name: Desplegar en Kubernetes
        if: github.ref == 'refs/heads/main'
        run: |
          kubectl set image deployment/miapp \
            miapp=mi-registry/miapp:${{ github.sha }}
          kubectl rollout status deployment/miapp
```

**Pipeline tipico:**

```
[Push a repo]
      |
      v
[1. Checkout + Compilacion] --> [2. Analisis estatico (Checkstyle, PMD)]
      |
      v
[3. Tests unitarios] --> [4. Analisis de calidad (SonarQube)]
      |
      v
[5. Empaquetado] --> [6. Tests de integracion]
      |
      v
[7. Construir imagen Docker] --> [8. Publicar en registry]
                                     |
                                     v
                              [9. Desplegar en K8s]
```

---

## 10.8 Convenciones de Codigo y Profesionalismo

Un codigo consistente reduce la friccion cognitiva y facilita las revisiones. Estas no son reglas arbitrarias: son patrones que la comunidad Java ha refinado durante decadas.

### 10.8.1 Nomenclatura

| Elemento | Convencion | Ejemplo |
|----------|------------|---------|
| Clases / Interfaces | `PascalCase` | `CalculadoraImpuestos`, `ServicioNotificacion` |
| Metodos / Variables | `camelCase` | `calcularTotal()`, `numeroEmpleados` |
| Constantes (`static final`) | `UPPER_SNAKE_CASE` | `MAX_INTENTOS`, `TASA_IMPUESTO_DEFAULT` |
| Paquetes | minusculas, dominio invertido | `com.empresa.modulo.util` |
| Enums | `PascalCase` tipo, `UPPER_SNAKE_CASE` constantes | `enum DiaSemana { LUNES, MARTES }` |
| Genericos | Una letra mayuscula significativa | `<T>`, `<K, V>`, `<E extends Exception>` |

### 10.8.2 Buenas Practicas Defensivas

**No retornes `null`:** usa `Collections.emptyList()` y `Optional<T>`.

```java
// MAL
public List<Usuario> buscarUsuarios() {
    if (noHayResultados) return null; // PELIGRO: NullPointerException
    return resultados;
}

// BIEN
public List<Usuario> buscarUsuarios() {
    if (noHayResultados) return Collections.emptyList();
    return resultados;
}

public Optional<Usuario> buscarPorId(long id) {
    return Optional.ofNullable(repositorio.get(id));
}
```

**Evita numeros magicos:**

```java
// MAL
if (empleado.getEdad() > 65) { /* ... */ }
Thread.sleep(3000);

// BIEN
private static final int EDAD_JUBILACION = 65;
private static final int TIMEOUT_CONEXION_MS = 3000;
if (empleado.getEdad() > EDAD_JUBILACION) { /* ... */ }
Thread.sleep(TIMEOUT_CONEXION_MS);
```

**Favorece la inmutabilidad** con `final`, sin setters:

```java
public final class Direccion {
    private final String calle;
    private final String ciudad;

    public Direccion(String calle, String ciudad) {
        this.calle = calle;
        this.ciudad = ciudad;
    }
    public String getCalle() { return calle; }
    public String getCiudad() { return ciudad; }
}
```

**Composicion sobre herencia:**

```java
// Herencia (rigida, alto acoplamiento)
public class Pila extends ArrayList<String> {
    public void push(String item) { add(item); }
    public String pop() { return remove(size() - 1); }
    // Problema: se heredan add(), remove(), clear(), get()...
    // Se puede romper: pila.add(0, "colado");
}

// Composicion (flexible, bajo acoplamiento) - PREFERIDO
public class Pila<E> {
    private final List<E> elementos = new ArrayList<>();

    public void push(E item) { elementos.add(item); }
    public E pop() {
        if (elementos.isEmpty()) throw new EmptyStackException();
        return elementos.remove(elementos.size() - 1);
    }
    public boolean estaVacia() { return elementos.isEmpty(); }
    // Solo exponemos operaciones de pila, no de lista
}
```

**Javadoc para APIs publicas:**

```java
/**
 * Calcula el coste total de envio basado en el peso y la distancia.
 * <p>
 * La formula utilizada es:
 * {@code peso * tarifaPorKg + distancia * tarifaPorKm}.
 * Se aplica un recargo del 20% para envios internacionales.
 *
 * @param paquete el paquete a enviar (no puede ser {@code null})
 * @param direccionOrigen  direccion de recogida
 * @param direccionDestino direccion de entrega
 * @return el coste total en la moneda local
 * @throws IllegalArgumentException si el paquete excede el peso maximo
 * @see EstrategiaEnvio
 */
public double calcularCosteEnvio(Paquete paquete,
        Direccion direccionOrigen, Direccion direccionDestino) {
    // ...
}
```

---

## 10.9 El Camino del Programador Java: El Siguiente Nivel

Has llegado al final de este libro. Pero en programacion, el final de un libro es solo el principio del camino. Esto es lo que te espera.

### Que Aprender Despues

**Si quieres ser desarrollador backend:**
- **Spring Boot en profundidad:** Spring Security, Spring Data JPA, Spring Cloud, Spring Batch.
- **Bases de datos:** PostgreSQL avanzado (indices, CTEs, window functions), MongoDB, Redis.
- **Mensajeria:** Apache Kafka, RabbitMQ. Event sourcing y CQRS avanzado.

**Si quieres ser arquitecto de software:**
- **Patrones de microservicios:** Saga, Circuit Breaker (Resilience4j), API Gateway, Service Mesh.
- **Domain-Driven Design avanzado:** Event Storming, Context Mapping, Anti-Corruption Layer.
- **Cloud:** AWS (EC2, RDS, SQS, Lambda) o GCP o Azure. Infraestructura como codigo (Terraform).

**Si quieres especializarte en rendimiento:**
- **JVM interna:** JIT compilation, GC algorithms (G1, ZGC, Shenandoah), class loading.
- **Profiling:** async-profiler, JFR avanzado, JMC.
- **Optimizacion:** lock-free data structures, off-heap memory, vectored API (Project Panama).

### Certificaciones

- **Oracle Certified Professional (OCP): Java SE 21 Developer.** Valida tus conocimientos a nivel profesional. Requiere aprobar el examen 1Z0-830.
- **Spring Professional.** Certificacion oficial de VMware para Spring.

### Comunidad

- **Conferencias:** Devoxx (Bruselas, Londres, Paris), Spring I/O (Barcelona), JVM Language Summit, QCon.
- **Blogs y newsletters:** Baeldung, Vlad Mihalcea (Hibernate), Martin Fowler, InfoQ, Java Weekly.
- **Open source:** contribuye a proyectos que uses. Empieza con documentacion, luego bugs simples. Spring, Hibernate, JUnit aceptan contribuciones.
- **Comunidades locales:** Java User Groups (JUGs) en casi todas las ciudades del mundo.

### Mentalidad de Crecimiento

La tecnologia cambia. Los frameworks van y vienen. Lo que permanece son los fundamentos que has adquirido en este libro: orientacion a objetos, estructuras de datos, concurrencia, diseno de sistemas, testing, patrones.

Tres habitos que te distinguiran como profesional:

1. **Lee codigo de otros.** Clona repositorios open source y lee su codigo fuente. Spring Framework, Hibernate, Guava, JUnit. Entender como estan construidas las herramientas que usas es la forma mas rapida de crecer.

2. **Contribuye a open source.** No necesitas ser un experto. Corregir un typo en la documentacion, anadir un test, reportar un bug con un caso reproducible. Cada contribucion te ensena el proceso de desarrollo profesional.

3. **Ensena lo que sabes.** Escribe un blog, da una charla en tu JUG local, ayuda a un companero junior. Ensenar es la forma mas efectiva de consolidar el conocimiento. No necesitas ser el maximo experto: solo necesitas saber algo que alguien mas no sabe.

---

## 10.10 Resumen Final del Capitulo

| Tema | Idea central |
|------|-------------|
| **SOLID** | Cinco principios que reducen acoplamiento y aumentan cohesion |
| **Patrones de Diseno** | Soluciones probadas: creacionales, estructurales, comportamiento, MVC, Repository, CQRS |
| **Arquitectura** | Capas, Hexagonal, DDD, Event-Driven, Clean Architecture |
| **Testing** | Piramide, TDD, Mockito avanzado, Property-Based, Mutation, Contract |
| **Observabilidad** | Logs estructurados (JSON), metricas (Micrometer), tracing (OpenTelemetry), health checks |
| **Rendimiento JVM** | JFR, JMH, flags GC, optimizaciones, GraalVM Native Image |
| **Ecosistema** | Spring Boot, Quarkus, Maven/Gradle, Docker, Kubernetes, CI/CD |
| **Profesionalismo** | Convenciones, inmutabilidad, composicion, Javadoc, mentalidad de crecimiento |

---

> *"El codigo limpio no es un conjunto de reglas rigidas, sino un compromiso personal y profesional con la excelencia."* — Robert C. Martin

> *"Primero aprende a escribir codigo. Luego aprende a escribirlo bien. Finalmente, aprende a no escribirlo — a ensamblarlo a partir de componentes que otros han escrito bien."* — Anonimo

---

Este capitulo cierra el libro, pero abre el camino hacia una carrera de crecimiento continuo. Los principios, patrones, arquitecturas y herramientas aqui descritos no son un fin en si mismos: son instrumentos. Usalos con criterio, adaptalos a tu contexto y, sobre todo, sigue aprendiendo.

Has recorrido un viaje desde el primer "Hola Mundo" hasta la arquitectura hexagonal, desde variables primitivas hasta sistemas distribuidos con Kafka. Lo que has construido en estas paginas no es solo conocimiento tecnico: es una base solida sobre la que edificaras el resto de tu carrera.

La industria del software evoluciona constantemente. Nuevos frameworks aparecen cada ano. Los paradigmas cambian. Pero los fundamentos que has aprendido — pensar en objetos, disenar con principios, probar con disciplina, medir con ciencia — esos te acompanaran siempre.

No dejes de programar. No dejes de aprender. No dejes de compartir.

El mundo necesita software bien construido. Y ahora tu sabes como hacerlo.

**Bienvenido a la profesion.**
