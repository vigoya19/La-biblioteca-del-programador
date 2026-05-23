# Capítulo 10: Seguridad y Control de Accesos

> "La seguridad de tu base de datos no es una armadura que se coloca al final del proyecto; es el cimiento invisible sobre el cual descansa toda la integridad de la infraestructura corporativa."

A medida que las bases de datos NoSQL se convirtieron en el motor estándar para almacenar información sensible de millones de usuarios globales en entornos Cloud, se transformaron también en el principal objetivo de ciberataques. La flexibilidad de esquemas dinámicos y la facilidad de conexión no deben malinterpretarse como una invitación a la laxitud en la seguridad. 

Una base de datos NoSQL expuesta a internet sin autenticación, con credenciales predeterminadas o vulnerable a manipulaciones lógicas de consulta representa una catástrofe legal y financiera. En este capítulo, estudiaremos el control de accesos basados en roles (**RBAC**), la encriptación de extremo a extremo, el diseño granular de políticas **IAM** en AWS para DynamoDB y cómo blindar nuestro código Node.js contra la temida **Inyección NoSQL**.

---

## 10.1 Autenticación y RBAC (Role-Based Access Control) en MongoDB

Para garantizar la seguridad en un cluster corporativo de MongoDB, el primer paso es deshabilitar el acceso libre sin credenciales y activar la autenticación obligatoria. Una vez activa, se aplica el principio de **Control de Accesos Basado en Roles (RBAC)**:

* **Privilegios**: Un privilegio consiste en una acción explícita (como `find`, `insert`, `update`, `dropCollection`) que se puede ejecutar sobre un recurso físico (una base de datos o una colección específica).
* **Roles**: En lugar de asignar privilegios caóticos de forma individual a cada desarrollador o microservicio, agrupamos privilegios comunes en **Roles**.
* **Roles Predefinidos**: MongoDB provee roles nativos listos para producción:
  * `readWrite`: Permite lecturas y escrituras ordinarias en una base de datos específica. Ideal para microservicios de backend.
  * `dbAdmin`: Permite crear índices, perfilar consultas y gestionar colecciones, pero no leer los datos de negocio sensibles.
  * `root`: Acceso absoluto y total al servidor de base de datos. Reservado únicamente para DBAs jefes de infraestructura.

---

## 10.2 Encriptación Física: En Tránsito y en Reposo

Para cumplir con normativas de protección de datos internacionales (como GDPR o HIPAA), la información confidencial debe estar cifrada en todo momento:

### 1. Encriptación en Tránsito (TLS/SSL)
Garantiza que la conexión por red entre tu servidor backend y los nodos de tu base de datos NoSQL esté cifrada mediante certificados SSL/TLS validados. Esto evita ataques de tipo **Man-in-the-Middle (MitM)**, donde un atacante intercepta físicamente los paquetes de datos que viajan por cable de red o Wi-Fi para extraer contraseñas o datos de negocio en texto plano.

### 2. Encriptación en Reposo (AES-256)
Cifra físicamente los archivos binarios que el motor de base de datos escribe en el disco duro físico (SSD/HDD) utilizando algoritmos avanzados de cifrado simétrico como **AES-256**. 
* **Protección**: Si un atacante vulnera físicamente tu servidor en el centro de datos y roba los discos duros físicos, o descarga una copia del archivo binario `dump.rdb` o las SSTables de Cassandra, los datos le resultarán completamente inútiles y ruido ilegible al no contar con la clave de descifrado física protegida por hardware de seguridad (KMS / HSM).

---

## 10.3 Políticas IAM en AWS para DynamoDB

En la nube de AWS, el acceso a Amazon DynamoDB se gestiona de forma centralizada y robusta mediante políticas **IAM (Identity and Access Management)**. Siguiendo el principio de **mínimo privilegio**, nunca debes utilizar credenciales raíz de AWS ni conceder accesos comodín (`"Action": "dynamodb:*"`).

### Ejemplo de Política IAM Granular de Mínimo Privilegio (JSON):
Esta política permite únicamente lecturas y escrituras ordinarias sobre una sola tabla específica, prohibiendo tajantemente destruir la tabla (`DeleteTable`) o alterar su infraestructura:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AccesoLecturaEscrituraGranularTabla",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/TablaProductos"
    }
  ]
}
```

---

## 10.4 Prevención de Inyección NoSQL: El Ataque Silencioso

A diferencia de las inyecciones SQL tradicionales (donde un atacante inyecta código de texto plano como `' OR 1=1 --`), en bases de datos documentales como MongoDB el peligro reside en la **Inyección de Objetos NoSQL**. 

Ocurre cuando tu backend en Node.js recibe un cuerpo de petición JSON del cliente y lo inyecta de forma directa dentro de una consulta de búsqueda de Mongoose sin sanitizar:

