# Capítulo 13: Secretos y Seguridad de Pipeline: HashiCorp Vault e Integraciones

> "Almacenar claves de API permanentes, contraseñas de bases de datos o llaves SSH escritas en texto plano en repositorios Git o en las variables de entorno de tu herramienta de CI/CD es una negligencia catastrófica. La seguridad moderna exige secretos dinámicos, efímeros, rotados continuamente y autenticaciones federadas libres de contraseñas."

En la ingeniería de DevOps, el robo de credenciales en pipelines es uno de los vectores de ataque más explotados por los hackers. Si un desarrollador sube accidentalmente una llave de AWS a un repositorio Git público, los bots automáticos de escaneo la detectarán en menos de 30 segundos, procediendo a aprovisionar máquinas virtuales masivas de minería de criptomonedas y generándote facturas de miles de dólares en minutos.

Para erradicar esta plaga de fugas de secretos, la industria implementa motores de gestión de identidades centralizados como **HashiCorp Vault** y protocolos de federación de identidades como **OIDC (OpenID Connect)**. En este capítulo, estudiaremos el escaneo preventivo de secretos, los **Secretos Dinámicos** y diseñaremos un pipeline en GitHub Actions que utiliza OIDC para autenticarse sin contraseñas contra AWS.

---

## 13.1 HashiCorp Vault: El Estandar de Gestión de Secretos

**HashiCorp Vault** es la solución líder de almacenamiento y control de accesos a secretos en caliente. Sus internals físicos destacan por:

