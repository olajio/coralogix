# OpenTelemetry Collector Configuration - Complete Guide

Let me explain OTel config files from scratch, then show you what's wrong with yours.

## 🎯 The Big Picture: 3 Main Components

Every OTel config has **3 building blocks** that work like a factory assembly line:

```
┌──────────┐     ┌────────────┐     ┌──────────┐
│RECEIVERS │ --> │ PROCESSORS │ --> │ EXPORTERS│
└──────────┘     └────────────┘     └──────────┘
   (Input)         (Transform)         (Output)
```

1. **RECEIVERS**: Where data comes IN (like a door)
2. **PROCESSORS**: How to modify/filter data (like a conveyor belt)
3. **EXPORTERS**: Where data goes OUT (like shipping)

Then you connect them with **PIPELINES**.

---

## 📝 Complete Coralogix Example (With Explanations)

```yaml
# ============================================
# RECEIVERS: Where telemetry data comes FROM
# ============================================
receivers:
  # OTLP receiver - accepts data in OpenTelemetry format
  otlp:
    protocols:
      grpc:                    # Listens on port 4317 (default)
        endpoint: 0.0.0.0:4317 # Accept from any IP
      http:                    # Listens on port 4318 (default)
        endpoint: 0.0.0.0:4318
  
  # Collects metrics from the host machine itself
  hostmetrics:
    collection_interval: 30s   # Collect every 30 seconds
    scrapers:                  # What metrics to collect
      cpu:                     # CPU usage
        metrics:
          system.cpu.utilization:
            enabled: true
      memory:                  # Memory usage
        metrics:
          system.memory.utilization:
            enabled: true
      disk:                    # Disk I/O
      network:                 # Network traffic
  
  # Collects logs from files
  filelog:
    include:
      - /var/log/app/*.log     # Which log files to read
    start_at: beginning        # Read from start of file
    include_file_path: true    # Add filename to log metadata

# ============================================
# PROCESSORS: How to TRANSFORM the data
# ============================================
processors:
  # Memory limiter - prevents OTel from using too much RAM
  memory_limiter:
    check_interval: 1s         # Check memory usage every second
    limit_mib: 512             # Max 512 MB of memory
    spike_limit_mib: 128       # Allow 128 MB spikes
  
  # Batch processor - groups data before sending (VERY IMPORTANT!)
  batch:
    timeout: 10s               # Send batch every 10 seconds
    send_batch_size: 1024      # Or when 1024 items collected
    send_batch_max_size: 2048  # Never exceed 2048 items
  
  # Resource detection - adds cloud/host metadata automatically
  resourcedetection:
    detectors: [env, system, ec2, eks]  # Auto-detect AWS info
    timeout: 5s
  
  # K8s attributes - adds Kubernetes metadata to telemetry
  k8sattributes:
    extract:
      metadata:
        - k8s.namespace.name
        - k8s.pod.name
        - k8s.node.name
        - k8s.deployment.name
    passthrough: false

# ============================================
# EXPORTERS: Where to SEND the data
# ============================================
exporters:
  # Coralogix exporter - sends data to Coralogix platform
  coralogix:
    # Your Coralogix domain (changes by region)
    domain: "coralogix.com"              # US: coralogix.com
                                         # EU: coralogix.com  
                                         # IN: coralogix.in
                                         # SG: coralogixsg.com
    
    # Your private API key (KEEP SECRET!)
    private_key: "${CORALOGIX_PRIVATE_KEY}"  # Use env variable
    
    # Application name (groups related services)
    application_name: "my-app"
    
    # Subsystem name (specific component)
    subsystem_name: "backend-api"
    
    # Timeout for sending data
    timeout: 30s
    
    # Configure which data types to send
    traces:
      endpoint: "https://ingress.coralogix.com:443"
    metrics:
      endpoint: "https://ingress.coralogix.com:443"
    logs:
      endpoint: "https://ingress.coralogix.com:443"
  
  # Logging exporter - prints to console (for debugging)
  logging:
    loglevel: info             # Options: debug, info, warn, error
    sampling_initial: 5        # Only print first 5 messages
    sampling_thereafter: 200   # Then 1 out of every 200

# ============================================
# SERVICE: Connect everything with PIPELINES
# ============================================
service:
  # Configure health check and monitoring
  telemetry:
    logs:
      level: info              # OTel's own log level
    metrics:
      address: 0.0.0.0:8888    # Prometheus metrics about OTel itself
  
  # Define pipelines - connect receivers → processors → exporters
  pipelines:
    # TRACES pipeline (distributed tracing data)
    traces:
      receivers: [otlp]                              # Accept OTLP traces
      processors: [memory_limiter, resourcedetection, k8sattributes, batch]  # Process in order
      exporters: [coralogix, logging]                # Send to Coralogix + console
    
    # METRICS pipeline (performance metrics)
    metrics:
      receivers: [otlp, hostmetrics]                 # Accept OTLP + host metrics
      processors: [memory_limiter, resourcedetection, batch]
      exporters: [coralogix, logging]
    
    # LOGS pipeline (application logs)
    logs:
      receivers: [otlp, filelog]                     # Accept OTLP + file logs
      processors: [memory_limiter, resourcedetection, batch]
      exporters: [coralogix, logging]
```

---

## 🔍 Understanding Each Section

### 1. **RECEIVERS** (Data Sources)

Think of receivers as **input ports**. Common types:

