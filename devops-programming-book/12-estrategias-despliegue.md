# Capítulo 12: Estrategias de Despliegue Avanzadas: Blue-Green y Canary Deployments

> "El despliegue de una nueva versión no puede ser un acto de fe. Lanzar el código directamente al $100\%$ de los usuarios concurrentes sin un mecanismo progresivo de enrutamiento y rollbacks automáticos basados en telemetría es una ruleta rusa operativa."

En la ingeniería de DevOps tradicional, subir software a producción implicaba realizar "mantenimientos nocturnos" que detenían el servicio, o cruzar los dedos con la esperanza de que el nuevo release no contuviera fugas de memoria o fallos lógicos graves.

En la actualidad, las arquitecturas cloud de alto rendimiento exigen **Cero Caídas (Zero-Downtime Deployments)**. En este capítulo, estudiaremos los internals del enrutamiento de tráfico progresivo, analizaremos las diferencias entre estrategias **Blue-Green** y **Canary Deployments**, y diseñaremos un manifiesto de Kubernetes Ingress para lanzamientos Canary con división de tráfico.

---

## 12.1 Blue-Green Deployments: Mitigación Total del Riesgo

**Blue-Green** es una estrategia de despliegue que reduce el riesgo de caídas manteniendo **dos entornos físicos idénticos de producción en paralelo**:

* **Entorno Blue (Azul)**: El entorno activo actual que atiende a todos los usuarios en caliente (ej. versión `v1.1.0`).
* **Entorno Green (Verde)**: El entorno inactivo donde se despliega y prueba de forma exhaustiva la nueva versión del software (ej. versión `v1.2.0`).

### Mecánica del Despliegue:
1. La nueva versión se despliega silenciosamente en el entorno **Green**.
2. El equipo realiza validaciones lógicas en caliente sin interferir con los usuarios reales de **Blue**.
3. Cuando todo está aprobado, el enrutador de tráfico principal (un balanceador de carga o proxy inverso) cambia instantáneamente de dirección: **el tráfico entrante se desvía al $100\%$ de Green**.
4. **La Salvaguarda**: Si a los 5 minutos del cambio se detecta un fallo crítico imprevisto en producción, el balanceador vuelve a apuntar de forma inmediata a **Blue**, logrando un **Rollback instantáneo de 1 segundo**.

---

## 12.2 Canary Deployments: Incrementos Progresivos de Tráfico

La estrategia **Canary (Canario)** debe su nombre a los canarios que los mineros de carbón llevaban a los túneles subterráneos: si el aire se llenaba de gases tóxicos invisibles, el canario moría primero alertando a los mineros para que evacuaran rápido.

En el desarrollo de software:
* En lugar de mandar el $100\%$ del tráfico al nuevo entorno Green, **desplegamos la nueva versión (Canary) junto a la versión estable (Production) y le enrutamos únicamente una pequeña fracción de usuarios concurrentes** (ej. sólo el $5\%$ o $10\%$).
* Si el canario sobrevive (las métricas de latencia y porcentaje de errores HTTP 500 permanecen normales en el $5\%$), incrementamos progresivamente el tráfico al $25\%$, $50\%$ y finalmente al $100\%$.
* Si las alertas lógicas de negocio o sistemas se disparan, el balanceador corta el tráfico del $5\%$ del canario en caliente redirigiéndolo a la versión estable, limitando el radio de impacto del bug al mínimo.

---

