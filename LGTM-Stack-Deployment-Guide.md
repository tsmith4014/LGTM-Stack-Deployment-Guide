# LGTM Stack Deployment Guide

This deployment guide provides detailed steps to set up the Grafana LGTM stack (Loki, Grafana, Tempo, Mimir). It focuses on deploying each component to provide a comprehensive observability solution for your infrastructure.

## Prerequisites

### 1. System Requirements

- **Operating System**: Linux (Ubuntu 20.04 or newer recommended)
- **RAM**: Minimum 4 GB for testing, 8 GB or more for production
- **CPU**: 2 vCPUs for testing, 4 or more for production
- **Docker**: Installed and configured
- **Kubernetes (Optional)**: Minikube or any Kubernetes cluster for Kubernetes-based deployment
- **Network**: Internet access for downloading images and software

### 2. Tools Required

- **Docker**: Install Docker from [Docker's official website](https://docs.docker.com/get-docker/).
- **Docker Compose**: For managing multiple containers.
- **Kubernetes**: Optional, for deploying in a Kubernetes environment.
- **Helm (Optional)**: For deploying applications in Kubernetes clusters.

## Step 1: Setting Up Loki

Loki is the log aggregation tool that stores and queries logs. It is highly scalable and can be easily integrated with Grafana.

### 1.1 Install Loki Using Docker

Create a `loki-config.yaml` for basic configuration:

```yaml
auth_enabled: false
server:
  http_listen_port: 3100
ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
schema_config:
  configs:
    - from: 2023-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 168h
storage_config:
  boltdb_shipper:
    active_index_directory: /tmp/loki/index
    cache_location: /tmp/loki/cache
    shared_store: filesystem
  filesystem:
    directory: /tmp/loki/chunks
limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
```

Run Loki using Docker:

```bash
docker run -d --name loki \
 -p 3100:3100 \
 -v $(pwd)/loki-config.yaml:/etc/loki/local-config.yaml \
 grafana/loki:latest \
 -config.file=/etc/loki/local-config.yaml
```

### 1.2 Loki Verification

- Visit `http://localhost:3100/metrics` to ensure Loki is up and running.
- You can also query logs using Grafana or directly with `curl`.

## Step 2: Setting Up Grafana

Grafana provides visualization for metrics, logs, and traces.

### 2.1 Install Grafana Using Docker

Run the following command to install Grafana:

```bash
docker run -d -p 3000:3000 \
 --name=grafana \
 grafana/grafana:latest
```

### 2.2 Configure Grafana

- **Access Grafana**: Open your browser and navigate to `http://localhost:3000`.
- **Default Login**: Username: `admin`, Password: `admin` (you will be prompted to change it).
- **Add Loki as Data Source**:
  1. Go to **Configuration** -> **Data Sources**.
  2. Click **Add data source**.
  3. Select **Loki** and set URL to `http://loki:3100` or `http://localhost:3100`.

## Step 3: Setting Up Tempo

Tempo is a high-scale, cost-effective distributed tracing backend.

### 3.1 Install Tempo Using Docker

Create a `tempo-config.yaml` with basic configuration:

```yaml
server:
  http_listen_port: 3200

distributor:
  receivers:
    jaeger:
      protocols:
        thrift_binary:
        thrift_compact:
        grpc:
    otlp:
      protocols:
        grpc:
        http:
    zipkin:
      endpoint: 0.0.0.0:9411

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1

storage:
  trace:
    backend: local
    local:
      path: /tmp/tempo/traces
```

Run Tempo using Docker:

```bash
docker run -d --name tempo \
 -p 3200:3200 \
 -p 9411:9411 \
 -v $(pwd)/tempo-config.yaml:/etc/tempo/tempo.yaml \
 grafana/tempo:latest \
 -config.file=/etc/tempo/tempo.yaml
```

### 3.2 Tempo Verification

- Ensure Tempo is running by accessing `http://localhost:3200/metrics`.
- Tempo listens on standard ports for receiving traces via various protocols.

## Step 4: Setting Up OpenTelemetry

OpenTelemetry provides instrumentation for metrics, logs, and traces across distributed systems.

### 4.1 Instrument Your Application

- **Install OpenTelemetry SDK**: Use language-specific SDKs to instrument your code. For example, in Python:

  ```bash
  pip install opentelemetry-sdk opentelemetry-exporter-otlp
  ```

- **Configure Exporter**: Set up your application to export traces to Tempo via OTLP or Jaeger protocols.

### 4.2 Example Configuration in Python

```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Set up the tracer provider
trace.set_tracer_provider(TracerProvider())

# Configure the OTLP exporter to send data to Tempo
otlp_exporter = OTLPSpanExporter(endpoint="http://localhost:4317", insecure=True)

# Create a span processor and add it to the tracer provider
span_processor = BatchSpanProcessor(otlp_exporter)
trace.get_tracer_provider().add_span_processor(span_processor)

# Now you can start creating spans
tracer = trace.get_tracer(__name__)
with tracer.start_as_current_span("foo"):
    print("Hello, world!")
```

**Note**: Ensure that your application is sending traces to the correct endpoint where Tempo is listening.

## Step 5: Setting Up Mimir

Mimir is a scalable storage solution for Prometheus metrics.

### 5.1 Install Mimir Using Docker

Create a `mimir-config.yaml` with basic configuration:

```yaml
server:
  http_listen_port: 9009

multitenancy_enabled: false

distributor:
  shard_by_all_labels: true

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1

storage:
  tsdb:
    dir: /var/tsdb

limits:
  enforce_metric_name: false
```

Run Mimir using Docker:

```bash
docker run -d --name mimir \
 -p 9009:9009 \
 -v $(pwd)/mimir-config.yaml:/etc/mimir/mimir.yaml \
 grafana/mimir:latest \
 -config.file=/etc/mimir/mimir.yaml
```

### 5.2 Mimir Verification

- Access `http://localhost:9009/metrics` to verify Mimir is running.
- Add Mimir as a Prometheus data source in Grafana.

## Step 6: Deploying the Full Stack Using Docker Compose

To simplify the deployment process, you can use Docker Compose to bring all components up at once.

### 6.1 Docker Compose File

Create a `docker-compose.yaml` file with the following content:

```yaml
version: "3.7"
services:
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
    command: -config.file=/etc/loki/local-config.yaml

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    depends_on:
      - loki
      - tempo
      - mimir

  tempo:
    image: grafana/tempo:latest
    container_name: tempo
    ports:
      - "3200:3200"
      - "9411:9411"
    volumes:
      - ./tempo-config.yaml:/etc/tempo/tempo.yaml
    command: -config.file=/etc/tempo/tempo.yaml

  mimir:
    image: grafana/mimir:latest
    container_name: mimir
    ports:
      - "9009:9009"
    volumes:
      - ./mimir-config.yaml:/etc/mimir/mimir.yaml
    command: -config.file=/etc/mimir/mimir.yaml
```

### 6.2 Deploy the Stack

Run the following command to bring up the entire stack:

```bash
docker-compose up -d
```

### 6.3 Configure Grafana Data Sources

- **Loki**:
  - URL: `http://loki:3100`
- **Tempo**:
  - URL: `http://tempo:3200`
- **Mimir**:
  - URL: `http://mimir:9009`
  - Set as a Prometheus data source.

### 6.4 Verification

- **Grafana**: Navigate to `http://localhost:3000` and verify that all data sources are connected.
- **Loki**: Check that logs are being ingested and can be queried from Grafana.
- **Tempo**: Ensure traces are available in Grafana's **Explore** section.
- **Mimir**: Confirm that metrics are visible in Grafana dashboards.

## Step 7: Kubernetes Deployment (Optional)

For a production-grade deployment, you can use Kubernetes to manage the LGTM stack.

### 7.1 Helm Charts

Helm is the preferred method for deploying the LGTM stack in Kubernetes.

- **Install Helm**: Follow the instructions at [Helm's official website](https://helm.sh/docs/intro/install/).

### 7.2 Deploy Loki, Grafana, Tempo, Mimir

Add the Grafana Helm repository:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

**Deploy Loki**:

```bash
helm install loki grafana/loki-stack
```

**Deploy Grafana**:

```bash
helm install grafana grafana/grafana
```

**Deploy Tempo**:

```bash
helm install tempo grafana/tempo-distributed
```

**Deploy Mimir**:

```bash
helm install mimir grafana/mimir-distributed
```

### 7.3 Configure Services

- **Expose Services**: Use Kubernetes Services and Ingress to expose Grafana, Loki, Tempo, and Mimir externally.
- **Configure Grafana**: Update data sources in Grafana to point to the appropriate service endpoints.

### 7.4 Verification

- **Access Grafana**: Use the service's external IP or domain to access Grafana.
- **Validate Data Sources**: Ensure all data sources (Loki, Tempo, Mimir) are connected and operational.
- **Monitor Logs, Traces, Metrics**: Verify that logs, traces, and metrics are being collected and displayed correctly.

## Summary

Deploying the Grafana LGTM stack involves setting up each component carefully to ensure complete observability:

- **Loki**: For log aggregation and querying.
- **Grafana**: For visualizing metrics, logs, and traces.
- **Tempo**: As the tracing backend for distributed tracing.
- **Mimir**: For scalable, long-term storage of Prometheus metrics.

Using Docker or Docker Compose provides a quick setup for testing and development environments, while Kubernetes deployment is recommended for production-grade observability solutions.

---