| Receiver | What It Does | Port |
|----------|-------------|------|
| `otlp` | Accepts OpenTelemetry data | 4317 (gRPC), 4318 (HTTP) |
| `hostmetrics` | Collects CPU, memory, disk stats | N/A |
| `filelog` | Reads log files | N/A |
| `prometheus` | Scrapes Prometheus metrics | Configurable |
| `jaeger` | Accepts Jaeger traces | 14250 |

### 2. **PROCESSORS** (Data Transformation)

Processors modify data as it flows through. Order matters!

**Recommended Order:**
```
memory_limiter → resourcedetection → k8sattributes → batch
```

| Processor | Purpose | Why Important |
|-----------|---------|---------------|
| `memory_limiter` | Prevents OOM crashes | Should be FIRST |
| `resourcedetection` | Adds cloud metadata | Context enrichment |
| `k8sattributes` | Adds K8s labels/names | For filtering in Coralogix |
| `batch` | Groups data for efficiency | Should be LAST, saves network calls |

### 3. **EXPORTERS** (Data Destinations)

Where processed data goes:

| Exporter | Sends To | Use Case |
|----------|----------|----------|
| `coralogix` | Coralogix platform | Production |
| `logging` | Console/stdout | Development/debugging |
| `prometheus` | Prometheus server | Metrics only |
| `otlp` | Another OTel collector | Collector chaining |

### 4. **PIPELINES** (Connecting It All)

Pipelines define the **flow** for each data type:

```yaml
service:
  pipelines:
    traces:              # Pipeline name (traces, metrics, or logs)
      receivers: [otlp]  # Where data comes from
      processors: [batch] # How to process it
      exporters: [coralogix] # Where to send it
```

---

## ❌ What's Wrong With Your Config

Here's your config with problems highlighted:

```yaml
receivers:
  otlp:
    protocols:
      grpc:              # ❌ PROBLEM 1: Missing configuration

processors:
  batch:                 # ❌ PROBLEM 2: Missing configuration

exporters:
  logging:
    loglevel: debug      # ⚠️ OK but will be very noisy

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [logging]
      # ❌ PROBLEM 3: Only handles traces, no logs/metrics
```

### 🔧 Issues & Fixes

#### **Problem 1: Incomplete Receiver**
```yaml
# ❌ WRONG - grpc has no config
receivers:
  otlp:
    protocols:
      grpc:

# ✅ CORRECT - Either add endpoint OR empty braces
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317  # Specify endpoint
      # OR just use empty braces
      grpc: {}                   # Use defaults
```

#### **Problem 2: Incomplete Processor**
```yaml
# ❌ WRONG - batch has no config
processors:
  batch:

# ✅ CORRECT - Add configuration OR empty braces
processors:
  batch:
    timeout: 10s
    send_batch_size: 1024
  # OR
  batch: {}  # Use defaults
```

#### **Problem 3: Missing Memory Limiter**
```yaml
# ⚠️ YOUR CONFIG - Will crash if too much data
processors:
  batch: {}

# ✅ BETTER - Add memory protection
processors:
  memory_limiter:
    limit_mib: 512
    spike_limit_mib: 128
  batch: {}
```

#### **Problem 4: Not Sending to Coralogix**
```yaml
# ⚠️ YOUR CONFIG - Just logging to console
exporters:
  logging:
    loglevel: debug

# ✅ FOR PRODUCTION - Send to Coralogix
exporters:
  coralogix:
    domain: "coralogix.com"
    private_key: "${CORALOGIX_PRIVATE_KEY}"
    application_name: "my-app"
    subsystem_name: "component-name"
  logging:  # Keep for debugging
    loglevel: info
```

#### **Problem 5: Only Traces Pipeline**
```yaml
# ⚠️ YOUR CONFIG - Missing logs & metrics
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [logging]

# ✅ COMPLETE - All three types
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [coralogix]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [coralogix]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [coralogix]
```

---

## ✅ Fixed Version of Your Config

Here's your config corrected for basic Coralogix usage:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317  # ✅ FIXED: Added endpoint
      http:
        endpoint: 0.0.0.0:4318  # ✅ ADDED: HTTP protocol

processors:
  memory_limiter:                # ✅ ADDED: Prevent OOM
    limit_mib: 512
    spike_limit_mib: 128
  batch:                         # ✅ FIXED: Added config
    timeout: 10s
    send_batch_size: 1024

exporters:
  coralogix:                     # ✅ ADDED: Send to Coralogix
    domain: "coralogix.com"
    private_key: "${CORALOGIX_PRIVATE_KEY}"
    application_name: "test-app"
    subsystem_name: "otel-collector"
    timeout: 30s
  logging:
    loglevel: info               # ✅ CHANGED: Less noise

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]  # ✅ Added memory_limiter
      exporters: [coralogix, logging]      # ✅ Send to both
    
    metrics:                     # ✅ ADDED: Metrics pipeline
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [coralogix, logging]
    
    logs:                        # ✅ ADDED: Logs pipeline
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [coralogix, logging]
```

---

## 🎓 Key Takeaways

1. **Always use `{}`** or proper config for receivers/processors/exporters
2. **Memory limiter should be first** in processor list
3. **Batch processor should be last** in processor list
4. **All three pipelines** (traces, metrics, logs) for complete observability
5. **Use environment variables** for secrets (`${CORALOGIX_PRIVATE_KEY}`)
6. **Logging exporter** is for debugging only - use Coralogix for production

Need help with any specific part? Let me know!
