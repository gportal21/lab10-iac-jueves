# Respuestas — LAB10. Preguntas a responder

## 1. ¿Por qué necesitamos Loki además de Prometheus si ya tenemos `/metrics`?

`/metrics` y Prometheus solo manejan **series numéricas en el tiempo** (contadores, gauges, histogramas): "cuánto" y "cuándo", pero no "qué pasó exactamente". Prometheus no está pensado para almacenar texto libre ni para indexar el contenido de eventos individuales.

Loki, en cambio, almacena **logs** (contexto: mensajes de error, trazas, stack traces, Ids de petición, etc.), indexando solo un pequeño conjunto de etiquetas y permitiendo buscar/filtrar.

Las métricas te dicen _que_ algo va mal (CPU > 50%) y los logs te dicen _por qué_ (un error concreto que disparó esa carga). Necesitamos ambas señales porque responden preguntas distintas y se complementan al depurar incidentes.

## 2. ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?

Al estar definidas en archivos de provisioning versionados, la configuración es **reproducible**: cualquier persona que levante el stack con `docker compose up` obtiene exactamente las mismas datasources (Prometheus y Loki) ya conectadas, sin pasos manuales propensos a error u olvido.

Además, los cambios quedan en control de versiones (se pueden revisar, auditar y revertir con git), se elimina el "drift" entre entornos (dev/staging/prod) y se puede recrear el entorno completo desde cero (`docker compose down -v && up`) sin perder la configuración de Grafana.

## 3. El panel "CPU contenedor" y el panel "CPU host" pueden mostrar valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?

El panel de **CPU del backend** (`rate(backend_process_cpu_seconds_total[1m]) * 100`) mide el uso de CPU del **proceso de esa aplicación específica**. El panel de **CPU del host** (`100 - avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100`) mide el uso **agregado de toda la máquina**, incluyendo el sistema operativo, otros contenedores y procesos.

Por eso pueden diferir mucho: el backend puede estar consumiendo el 100% de un núcleo (proceso saturado) mientras el host, con varios núcleos y poca otra carga, muestra un porcentaje global bajo; o al revés, otro proceso puede saturar el host sin que el backend esté haciendo nada.

Para **alertar sobre una aplicación concreta** hay que usar la métrica del **proceso/contenedor de esa app** (CPU backend), porque muestra directamente su comportamiento independientemente de lo que ocurra en el resto de la máquina. La métrica de host sirve para alertas de **infraestructura general** (la máquina se está quedando sin CPU), no para diagnosticar un servicio puntual.

## 4. ¿Qué diferencia hay entre el _evaluation interval_ y el _pending period_ de una alarma?

El **evaluation interval** es cada cuánto tiempo Grafana vuelve a ejecutar la query y evaluar la condición de la alarma (en este laboratorio, cada `10s`).

El **pending period** es el tiempo que la condición debe mantenerse cumplida de forma **continua** (durante evaluaciones consecutivas) antes de que la alarma pase de `Pending` a `Firing` (`30s`).

En resumen: el evaluation interval define la **frecuencia del check**, y el pending period define **cuánto tiempo continuo** debe cumplirse la condición antes de disparar realmente la alerta, evitando falsas alarmas por picos momentáneos.

---

# Explicación de cada componente del stack

- **node-exporter**: Expone métricas del host (CPU, memoria, disco, red) para que Prometheus las recolecte.
- **cAdvisor**: Expone métricas de uso de recursos por contenedor (CPU, memoria) para Prometheus.
- **Prometheus**: Recolecta (scrape) y almacena las métricas de node-exporter, cAdvisor y de las apps (`/metrics`), y evalúa las reglas de alerta.
- **Loki**: Almacena logs enviados por Alloy, indexando solo etiquetas para permitir búsquedas eficientes con LogQL.
- **Grafana Alloy**: Recolecta los logs de stdout de todos los contenedores y los envía a Loki.
- **Grafana**: Herramienta de visualización; se conecta a Prometheus y Loki para construir dashboards y gestiona las alarmas (Alerting), notificando vía contact points.
- **Frontend y Backend ("Hello World")**: Aplicaciones de ejemplo que generan tráfico real, exponen sus propias métricas en `/metrics` y escriben logs en JSON para que Alloy los recolecte.
