# Capítulo 14: Observabilidad en Entrega Continua: Grafana, Prometheus y OpenTelemetry

> "El monitoreo reactivo tradicional basado en revisar logs físicos en servidores después de que el sitio se ha caído es una práctica obsoleta de la prehistoria del software. La observabilidad moderna exige telemetría en tiempo real, correlación tridimensional de métricas, trazas y logs, y sistemas de alerta predictivos antes de que el usuario sienta la latencia."

Cuando implementamos la entrega continua avanzada (CD) con estrategias Canary o GitOps, el éxito del despliegue depende de nuestra capacidad para medir y evaluar la salud del sistema de forma científica en producción.

Si despliegas una versión y no tienes visibilidad de su comportamiento en caliente, estarás volando a ciegas. En este capítulo, desglosaremos los pilares de la **Observabilidad**, estudiaremos la arquitectura de **OpenTelemetry**, **Prometheus** y **Grafana**, y diseñaremos una regla de alerta en caliente en YAML para disparar auto-rollbacks.

---

## 14.1 Los Tres Pilares de la Observabilidad y Golden Signals

La observabilidad no consiste en acumular logs de texto plano de forma infinita. Se fundamenta en recopilar de forma correlacionada tres tipos de telemetría:

1. **Métricas (Metrics)**: Datos cuantitativos estructurados a lo largo del tiempo (ej. porcentaje de uso de CPU, cantidad de peticiones HTTP concurrentes, tasa de errores). Son sumamente baratas de almacenar y excelentes para disparar alertas inmediatas.
2. **Logs (Bitácoras)**: Texto estructurado que describe un evento particular que ocurrió en un milisegundo específico (ej. un stack trace de excepción de base de datos). Son costosos de almacenar pero cruciales para diagnosticar la causa raíz del fallo (*Root Cause Analysis*).
3. **Trazas (Traces)**: El mapa de vida completo de una petición HTTP a medida que recorre decenas de microservicios independientes en red de forma distribuida (ayuda a identificar qué microservicio exacto de la cadena está causando latencia).

### Las 4 Golden Signals (Señales de Oro) del Monitoreo:
Definidas por Google en su disciplina de SRE (Site Reliability Engineering):
* **Latencia**: El tiempo que tarda en resolverse una petición.
* **Tráfico**: Demanda del sistema (ej. peticiones HTTP por segundo).
* **Errores**: Tasa de peticiones que fallan (ej. HTTP 5xx).
* **Saturación**: Qué tan "lleno" está tu servicio (ej. memoria RAM libre o hilos de base de datos en cola).

---

## 14.2 El Ecosistema: Prometheus, Grafana y OpenTelemetry

Para instrumentar y visualizar la observabilidad sin acoplar nuestro software a marcas comerciales cerradas, la industria se apoya en el estándar de la **CNCF (Cloud Native Computing Foundation)**:

* **OpenTelemetry (OTel)**: El estándar universal de APIs e instrumentación de código. Permite a los desarrolladores instrumentar sus aplicaciones TypeScript, Go o Java una sola vez de forma abierta, transmitiendo la telemetría a cualquier destino.
* **Prometheus**: Base de datos de series temporales de alto rendimiento que recopila métricas mediante el modelo de **Pull (Scraping)** en red a intervalos definidos (ej. cada 15 segundos consulta el endpoint `/metrics` de tus servidores).
* **Grafana**: El motor de renderizado y visualización analítica. Permite consolidar datos de múltiples fuentes (Prometheus, Loki, Elasticsearch) en hermosos Dashboards en tiempo real.

---