```typescript
// ❌ CÓDIGO VULNERABLE: Inyección de Objetos
app.post('/api/login', async (req, res) => {
  // Si req.body.password es {"$ne": ""}, la consulta buscará un usuario cuyo email coincida y su contraseña NO sea vacía.
  const usuario = await Usuario.findOne({
    email: req.body.email,
    password: req.body.password // Si el atacante manda un objeto inyectado, se salta la autenticación
  });
  
  if (usuario) res.send("Acceso Autorizado");
});
```

* **El Ataque**: Si un hacker envía un JSON malicioso:
  ```json
  {
    "email": "admin@empresa.com",
    "password": { "$ne": "cualquier_cosa" }
  }
  ```
  La consulta final en la base de datos se evaluará como: *"Buscar usuario con email 'admin@empresa.com' y cuya contraseña NO sea 'cualquier_cosa'"*. Al cumplirse la condición lógicamente, el servidor autorizará el login del administrador al atacante **sin que este conozca la contraseña física**.

> [!NOTE]
> ### 🛡️ El Guardaespalda con Lista de Invitados y Lentes Infrarrojos
> 
> Visualicemos la seguridad en tu base de datos como un club nocturno VIP de alta seguridad:
> 
> - **RBAC (Roles)** es equivalente a un **Guardaespalda en la Entrada Principal con Lista de Invitados**:
>   - Llega un empleado con uniforme de "Bartender" (el Rol `readWrite`). El guardaespaldas revisa su acreditación y lo deja pasar a la zona de la barra de tragos.
>   - Si de pronto el Bartender intenta caminar y entrar en la "Bóveda VIP de la Oficina Principal" (los privilegios de `dbAdmin` o `root`), el guardaespaldas lo detiene físicamente y lo expulsa del club de inmediato.
> - **Inyección de Objetos NoSQL** es equivalente a un **Espía Astuto disfrazado de Proveedor**:
>   - El espía le entrega una orden de pedido escrita en un papel al Bartender que dice: *"Entregar 1 Caja de Limones al camarógrafo cuyo ID de empleado sea DIFERENTE de vacío"*.
>   - Si el Bartender es ingenuo y lee la orden textualmente sin analizar si tiene sentido lógico, caminará a los almacenes y repartirá limones de forma indiscriminada a todos los empleados del club, violando la privacidad de la empresa.
>   - **La Sanitización (Middleware)** es equivalente a un **Inspector de Recepción con Lentes Infrarrojos**: El inspector intercepta todas las notas de pedido antes de que lleguen a manos del Bartender. Si detecta palabras clave sospechosas o modificadores lógicos escritos en bolígrafo rojo (como `$ne` o `$gt`), rompe el papel y arresta al espía al instante.

---

## 10.5 Implementación en TypeScript de Middleware de Sanitización

Implementemos un middleware Express en TypeScript para sanitizar de forma automática todas las peticiones entrantes contra inyecciones de objetos NoSQL, despojando cualquier clave que empiece con el carácter reservado `$`:

#### [middlewareSanitizar.ts](file:///Users/andres/Documents/biblioteca/nosql-databases-programming-book/src/middlewares/middlewareSanitizar.ts)
```typescript
import { Request, Response, NextFunction } from 'express';

// 1. Función recursiva profunda para sanitizar objetos y eliminar modificadores '$'
export function sanitizarObjeto(objeto: any): any {
  if (objeto instanceof Object) {
    for (const clave in objeto) {
      if (clave.startsWith('$')) {
        // Borramos físicamente la clave del objeto para neutralizar operadores maliciosos
        delete objeto[clave];
      } else {
        // Llamada recursiva profunda para analizar objetos anidados
        sanitizarObjeto(objeto[clave]);
      }
    }
  }
  return objeto;
}

// 2. Middleware de Express robusto e integrable
export function middlewarePrevenirInyeccionNoSQL(req: Request, res: Response, next: NextFunction): void {
  if (req.body) {
    req.body = sanitizarObjeto(req.body);
  }
  if (req.query) {
    req.query = sanitizarObjeto(req.query);
  }
  if (req.params) {
    req.params = sanitizarObjeto(req.params);
  }
  
  next();
}
```

