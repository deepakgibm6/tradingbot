# Monitoring & Observability

## Overview

Comprehensive observability stack for production monitoring, alerting, and debugging.

### Three Pillars of Observability
1. **Metrics**: System and application performance (Prometheus + Grafana)
2. **Logs**: Centralized logging (ELK Stack / DataDog)
3. **Traces**: Distributed tracing (AWS X-Ray / Jaeger)

---

## Metrics (Prometheus + Grafana)

### Prometheus Configuration

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'trading-platform-prod'
    environment: 'production'

scrape_configs:
  # Kubernetes API Server
  - job_name: 'kubernetes-apiservers'
    kubernetes_sd_configs:
    - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

  # Kubernetes Nodes
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
    - role: node
    relabel_configs:
    - action: labelmap
      regex: __meta_kubernetes_node_label_(.+)

  # Kubernetes Pods
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
    - role: pod
    relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)
    - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
      target_label: __address__

  # Market Data Service
  - job_name: 'market-data-service'
    static_configs:
    - targets: ['market-data-service:9090']
    metrics_path: '/metrics'

  # Strategy Engine
  - job_name: 'strategy-engine'
    static_configs:
    - targets: ['strategy-engine:9090']

  # ML Service
  - job_name: 'ml-service'
    static_configs:
    - targets: ['ml-service:9090']

  # PostgreSQL
  - job_name: 'postgresql'
    static_configs:
    - targets: ['postgres-exporter:9187']

  # Dragonfly (Redis)
  - job_name: 'dragonfly'
    static_configs:
    - targets: ['redis-exporter:9121']

  # Kafka
  - job_name: 'kafka'
    static_configs:
    - targets: ['kafka-exporter:9308']

# Alerting rules
rule_files:
  - '/etc/prometheus/alerts/*.yml'

alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 'alertmanager:9093'
```

### Application Metrics (Python FastAPI)

```python
# common/metrics.py
from prometheus_client import Counter, Histogram, Gauge, generate_latest
from fastapi import FastAPI, Response
import time

app = FastAPI()

# === REQUEST METRICS ===
http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

http_request_duration_seconds = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

# === MARKET DATA METRICS ===
market_ticks_received = Counter(
    'market_ticks_received_total',
    'Total market ticks received',
    ['symbol']
)

market_data_latency_seconds = Histogram(
    'market_data_latency_seconds',
    'Market data processing latency',
    ['symbol'],
    buckets=[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5]
)

websocket_connections = Gauge(
    'websocket_connections_active',
    'Active WebSocket connections'
)

# === STRATEGY METRICS ===
signals_generated = Counter(
    'signals_generated_total',
    'Total signals generated',
    ['strategy', 'symbol', 'action']
)

strategy_execution_duration_seconds = Histogram(
    'strategy_execution_duration_seconds',
    'Strategy execution time',
    ['strategy'],
    buckets=[0.001, 0.005, 0.01, 0.025, 0.05, 0.1]
)

# === ML METRICS ===
ml_predictions_total = Counter(
    'ml_predictions_total',
    'Total ML predictions',
    ['model', 'symbol', 'direction']
)

ml_inference_duration_seconds = Histogram(
    'ml_inference_duration_seconds',
    'ML inference latency',
    ['model'],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0]
)

ml_model_accuracy = Gauge(
    'ml_model_accuracy',
    'ML model accuracy',
    ['model', 'symbol']
)

# === BACKTEST METRICS ===
backtests_executed = Counter(
    'backtests_executed_total',
    'Total backtests executed',
    ['user_id', 'strategy']
)

backtest_duration_seconds = Histogram(
    'backtest_duration_seconds',
    'Backtest execution time',
    ['strategy'],
    buckets=[1, 5, 10, 30, 60, 120, 300]
)

# === DATABASE METRICS ===
db_query_duration_seconds = Histogram(
    'db_query_duration_seconds',
    'Database query duration',
    ['query_type', 'table'],
    buckets=[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0]
)

db_connection_pool_active = Gauge(
    'db_connection_pool_active',
    'Active database connections'
)

# === CACHE METRICS ===
cache_hits = Counter(
    'cache_hits_total',
    'Total cache hits',
    ['cache_key_prefix']
)

cache_misses = Counter(
    'cache_misses_total',
    'Total cache misses',
    ['cache_key_prefix']
)