1. **Cifrado en Reposo**: Todos los datos se cifran utilizando AES-256-GCM antes de tocar el almacenamiento físico de disco.
2. **Secretos Dinámicos (Dynamic Secrets)**: La joya de la corona. En lugar de almacenar una contraseña de base de datos fija de larga duración, el pipeline le solicita credenciales a Vault. Vault se comunica con el motor de base de datos (ej. PostgreSQL), **crea un usuario transaccional temporal único de un solo uso con un TTL (Time-To-Live) de 15 minutos**, y se lo entrega al pipeline. Al pasar los 15 minutos, Vault destruye físicamente el usuario de la base de datos de forma automática, anulando cualquier credencial robada.
3. **Sellado/Desellado (Shamir's Secret Sharing)**: El almacenamiento se cifra con una llave maestra que se fragmenta matemáticamente en múltiples llaves secundarias. Para desellar el servidor al iniciar, se requiere la concurrencia física de múltiples ingenieros senior ingresando sus fragmentos de llave de forma coordinada, impidiendo fraudes internos.

---

## 13.2 Autenticación Federada OIDC (Passwordless CI/CD)

Tradicionalmente, para que tu pipeline desplegara recursos en nubes como AWS, debías crear un usuario de IAM, generar un `AWS_ACCESS_KEY_ID` y un `AWS_SECRET_ACCESS_KEY` con validez infinita y guardarlos en los Secrets de tu repositorio.

Con **OIDC (OpenID Connect)**, eliminamos por completo las llaves permanentes:
* **Mecánica**:
  1. Configuras una relación de confianza federada de identidades entre tu proveedor cloud (AWS) y tu herramienta de CI/CD (GitHub Actions).
  2. Al arrancar un Job, el Runner de GitHub solicita un token criptográfico firmado temporal (**JWT - JSON Web Token**) de corta duración que acredita su identidad física.
  3. El Runner presenta este token a AWS.
  4. AWS valida la firma criptográfica de GitHub y le entrega de vuelta al Runner **credenciales temporales de AWS IAM que caducan de forma automática a los 60 minutos**.
* **Resultado**: No existe ninguna llave permanente guardada en ningún lugar del mundo. Si un atacante entra a tus variables del pipeline, no encontrará nada que robar.

---

> [!NOTE]
> ### 🔑 La Caja Fuerte de Combinación Dinámica de un Solo Uso
> 
> Entendamos la gestión de secretos dinámicos e integraciones OIDC libres de contraseñas utilizando una analogía de alta seguridad física:
> 
> - **El Enfoque Tradicional Inseguro (La Llave de Latón Bajo el Tapete)**:
>   - Imagina que eres dueño de una joyería fina en la ciudad.
>   - Para que tus empleados abran el local por la mañana, decides forjar una llave de latón física única, le sacas 10 copias y se las entregas a todos tus empleados (**Las Access Keys de AWS guardadas en Git/CI**).
>   - Si un empleado pierde su mochila, si le roban la llave en la calle o si se va enojado de la empresa (**fuga de secretos**), el ladrón podrá entrar a la joyería a la medianoche a robar todo el oro. 
>   - Para solucionarlo, tendrías que cambiar la cerradura física de la puerta y volver a forjar y distribuir 10 copias de llaves nuevas a prisa de forma manual, un proceso caótico y lento.
> 
> - **El Enfoque OIDC y HashiCorp Vault (La Bóveda con Token de Un Solo Uso)**:
>   - Decides tirar a la basura las llaves físicas de latón. Instalas una **bóveda digital de nivel militar (HashiCorp Vault con autenticación federada OIDC)**.
>   - Cuando un empleado llega a la puerta de la joyería por la mañana:
>     - No saca ninguna llave. Muestra un dispositivo de escaneo de iris biométrico que valida su identidad y vigila que porte el uniforme corporativo oficial (**El Token JWT firmado por GitHub Actions**).
>     - La cerradura electrónica se comunica con la central satelital de seguridad en milisegundos, valida la autenticidad del iris (**La relación de confianza federada OIDC**) y genera **un código numérico digital único temporal de 6 dígitos válido por 10 segundos**.
>     - El empleado ingresa el código, la puerta se abre y el código numérico se autodestruye en caliente del sistema para siempre.
>     - Si un hacker espía al empleado y anota el código de 6 dígitos que usó, no le servirá de nada a la medianoche: ese código ya está muerto y el sistema jamás lo volverá a aceptar. No hay ninguna llave bajo el tapete que robar.

---

## 13.3 Código YAML: Autenticación OIDC Passwordless en GitHub Actions

A continuación, implementaremos la configuración real de producción para autenticar de forma federada un pipeline de **GitHub Actions** contra la nube de **AWS** utilizando **OpenID Connect (OIDC)** de forma totalmente segura y libre de contraseñas permanentes en caliente:

#### [desplieguePasswordless.yml](file:///Users/andres/Documents/biblioteca/devops-programming-book/.github/workflows/desplieguePasswordless.yml)
```yaml
name: Tubería de Despliegue Seguro (OIDC Passwordless AWS)

on:
  push:
    branches: [ main ]

# 1. Configurar de forma explícita los permisos criptográficos mínimos requeridos
# para que el runner pueda solicitar y firmar tokens JWT de visibilidad temporal
permissions:
  id-token: write # Habilita la solicitud de ID Token federado OIDC
  contents: read  # Permiso ordinario de lectura del repositorio

jobs:
  despliegue-seguro-oidc:
    name: Despliegue Cloud Seguro OIDC
    runs-on: ubuntu-latest
    steps:
      - name: Clonar código fuente
        uses: actions/checkout@v4

      # 2. Autenticación Federada OIDC contra AWS IAM.
      # Solicitamos credenciales efímeras presentando el token JWT firmado de GitHub.
      # Únicamente requerimos configurar el rol de AWS (Role-to-Assume) mapeado de confianza.
      - name: Configurar credenciales AWS usando OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          # ARN del rol de IAM en AWS configurado con la relación de confianza de OpenID Connect
          role-to-assume: arn:aws:iam::123456789012:role/GithubActionsOIDCDeployerRole
          role-session-name: GithubActionsDeploymentSession
          aws-region: us-east-1

      # 3. Al culminar el paso anterior, el runner tiene credenciales efímeras con validez de 60 minutos
      - name: Validar credenciales temporales activas
        run: |
          aws sts get-caller-identity
          echo "Conexión a AWS establecida de forma segura sin contraseñas guardadas."
```

---

## Resumen del Capítulo

* Guardar credenciales de infraestructura permanentes en repositorios o variables de CI/CD representa un vector de ataque inmenso sujeto a exfiltraciones por bots de escaneo.
* **HashiCorp Vault** centraliza la seguridad relacional cifrando datos en reposo y aprovisionando **Secretos Dinámicos** efímeros con TTLs de corta duración de forma automatizada.
* La federación de identidades mediante **OIDC (OpenID Connect)** erradica el uso de credenciales fijas intercambiando tokens JWT de confianza por llaves efímeras válidas por 60 minutos.
* Configurar los permisos mínimos (`id-token: write`) es obligatorio para autorizar a los runners a intercambiar tokens criptográficos firmados por el proveedor cloud.

En el próximo capítulo, ingresaremos al estudio del monitoreo de nuestros pipelines e infraestructura mediante **Observabilidad en Entrega Continua: Grafana, Prometheus y OpenTelemetry** en producción.

---

[← Capítulo anterior (Capítulo 12)](12-estrategias-despliegue.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 14) →](14-observabilidad-devops.md)
