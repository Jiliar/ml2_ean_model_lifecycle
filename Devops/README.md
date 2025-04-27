Para implementar el monitoreo y rollback automático usando Docker, Prometheus y Grafana para monitorear el rendimiento de tu modelo en tiempo real, y configurar alertas para realizar un rollback automático si el rendimiento cae por debajo de un umbral predefinido, sigue estos pasos.

1. Crear un Contenedor Docker para Prometheus

Primero, vamos a configurar un contenedor para Prometheus para recolectar las métricas de tu modelo.

Paso 1: Crear un prometheus.yml

Configura el archivo prometheus.yml para que Prometheus pueda recolectar métricas de tu modelo. En este ejemplo, se está utilizando un servicio en el puerto 5000 donde el modelo expone las métricas.

global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'ml_model_metrics'
    static_configs:
      - targets: ['model-service:5000']  # Aquí apuntas al contenedor que ejecuta tu modelo

Paso 2: Crear Dockerfile para Prometheus

Crea un Dockerfile para Prometheus (o usa una imagen preconfigurada de Prometheus).

# Usa la imagen oficial de Prometheus
FROM prom/prometheus:latest

# Copia tu archivo de configuración de Prometheus al contenedor
COPY prometheus.yml /etc/prometheus/prometheus.yml

# Exponer el puerto de Prometheus
EXPOSE 9090

Paso 3: Crear un contenedor para Prometheus

Crea un contenedor Docker para Prometheus con el siguiente comando:

docker build -t prometheus-custom .
docker run -d -p 9090:9090 prometheus-custom

Ahora Prometheus debería estar corriendo en http://localhost:9090.

2. Crear un Contenedor Docker para Grafana

Grafana se utilizará para visualizar las métricas recolectadas por Prometheus.

Paso 1: Crear un contenedor para Grafana

Grafana tiene una imagen oficial en Docker, por lo que puedes usarla directamente para crear un contenedor:

docker run -d -p 3000:3000 --name=grafana grafana/grafana

Esto pondrá Grafana en http://localhost:3000.

Paso 2: Configurar la fuente de datos de Prometheus en Grafana
	1.	Inicia sesión en Grafana (admin/admin es la contraseña predeterminada).
	2.	Ve a Configuration (el icono del engranaje) > Data Sources > Add Data Source.
	3.	Selecciona Prometheus y configura la URL de Prometheus: http://<host>:9090 (si estás usando Docker, puede ser http://prometheus:9090 dependiendo de tu configuración).
	4.	Guarda y prueba la conexión.

3. Exponer Métricas del Modelo en Docker

Ahora necesitamos que tu modelo exponga métricas en formato de Prometheus. Si estás utilizando Flask para servir tu modelo, podemos usar la librería prometheus_client para exponer las métricas.

Paso 1: Instalar prometheus_client

pip install prometheus_client

Paso 2: Exponer métricas en Flask

Aquí te dejo un ejemplo de cómo exponer las métricas en Flask:

from flask import Flask
from prometheus_client import start_http_server, Gauge
from keras.models import load_model
import numpy as np

app = Flask(__name__)

# Define una métrica para la precisión del modelo
precision_gauge = Gauge('model_precision', 'Precision del modelo')

# Cargar tu modelo
autoencoder = load_model("autoencoder-model.h5")

# Función para obtener la precisión
def get_model_precision():
    # Aquí usas tu lógica de evaluación del modelo
    X_pred = autoencoder.predict(X_test)
    precision = np.mean(np.equal(X_pred, X_test))  # Simple ejemplo de precisión
    return precision

@app.route('/metrics')
def metrics():
    precision = get_model_precision()
    precision_gauge.set(precision)  # Actualizar la métrica
    return generate_latest()  # Exponer métricas en formato Prometheus

if __name__ == "__main__":
    start_http_server(5000)  # Inicia el servidor Prometheus
    app.run(host='0.0.0.0', port=5000)  # Exponer tu API

4. Configuración de Alertas en Grafana

Una vez que hayas configurado Grafana, puedes crear un panel para visualizar las métricas y establecer alertas.
	1.	En tu dashboard de Grafana, selecciona el panel que deseas configurar.
	2.	Ve a Alert en el panel.
	3.	Configura una alerta que se dispare si la precisión del modelo cae por debajo de un umbral determinado (por ejemplo, precision < 0.75).
	4.	Configura notificaciones (por ejemplo, Slack, Email) para que te avise cuando la alerta se dispare.

Ejemplo de alerta:

WHEN avg() OF query(A, 5m, now) IS BELOW 0.75

5. Configuración de Rollback Automático en Docker

Cuando la alerta de Grafana se dispare (por ejemplo, si la precisión cae por debajo de 0.75), puedes ejecutar un script para hacer un rollback automático al modelo anterior. Esto se puede hacer mediante un script Docker.

Paso 1: Crear un script para el rollback:

#!/bin/bash

# Detener y eliminar el contenedor actual del modelo
docker stop model-container
docker rm model-container

# Descargar y ejecutar una versión anterior del modelo
docker pull <previous_model_image>
docker run -d -p 5000:5000 --name model-container <previous_model_image>

Paso 2: Integrar el rollback con las alertas de Grafana:

Puedes usar Webhook o Alertmanager en Grafana para llamar a este script cuando la alerta se dispare. Para configurar un webhook:
	1.	Ve a Alerting > Notification channels en Grafana.
	2.	Configura un webhook que apunte a un servidor que ejecute el script de rollback.
	3.	Cuando la alerta se dispare, el webhook ejecutará el script de rollback.

6. Conclusión

Con esta solución, tendrás un sistema robusto de monitoreo en tiempo real utilizando Prometheus y Grafana, donde las métricas de tu modelo de ML se monitorean continuamente. Si el rendimiento cae por debajo de un umbral predefinido, se ejecutará un rollback automático utilizando Docker, asegurando que siempre se utilice la versión más estable del modelo.