# === Middleware for automatic instrumentation ===
@app.middleware("http")
async def metrics_middleware(request, call_next):
    start_time = time.time()
    
    response = await call_next(request)
    
    duration = time.time() - start_time
    
    # Record metrics
    http_requests_total.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()
    
    http_request_duration_seconds.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(duration)
    
    return response

# === Metrics endpoint ===
@app.get("/metrics")
async def metrics():
    return Response(content=generate_latest(), media_type="text/plain")
```

### Grafana Dashboards

#### Dashboard 1: System Overview

```json
{
  "dashboard": {
    "title": "Trading Platform - System Overview",
    "panels": [
      {
        "title": "HTTP Request Rate",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (service)"
          }
        ]
      },
      {
        "title": "HTTP Request Latency (p95)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint))"
          }
        ]
      },
      {
        "title": "Active WebSocket Connections",
        "targets": [
          {
            "expr": "sum(websocket_connections_active)"
          }
        ]
      },
      {
        "title": "CPU Usage by Service",
        "targets": [
          {
            "expr": "sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)"
          }
        ]
      },
      {
        "title": "Memory Usage by Service",
        "targets": [
          {
            "expr": "sum(container_memory_working_set_bytes) by (pod) / 1024 / 1024 / 1024"
          }
        ]
      },
      {
        "title": "Database Connection Pool",
        "targets": [
          {
            "expr": "db_connection_pool_active"
          }
        ]
      }
    ]
  }
}
```

#### Dashboard 2: Market Data

```json
{
  "dashboard": {
    "title": "Market Data Monitoring",
    "panels": [
      {
        "title": "Market Ticks Received",
        "targets": [
          {
            "expr": "sum(rate(market_ticks_received_total[1m])) by (symbol)"
          }
        ]
      },
      {
        "title": "Market Data Latency (p99)",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, sum(rate(market_data_latency_seconds_bucket[5m])) by (le, symbol))"
          }
        ]
      },
      {
        "title": "Upstox API Errors",
        "targets": [
          {
            "expr": "sum(rate(upstox_api_errors_total[5m]))"
          }
        ]
      },
      {
        "title": "Cache Hit Rate",
        "targets": [
          {
            "expr": "sum(rate(cache_hits_total[5m])) / (sum(rate(cache_hits_total[5m])) + sum(rate(cache_misses_total[5m])))"
          }
        ]
      }
    ]
  }
}
```

#### Dashboard 3: ML Performance

```json
{
  "dashboard": {
    "title": "ML Model Performance",
    "panels": [
      {
        "title": "Prediction Rate",
        "targets": [
          {
            "expr": "sum(rate(ml_predictions_total[5m])) by (model)"
          }
        ]
      },
      {
        "title": "Inference Latency (p95)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(ml_inference_duration_seconds_bucket[5m])) by (le, model))"
          }
        ]
      },
      {
        "title": "Model Accuracy (Real-time)",
        "targets": [
          {
            "expr": "ml_model_accuracy"
          }
        ]
      },
      {
        "title": "Prediction Distribution",
        "targets": [
          {
            "expr": "sum(ml_predictions_total) by (direction)"
          }
        ]
      }
    ]
  }
}
```

---

## Alerting (AlertManager)

### Alert Rules

```yaml
# prometheus/alerts/critical.yml
groups:
  - name: critical_alerts
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service) 
          / sum(rate(http_requests_total[5m])) by (service) > 0.05
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate detected on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.service }}"

      # High API latency
      - alert: HighAPILatency
        expr: |
          histogram_quantile(0.95, 
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint)
          ) > 1
        for: 10m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "High API latency on {{ $labels.endpoint }}"
          description: "P95 latency is {{ $value }}s for {{ $labels.endpoint }}"

      # Market data pipeline down
      - alert: MarketDataPipelineDown
        expr: |
          rate(market_ticks_received_total[5m]) == 0
        for: 5m
        labels:
          severity: critical
          team: data-platform
        annotations:
          summary: "Market data pipeline is not receiving ticks"
          description: "No market ticks received in the last 5 minutes"

      # Database connection pool exhausted
      - alert: DatabaseConnectionPoolExhausted
        expr: |
          db_connection_pool_active / db_connection_pool_max > 0.9
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Database connection pool nearly exhausted"
          description: "Connection pool is {{ $value | humanizePercentage }} full"

      # High memory usage
      - alert: HighMemoryUsage
        expr: |
          (sum(container_memory_working_set_bytes) by (pod) 
          / sum(container_spec_memory_limit_bytes) by (pod)) > 0.9
        for: 5m
        labels:
          severity: warning
          team: devops
        annotations:
          summary: "High memory usage on {{ $labels.pod }}"
          description: "Memory usage is {{ $value | humanizePercentage }}"

      # Kafka consumer lag
      - alert: KafkaConsumerLag
        expr: |
          kafka_consumer_group_lag{topic="market-ticks"} > 10000
        for: 5m
        labels:
          severity: warning
          team: data-platform
        annotations:
          summary: "High Kafka consumer lag on {{ $labels.topic }}"
          description: "Consumer lag is {{ $value }} messages"

      # ML model accuracy drop
      - alert: MLModelAccuracyDrop
        expr: |
          ml_model_accuracy < 0.5
        for: 30m
        labels:
          severity: warning
          team: ml
        annotations:
          summary: "ML model accuracy dropped on {{ $labels.model }}"
          description: "Accuracy is {{ $value | humanizePercentage }} for {{ $labels.model }}"

      # WebSocket connection spike
      - alert: WebSocketConnectionSpike
        expr: |
          websocket_connections_active > 50000
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "High number of WebSocket connections"
          description: "{{ $value }} active connections (consider scaling)"