> [!NOTE]
> ### 🔀 Los Interruptores de Desvío del Acueducto de la Ciudad
> 
> Entendamos la diferencia conceptual entre Blue-Green y Canary Deployments utilizando una analogía física de ingeniería hidráulica:
> 
> - **El Despliegue Tradicional (Detener el Suministro e Inundar)**:
>   - Imagina que necesitas cambiar la tubería principal de agua potable de una ciudad entera.
>   - En el enfoque tradicional inseguro, cierras la llave de paso de agua de toda la ciudad durante 12 horas (caída del sitio), cambias el tubo a prisa y abres la llave con toda la presión del agua de golpe. 
>   - Si el tubo nuevo tiene una fuga, el agua inundará las calles y dejará a toda la ciudad sin suministro indefinidamente hasta repararlo.
> 
> - **El Enfoque Blue-Green (El Segundo Acueducto Gemelo Alternativo)**:
>   - Decides no arriesgarte. Construyes una **segunda red de tuberías idéntica y paralela a la primera (El Acueducto Verde)** de forma aislada mientras la ciudad sigue bebiendo agua de la red original (El Acueducto Azul).
>   - Pruebas el acueducto verde con agua a presión. Al confirmar que no hay fugas, vas al gran interruptor de control de la ciudad.
>   - Mueves la palanca gigante en 1 segundo. El agua fluye instantáneamente por el Acueducto Verde y los ciudadanos reciben el agua por las nuevas tuberías sin haber sentido un solo segundo de corte de servicio. Si algo falla, regresas la palanca a la red Azul en un pestañeo.
> 
> - **El Enfoque Canary (La Pequeña Válvula de Desvío de Barrio)**:
>   - Tienes el tubo nuevo, pero quieres probarlo con agua real sin arriesgar a la ciudad entera.
>   - Instalas la tubería nueva, pero colocas un **pequeño interruptor regulador de presión (Ingress Canary Controller)** que desvía **únicamente el $5\%$ del caudal de agua** hacia un solo barrio periférico experimental de la ciudad (El Canario).
>   - Los ingenieros vigilan los sensores de presión de las casas de ese barrio. Si se detecta que el agua sale turbia o con baja presión, cierras la válvula del $5\%$ de inmediato. 
>   - Solo el $5\%$ del barrio experimentó agua turbia por 2 minutos, mientras que el $95\%$ de los ciudadanos de la metrópolis siguieron consumiendo agua perfecta de la red estable de forma continua.

---

## 12.3 Código YAML: Lanzamiento Canary en Kubernetes con Nginx Ingress

A continuación, implementaremos la configuración real para orquestar un **Canary Deployment** en Kubernetes utilizando el controlador popular de **Nginx Ingress**. Definiremos el Ingress de soporte de la versión Canary que desvía de forma automática e inteligente exactamente el **$10\%$ del tráfico de Internet** hacia nuestro servicio experimental en caliente:

#### [ingressCanary.yml](file:///Users/andres/Documents/biblioteca/devops-programming-book/kubernetes/ingress/ingressCanary.yml)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-financiera-canary
  namespace: staging-apps
  annotations:
    # 1. Indicar que el controlador de Nginx de Kubernetes Ingress gestione este balanceo
    kubernetes.io/ingress.class: nginx
    
    # 2. Habilitar la característica Canary de Nginx
    nginx.ingress.kubernetes.io/canary: "true"
    
    # 3. Definir la proporción física del tráfico que deseamos desviar en caliente
    # En este caso, el Ingress enviará el 10% de las peticiones HTTP concurrentes al servicio Canary
    # y el 90% restante al servicio de producción estable por defecto.
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
    - host: api.mi-banco.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                # El tráfico desviado del 10% llegará al Service Canary de la versión nueva
                name: api-financiera-service-canary
                port:
                  number: 80
```

---

## Resumen del Capítulo

* **Blue-Green Deployments** minimizan riesgos manteniendo dos entornos físicos idénticos en paralelo, logrando migraciones rápidas y rollbacks de 1 segundo mediante redirección de tráfico al $100\%$.
* **Canary Deployments** inyectan paulatinamente el cambio a una fracción diminuta de usuarios reales de producción (ej. $10\%$) antes de iniciar una propagación global.
* Las anotaciones **`canary-weight`** de controladores como Nginx Ingress delegan el enrutamiento progresivo de tráfico directamente a la capa de red del balanceador del clúster.
* La automatización de Canary exige el enlace de sistemas de observabilidad (Prometheus) para disparar **auto-rollbacks** inmediatos si el porcentaje de errores HTTP del canario supera los límites aceptables.

En el próximo capítulo, abordaremos la protección de llaves criptográficas y credenciales en caliente mediante el estudio de **Secretos y Seguridad de Pipeline: HashiCorp Vault e Integraciones** en producción.

---

[← Capítulo anterior (Capítulo 11)](11-helm-kustomize-kubernetes.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 13) →](13-seguridad-secretos-vault.md)