> [!NOTE]
> ### 📟 La Sala de Control con Pantallas Clínicas de Pacientes
> 
> Entendamos los pilares de la observabilidad y los sistemas de alertas predictivos utilizando una analogía física e intuitiva de un hospital médico:
> 
> - **El Monitoreo Reactivo Tradicional (El Médico Forense)**:
>   - Imagina que un hospital no tiene sensores, ni pantallas, ni enfermeras en las salas de cuidados intensivos. Los pacientes están acostados solos en habitaciones cerradas.
>   - Si un paciente sufre una arritmia cardíaca grave a la medianoche, nadie se entera. Al día siguiente a las 08:00 AM, un enfermero abre la puerta y encuentra al paciente fallecido. 
>   - Llama al médico forense para que le abra el pecho al cuerpo, lea el reporte de autopsia y determine de qué murió (**revisar archivos de logs en texto plano un día después de la caída del servidor**). El diagnóstico es exacto, pero el paciente ya está muerto.
> 
> - **La Observabilidad Moderna (La Unidad de Cuidados Intensivos Digital)**:
>   - Decides instrumentar al paciente con sensores biométricos de última generación conectados a su pecho y brazos (**La instrumentación con OpenTelemetry**).
>   - Conectas los sensores a una **computadora con pantallas y alarmas en tiempo real en la estación de enfermería (Prometheus y Grafana)**.
>   - La pantalla muestra constantemente tres señales críticas:
>     - *El ritmo cardíaco y presión arterial (Métricas en Prometheus)*.
>     - *Un pitido secuencial por cada latido del corazón (Logs transaccionales)*.
>     - *Un monitor tridimensional que muestra cómo fluye la sangre desde las arterias principales hasta el cerebro segundo a segundo (Trazas distribuidas)*.
>   - **La Alerta (Alertmanager)**: Si la presión del paciente baja del umbral saludable, la pantalla del lobby no espera a que el paciente fallezca. Dispara una **alarma sonora roja de alta intensidad (Alertmanager)**. 
>   - Las enfermeras corren a inyectarle medicamentos estabilizadores de forma automatizada, salvándole la vida al paciente en microsegundos antes de que sufra cualquier daño permanente.

---

## 14.3 Código YAML: Reglas de Alerta en Prometheus (Alerting Rules)

A continuación, implementaremos la configuración real y sin placeholders para definir **Prometheus Alerting Rules**. El archivo describe una regla de alerta en Kubernetes que vigila la tasa de errores HTTP 500 y las latencias de nuestra API financiera, disparando una notificación de nivel crítico a través de **Alertmanager** si el sistema supera los umbrales seguros:

### `reglasAlertaPrometheus.yml`
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: alertas-api-financiera
  namespace: monitoring
spec:
  groups:
    - name: api-financiera-metrics
      rules:
        # Alerta 1: Tasa de errores HTTP 5xx excede el 5% en un intervalo de 5 minutos
        - alert: AltaTasaErroresHttp5xx
          # PromQL (Prometheus Query Language) para calcular el porcentaje de errores
          expr: >
            sum(rate(http_requests_total{status=~"5.."}[5m])) 
            / 
            sum(rate(http_requests_total[5m])) * 100 > 5
          for: 2m # La condición debe cumplirse de forma ininterrumpida por 2 minutos para disparar
          labels:
            severity: critical
            team: devops-infra
          annotations:
            summary: "Alta tasa de errores HTTP 5xx en la API financiera"
            description: "La API financiera está arrojando un ${{ $value }}% de errores de nivel 5xx (Server Error) en los últimos 5 minutos en el entorno de producción."

        # Alerta 2: La latencia (percentil 95) excede los 200ms
        - alert: LatenciaExcesivaPercentil95
          expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 0.200
          for: 5m
          labels:
            severity: warning
            team: devops-infra
          annotations:
            summary: "Latencia percentil 95 lenta en producción"
            description: "El 95% de los usuarios experimentan latencias de respuesta superiores a 200ms (Valor actual: ${{ $value }}s) en los últimos 5 minutos."
```

---

## Resumen del Capítulo

* **Observabilidad** es la capacidad de deducir el estado interno de un sistema complejo analizando sus salidas externas organizadas en: **Métricas**, **Logs** y **Trazas**.
* **OpenTelemetry** es el estándar de instrumentación abierta que unifica la recopilación de datos evitando el acoplamiento a marcas comerciales propietarias.
* **Prometheus** recopila métricas mediante el modelo Pull (Scraping) en red, y **Grafana** las procesa y visualiza en tiempo real de forma gráfica en Dashboards analíticos.
* Definir reglas de alerta proactivas en Prometheus mediante lenguaje **PromQL** permite anticiparse a incidentes y automatizar rollbacks de despliegues fallidos.

En el próximo capítulo, consolidaremos y fusionaremos todo lo aprendido a lo largo de este volumen mediante el desarrollo de nuestro **Proyecto Integrador: Pipeline Global Híbrido Multicloud**.

---

[← Capítulo anterior (Capítulo 13)](13-seguridad-secretos-vault.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 15) →](15-proyecto-integrador-devops.md)