```

### AlertManager Configuration

```yaml
# alertmanager/config.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR_WEBHOOK_URL'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  routes:
    # Critical alerts to PagerDuty + Slack
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true
    
    - match:
        severity: critical
      receiver: 'slack-critical'
    
    # Warning alerts to Slack only
    - match:
        severity: warning
      receiver: 'slack-warning'

receivers:
  - name: 'default'
    slack_configs:
    - channel: '#alerts'
      title: '{{ .GroupLabels.alertname }}'
      text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'pagerduty-critical'
    pagerduty_configs:
    - service_key: 'YOUR_PAGERDUTY_KEY'
      description: '{{ .GroupLabels.alertname }}'

  - name: 'slack-critical'
    slack_configs:
    - channel: '#critical-alerts'
      color: 'danger'
      title: ':fire: CRITICAL: {{ .GroupLabels.alertname }}'
      text: |
        {{ range .Alerts }}
        *Alert:* {{ .Annotations.summary }}
        *Description:* {{ .Annotations.description }}
        *Severity:* {{ .Labels.severity }}
        *Service:* {{ .Labels.service }}
        {{ end }}

  - name: 'slack-warning'
    slack_configs:
    - channel: '#warnings'
      color: 'warning'
      title: ':warning: Warning: {{ .GroupLabels.alertname }}'
      text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

inhibit_rules:
  # Don't alert on dependent services if root cause is identified
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['cluster', 'service']
```

---

## Logging (ELK Stack / DataDog)

### Structured Logging (Python)

```python
# common/logging.py
import logging
import json
from datetime import datetime
from pythonjsonlogger import jsonlogger

class CustomJsonFormatter(jsonlogger.JsonFormatter):
    def add_fields(self, log_record, record, message_dict):
        super(CustomJsonFormatter, self).add_fields(log_record, record, message_dict)
        log_record['timestamp'] = datetime.utcnow().isoformat()
        log_record['level'] = record.levelname
        log_record['service'] = 'market-data-service'  # Set per service
        log_record['environment'] = 'production'

# Configure logger
logger = logging.getLogger()
logger.setLevel(logging.INFO)

handler = logging.StreamHandler()
formatter = CustomJsonFormatter('%(timestamp)s %(level)s %(service)s %(message)s')
handler.setFormatter(formatter)
logger.addHandler(handler)

# Usage
logger.info("Market tick received", extra={
    'symbol': 'RELIANCE',
    'price': 2450.50,
    'volume': 1000,
    'latency_ms': 45
})