---

  next();
}
```

---

## 10.6 Deep Dive: Client-Side Field Level Encryption (CSFLE)

Cuando diseñas arquitecturas bancarias, de salud o de defensa nacional, cifrar los datos en tránsito (TLS) y en reposo (AES-256) no es suficiente. Si un cibercriminal vulnera el sistema operativo de tu base de datos o si un administrador deshonesto (*DBA*) tiene acceso root a la consola de MongoDB, este podrá ejecutar búsquedas y ver de inmediato en memoria RAM toda la información confidencial en texto plano.

Para erradicar esta vulnerabilidad, implementamos **Cifrado a Nivel de Campo del Lado del Cliente (CSFLE - Client-Side Field Level Encryption)**:

> [!NOTE]
> ### ✉️ La Analogía de la Carta en Idioma Secreto y el Cartero Ciego
> 
> Visualicemos el cifrado del lado del cliente frente al cifrado de disco tradicional:
> 
> - **El Cifrado Tradicional en Reposo (AES-256)** es equivalente a **Guardar Cartas en una Caja Fuerte con Llave**:
>   - Escribes una carta de amor en texto plano y se la entregas a la secretaria (el Servidor de MongoDB). La secretaria la lee, sonríe, la mete en un sobre y la guarda con llave dentro de una caja fuerte de metal (el cifrado en disco).
>   - Si un espía roba la caja fuerte entera por la noche, no podrá leer la carta. Pero si el espía convence o hackea a la secretaria de día, ella abrirá la caja fuerte de buena gana, sacará la carta y el espía la leerá completa en texto plano.
> - **El Cifrado CSFLE** es equivalente a **Escribir la Carta en un Idioma Secreto Cifrado**:
>   - Antes de salir de tu casa, tomas un **Diccionario de Claves Matemáticas de Alta Seguridad (el KMS)** que solo tú y tu destinatario poseen.
>   - Traduces la carta a una sopa de caracteres aleatorios incomprensibles: `"$2x#a%9o"`.
>   - Se la entregas a la secretaria. Ella recibe la carta cifrada, no entiende absolutamente nada de lo que dice, y la mete en la carpeta. La secretaria es un **Cartero Ciego**: transporta y clasifica la carta, pero jamás puede saber su contenido.
>   - Cuando tu destinatario (el microservicio autorizado) recibe la carta, saca su diccionario local (el KMS) y la descifra en la privacidad de su hogar. El servidor de base de datos es 100% inmune a espionajes internos.

### 1. El Flujo de Cifrado CSFLE en Producción
CSFLE cambia por completo el flujo de datos delegando las tareas de cifrado matemáticamente intensivas directamente a tu microservicio backend, utilizando un gestor de claves externo como **AWS KMS** o **HashiCorp Vault**:

```
 ┌──────────────────────┐                     ┌──────────────────────┐
 │  Backend Client RAM  │                     │  MongoDB Server RAM  │
 ├──────────────────────┤                     ├──────────────────────┤
 │ 1. Query Intercepted │                     │                      │
 │ 2. Request Data Key  │ ◄──► [ AWS KMS ]    │                      │
 │ 3. Encrypt Fields    │                     │                      │
 │ 4. Send Cipherbytes  │ ──────────────────► │ 5. Receive BinData   │
 └──────────────────────┘                     │    (Completely Blind)│
                                              └──────────────────────┘
```

1. **Intercepción**: El driver de MongoDB en tu backend intercepta la operación de guardado.
2. **Obtención de Llave**: Se conecta a través de red segura a un servicio de gestión de claves externo (KMS) y solicita la clave de cifrado del cliente de forma segura en memoria RAM.
3. **Cifrado local**: El driver encripta de forma local en la RAM del backend los campos estrictamente marcados como sensibles (como el número de la tarjeta de crédito o el DNI) antes de serializarlos.
4. **Envío**: Envía el documento a MongoDB por red. MongoDB recibe el campo en un tipo de dato binario cifrado (**`BinData` de tipo 6**).
5. **Base de Datos Ciega**: MongoDB indexa y almacena el documento en disco. El servidor de base de datos solo almacena ruido criptográfico; carece por completo de la capacidad matemática para descifrar el campo o ver el texto plano en su memoria RAM o logs de auditoría.
6. **Lectura Segura**: Al consultar, el driver de Node.js en el backend recibe el `BinData` cifrado de MongoDB, solicita el descifrado al KMS y le entrega el texto plano descifrado a tu aplicación de forma transparente.

---

## Resumen del Capítulo


* La seguridad robusta en NoSQL requiere activar la autenticación obligatoria y aplicar políticas de accesos basados en roles **RBAC** (`readWrite`, `dbAdmin`).
* La protección física de datos exige cifrar conexiones mediante protocolos de red **TLS/SSL** (en tránsito) y archivos de disco con **AES-256** (en reposo).
* En entornos Cloud, el uso de políticas **AWS IAM** granulares de mínimo privilegio previene fugas de información catastróficas.
* La inyección de objetos NoSQL aprovecha la falta de sanitización de request bodies JSON para inyectar operadores como `$ne` o `$gt`. Se neutraliza interceptando y sanitizando las entradas mediante middlewares de desinfección profunda.

En el próximo capítulo, estudiaremos el diagnóstico y afinamiento de nuestros clusters en caliente, analizando el **Monitoreo, Profiling y Optimización de Consultas**.

---

[← Capítulo anterior (Capítulo 9)](09-indexacion-y-busqueda.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 11) →](11-monitoreo-y-optimizacion.md)