logger.error("Upstox API error", extra={
    'error_code': 'RATE_LIMIT_EXCEEDED',
    'retry_after': 60
}, exc_info=True)
```

### Fluentd Configuration (Log Aggregation)

```yaml
# fluentd/fluent.conf
<source>
  @type tail
  path /var/log/containers/*.log
  pos_file /var/log/fluentd-containers.log.pos
  tag kubernetes.*
  read_from_head true
  <parse>
    @type json
    time_format %Y-%m-%dT%H:%M:%S.%NZ
  </parse>
</source>

# Filter: Add Kubernetes metadata
<filter kubernetes.**>
  @type kubernetes_metadata
  @id filter_kube_metadata
</filter>

# Filter: Parse application logs
<filter kubernetes.var.log.containers.market-data-service**>
  @type parser
  key_name log
  reserve_data true
  <parse>
    @type json
  </parse>
</filter>

# Output: Elasticsearch
<match kubernetes.**>
  @type elasticsearch
  @id out_es
  @log_level info
  include_tag_key true
  host "elasticsearch.logging.svc.cluster.local"
  port 9200
  logstash_format true
  logstash_prefix kubernetes
  <buffer>
    @type file
    path /var/log/fluentd-buffers/kubernetes.system.buffer
    flush_mode interval
    retry_type exponential_backoff
    flush_thread_count 2
    flush_interval 5s
    retry_forever false
    retry_max_interval 30
    chunk_limit_size 2M
    queue_limit_length 8
    overflow_action block
  </buffer>
</match>
```

### Kibana Queries (Common Searches)

```
# Find all errors in last hour
level:ERROR AND timestamp:[now-1h TO now]

# Market data latency > 100ms
service:"market-data-service" AND latency_ms:>100

# Upstox API errors
service:"market-data-service" AND error_code:*

# Slow database queries
service:* AND db_query_duration_ms:>1000

# Failed backtests
service:"backtest-service" AND status:"failed"

# ML prediction confidence < 60%
service:"ml-service" AND confidence:<0.6

# User login events
event_type:"login" AND timestamp:[now-24h TO now]

# WebSocket disconnections
event_type:"websocket_disconnect" AND reason:*
```

---

## Distributed Tracing (AWS X-Ray / Jaeger)

### X-Ray Integration (Python)

```python
# common/tracing.py
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware

# Configure X-Ray
xray_recorder.configure(
    service='market-data-service',
    sampling=True,
    context_missing='LOG_ERROR'
)

# FastAPI middleware
from fastapi import FastAPI
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.ext.fastapi.middleware import XRayMiddleware as FastAPIXRayMiddleware

app = FastAPI()
app.add_middleware(FastAPIXRayMiddleware, recorder=xray_recorder)

# Trace custom segments
@xray_recorder.capture('fetch_market_data')
async def fetch_market_data(symbol: str):
    # Add metadata
    xray_recorder.put_annotation('symbol', symbol)
    xray_recorder.put_metadata('timestamp', datetime.utcnow().isoformat())
    
    # Trace subsegments
    with xray_recorder.capture('upstox_api_call'):
        data = await upstox_client.get_tick(symbol)
    
    with xray_recorder.capture('cache_write'):
        await cache.set(f"tick:{symbol}", data)
    
    return data

# Trace database queries
@xray_recorder.capture('db_query')
async def get_user(user_id: str):
    xray_recorder.put_annotation('user_id', user_id)
    xray_recorder.put_annotation('query_type', 'SELECT')
    
    result = await db.execute(f"SELECT * FROM users WHERE id = '{user_id}'")
    return result
```

### Jaeger Configuration (Alternative)

```python
# common/tracing_jaeger.py
from opentelemetry import trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.resources import SERVICE_NAME, Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor

# Configure Jaeger
resource = Resource(attributes={
    SERVICE_NAME: "market-data-service"
})

jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger-agent.tracing.svc.cluster.local",
    agent_port=6831,
)

provider = TracerProvider(resource=resource)
processor = BatchSpanProcessor(jaeger_exporter)
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

# Auto-instrument FastAPI
FastAPIInstrumentor.instrument_app(app)

# Auto-instrument SQLAlchemy
SQLAlchemyInstrumentor().instrument()

# Auto-instrument Redis
RedisInstrumentor().instrument()

# Get tracer
tracer = trace.get_tracer(__name__)

# Manual instrumentation
with tracer.start_as_current_span("strategy_calculation") as span:
    span.set_attribute("strategy", "bollinger_bands")
    span.set_attribute("symbol", "RELIANCE")
    
    result = calculate_strategy()
    
    span.set_attribute("signal_generated", result.action)
    span.add_event("Signal generated", {"confidence": result.confidence})
```

---

## SLO/SLI Definitions

### Service Level Indicators (SLIs)

```yaml
# SLI Definitions
slis:
  # Availability
  - name: api_availability
    description: "Percentage of successful API requests"
    query: |
      sum(rate(http_requests_total{status!~"5.."}[5m])) 
      / sum(rate(http_requests_total[5m]))
    target: 0.999  # 99.9%
  
  # Latency
  - name: api_latency_p95
    description: "95th percentile API latency"
    query: |
      histogram_quantile(0.95, 
        sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
      )
    target: 0.1  # 100ms
  
  # Market Data Freshness
  - name: market_data_freshness
    description: "Time since last market tick received"
    query: |
      time() - max(market_ticks_last_received_timestamp)
    target: 5  # 5 seconds max staleness
  
  # ML Inference Latency
  - name: ml_inference_latency_p99
    description: "99th percentile ML inference latency"
    query: |
      histogram_quantile(0.99,
        sum(rate(ml_inference_duration_seconds_bucket[5m])) by (le)
      )
    target: 0.1  # 100ms
```

### Service Level Objectives (SLOs)

```yaml
# 30-day rolling window SLOs
slos:
  - name: API Availability
    sli: api_availability
    target: 99.95%
    window: 30d
    
  - name: API Latency
    sli: api_latency_p95
    target: 95% of requests < 100ms
    window: 30d
  
  - name: Market Data Pipeline
    sli: market_data_freshness
    target: 99.9% of time < 5s staleness
    window: 30d
  
  - name: ML Inference
    sli: ml_inference_latency_p99
    target: 99% of predictions < 100ms
    window: 30d

# Error Budget Calculation
# If SLO = 99.95% over 30 days:
# Allowed downtime = 30 days * (1 - 0.9995) = 21.6 minutes/month
```

---

## On-Call Runbooks

### Runbook 1: High API Error Rate

```markdown
# Runbook: High API Error Rate

## Symptoms
- Alert: `HighErrorRate`
- Error rate > 5% for 5 minutes

## Possible Causes
1. Upstox API rate limiting
2. Database connection pool exhausted
3. Dragonfly cache unavailable
4. Code bug in recent deployment

## Investigation Steps

1. Check recent deployments:
   ```bash
   kubectl rollout history deployment/market-data-service -n trading-platform
   ```

2. Check error logs:
   ```bash
   kubectl logs -l app=market-data-service --tail=100 | grep ERROR
   ```

3. Check Upstox API status:
   ```bash
   curl https://api.upstox.com/health
   ```

4. Check database connections:
   ```sql
   SELECT count(*) FROM pg_stat_activity WHERE state = 'active';
   ```

5. Check cache availability:
   ```bash
   redis-cli -h dragonfly.svc.cluster.local ping
   ```

## Remediation

### If Upstox API rate limit:
- Increase caching TTL temporarily
- Reduce polling frequency
- Switch to lower-priority symbols

### If database connection pool:
- Scale up connection pool size
- Identify and kill long-running queries
- Add read replicas

### If recent deployment issue:
```bash
# Rollback to previous version
kubectl rollout undo deployment/market-data-service -n trading-platform
```

### If cache unavailable:
- Restart Dragonfly pods
- Fallback to database queries (slower but functional)

## Prevention
- Add more comprehensive integration tests
- Implement circuit breakers for all external APIs
- Add request retries with exponential backoff
```

### Runbook 2: Market Data Pipeline Down

```markdown
# Runbook: Market Data Pipeline Down

## Symptoms
- Alert: `MarketDataPipelineDown`
- No market ticks received for 5 minutes

## Investigation Steps

1. Check market-data-service logs:
   ```bash
   kubectl logs -l app=market-data-service --tail=50
   ```

2. Check WebSocket connection status:
   ```bash
   kubectl exec -it market-data-service-xxx -- curl localhost:8000/health
   ```

3. Check Kafka topic lag:
   ```bash
   kafka-consumer-groups --bootstrap-server kafka:9092 --describe --group market-data-consumer
   ```

4. Check Upstox API status:
   ```bash
   curl https://api.upstox.com/status
   ```

## Remediation

### If WebSocket disconnected:
- Automatic reconnection should kick in within 30s
- If not, restart market-data-service pods:
  ```bash
  kubectl rollout restart deployment/market-data-service
  ```

### If Kafka brokers down:
- Check Kafka pod status:
  ```bash
  kubectl get pods -l app=kafka
  ```
- Restart Kafka if needed (has data persistence)

### If Upstox API down:
- Switch to REST API fallback mode
- Monitor Upstox status page
- Consider alternative data source if prolonged

## Prevention
- Implement redundant data sources
- Add dead-letter queue for failed messages
- Improve connection retry logic
```

---

This comprehensive monitoring and observability setup ensures production-grade reliability with proactive alerting, detailed logging, and distributed tracing for debugging complex issues across microservices.
