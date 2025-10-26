# Complete Guide: Coralogix TAM Technical Assignment

Great! You have your EKS cluster ready. Let me break down **all the steps** to complete this assignment systematically.

---

## 📋 **Overview of Tasks**

1. ✅ Login to Coralogix
2. 🔧 Deploy OpenTelemetry Demo to EKS
3. 📊 Connect Data (Logs, Metrics, Traces)
4. 🔍 Parse Data with Regex Rules
5. 🌍 Enrich IP Addresses
6. 📈 Create Events2Metrics
7. 📊 Create Custom Dashboards
8. 🚨 Create 3 Alerts
9. 📤 Submit Results

---

## **PHASE 1: Setup & Login**

### Step 1: Login to Coralogix Account

```bash
# Your account URL format
https://cs-task-<YOUR_NAME>.coralogix.com/
```

**Actions:**
1. Login with credentials provided by Coralogix
2. Navigate to **Settings → Send Your Data** to get your API key
3. Note your Coralogix domain (for US: `coralogix.com`, EU: `coralogix.com`, etc.)

---

## **PHASE 2: Deploy OpenTelemetry Demo to EKS**

### Step 2: Prepare Your EKS Cluster

First, make sure your cluster is running:

```bash
# Verify cluster is ready
kubectl get nodes

# Should show 2-3 nodes in Ready state
```

### Step 3: Install OpenTelemetry Demo on Kubernetes

The assignment references the **OTel Demo** - this is the official OpenTelemetry Demo Application.

**Official Links:**
- **OTel Demo GitHub:** https://github.com/open-telemetry/opentelemetry-demo
- **Kubernetes Deployment:** https://opentelemetry.io/docs/demo/kubernetes-deployment/
- **Coralogix K8s Blog:** https://coralogix.com/blog/opentelemetry-demo-application-with-kubernetes/

#### A. Add Helm Repository

```bash
# Add OTel Demo Helm repository
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
```

#### B. Create Namespace

```bash
# Create namespace for the demo
kubectl create namespace otel-demo
```

#### C. Create Coralogix Configuration File

Create a file called `coralogix-values.yaml`:

```yaml
# coralogix-values.yaml
default:
  env:
    - name: OTEL_SERVICE_NAME
      value: "otel-demo"
    - name: OTEL_EXPORTER_OTLP_ENDPOINT
      value: "https://ingress.coralogix.com:443"  # Change based on your region
    - name: OTEL_EXPORTER_OTLP_HEADERS
      value: "Authorization=Bearer YOUR_CORALOGIX_SEND_DATA_API_KEY"

# Enable all demo components
components:
  frontend: 
    enabled: true
  productCatalogService:
    enabled: true
  cartService:
    enabled: true
  checkoutService:
    enabled: true
  currencyService:
    enabled: true
  emailService:
    enabled: true
  paymentService:
    enabled: true
  recommendationService:
    enabled: true
  shippingService:
    enabled: true
  adService:
    enabled: true
  loadGenerator:
    enabled: true

# OpenTelemetry Collector configuration
opentelemetry-collector:
  enabled: true
  mode: deployment
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    
    processors:
      batch:
        timeout: 10s
        send_batch_size: 1024
      
      memory_limiter:
        limit_mib: 512
        spike_limit_mib: 128
      
      # Add Kubernetes attributes
      k8sattributes:
        auth_type: "serviceAccount"
        passthrough: false
        extract:
          metadata:
            - k8s.namespace.name
            - k8s.deployment.name
            - k8s.pod.name
            - k8s.pod.uid
            - k8s.node.name
            - k8s.container.name
        pod_association:
          - sources:
              - from: resource_attribute
                name: k8s.pod.ip
          - sources:
              - from: resource_attribute
                name: k8s.pod.uid
          - sources:
              - from: connection
    
    exporters:
      coralogix:
        domain: "coralogix.com"  # Change to your region
        private_key: "YOUR_CORALOGIX_SEND_DATA_API_KEY"
        application_name: "otel-demo"
        subsystem_name: "kubernetes"
        timeout: 30s
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, k8sattributes, batch]
          exporters: [coralogix]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, k8sattributes, batch]
          exporters: [coralogix]
        logs:
          receivers: [otlp]
          processors: [memory_limiter, k8sattributes, batch]
          exporters: [coralogix]

# For infrastructure metrics, enable host metrics
hostMetrics:
  enabled: true
```

**Important Configuration Updates:**

Replace these values:
- `YOUR_CORALOGIX_SEND_DATA_API_KEY` - Get from Coralogix UI (Settings → Send Your Data)
- `domain: "coralogix.com"` - Change based on your region:
  - **US**: `coralogix.com`
  - **EU**: `coralogix.com`
  - **India**: `coralogix.in`
  - **Singapore**: `coralogixsg.com`

#### D. Install OpenTelemetry Demo

```bash
# Install the demo using Helm
helm install my-otel-demo open-telemetry/opentelemetry-demo \
  --namespace otel-demo \
  --values coralogix-values.yaml
```

#### E. Verify Installation

```bash
# Check if all pods are running
kubectl get pods -n otel-demo

# Watch until all pods are Running
kubectl get pods -n otel-demo -w

# Check services
kubectl get svc -n otel-demo
```

---

### Step 4: Install Kubernetes Infrastructure Monitoring

To ship **infrastructure metrics** and **K8s metadata**, install Coralogix's Kubernetes integration:

```bash
# Add Coralogix Helm repo
helm repo add coralogix https://cgx.jfrog.io/artifactory/coralogix-charts-virtual
helm repo update

# Create values file for Coralogix integration
cat > coralogix-integration-values.yaml <<EOF
global:
  domain: "coralogix.com"  # Change to your region
  clusterName: "coralogix"  # Your EKS cluster name

secret:
  data:
    apiKey: "YOUR_CORALOGIX_SEND_DATA_API_KEY"

opentelemetry-agent:
  enabled: true
  mode: daemonset

opentelemetry-cluster-collector:
  enabled: true
  mode: deployment
EOF

# Install Coralogix integration
helm install coralogix-integration coralogix/coralogix \
  --namespace coralogix \
  --create-namespace \
  --values coralogix-integration-values.yaml
```

**Official Documentation:**
- **Coralogix K8s Integration:** https://coralogix.com/docs/kubernetes-dashboard/

---

## **PHASE 3: Verify Data is Flowing**

### Step 5: Check Data in Coralogix

1. **Login to Coralogix:** `https://cs-task-<YOUR_NAME>.coralogix.com/`

2. **Check Logs:**
   - Navigate to **Explore → Logs**
   - Filter by `application:otel-demo`
   - You should see logs from various services

3. **Check Traces:**
   - Navigate to **Explore → Tracing (APM)**
   - You should see distributed traces

4. **Check Metrics:**
   - Navigate to **Explore → Metrics**
   - Search for metrics like `http_server_duration`

---

## **PHASE 4: Parse Data with Regex Rules**

### Step 6: Create Parsing Rules

**Goal:** Extract valuable fields from logs (e.g., HTTP status codes, response times, IP addresses)

**Links:**
- **Coralogix Parsing Rules:** https://coralogix.com/docs/log-parsing-rules/
- **Regex Tester:** https://regex101.com/

#### Example Parsing Rules to Create:

1. **Navigate to:** Data Flow → Parsing Rules → Create Rule

2. **Rule 1: Extract HTTP Status Code**
```yaml
Rule Name: Extract HTTP Status Code
Source Field: text
Destination Field: http.status_code
Regex: "status":\s*(\d{3})
# This extracts: 200, 404, 500, etc.
```

3. **Rule 2: Extract Response Time**
```yaml
Rule Name: Extract Response Time
Source Field: text
Destination Field: http.response_time_ms
Regex: "duration":\s*(\d+\.?\d*)
# This extracts: 123.45, 500, etc.
```

4. **Rule 3: Extract IP Address**
```yaml
Rule Name: Extract Client IP
Source Field: text
Destination Field: client.ip
Regex: "ip":\s*"((?:\d{1,3}\.){3}\d{1,3})"
# This extracts: 192.168.1.1, 10.0.0.1, etc.
```

5. **Rule 4: Extract HTTP Method**
```yaml
Rule Name: Extract HTTP Method
Source Field: text
Destination Field: http.method
Regex: "method":\s*"(GET|POST|PUT|DELETE|PATCH)"
```

**Steps in Coralogix UI:**
1. Go to **Data Flow → Rules**
2. Click **Create Rule Group**
3. Select **Parse** as rule type
4. Add your regex pattern
5. Test with sample logs
6. Save and apply

---

## **PHASE 5: Enrich IP Addresses**

### Step 7: Add GeoIP and Security Enrichment

**Links:**
- **Coralogix Enrichment:** https://coralogix.com/docs/enrichment/

1. Navigate to **Data Flow → Enrichment**

2. **Create GeoIP Enrichment:**
   - Rule Type: **GeoIP**
   - Source Field: `client.ip` (from parsing rule)
   - This adds fields like:
     - `client.geo.country`
     - `client.geo.city`
     - `client.geo.location`

3. **Create Security Enrichment:**
   - Rule Type: **Security**
   - Source Field: `client.ip`
   - This adds threat intelligence data

---

## **PHASE 6: Create Events2Metrics**

### Step 8: Create Events2Metrics

**Goal:** Convert log events into metrics for long-term analysis

**Links:**
- **Events2Metrics Docs:** https://coralogix.com/docs/events2metrics/

#### Example Events2Metrics:

1. **Navigate to:** Data Flow → Events2Metrics

2. **Metric 1: HTTP Request Rate by Status Code**
```yaml
Name: http_requests_total
Type: Counter
Labels:
  - http.status_code
  - http.method
  - service.name
Query: text contains "http"
```

3. **Metric 2: Response Time Histogram**
```yaml
Name: http_response_duration_seconds
Type: Histogram
Labels:
  - http.method
  - service.name
Field to Aggregate: http.response_time_ms
Query: http.response_time_ms exists
```

---

## **PHASE 7: Create Custom Dashboards**

### Step 9: Create Full Stack Observability Dashboard

**Links:**
- **Custom Dashboards:** https://coralogix.com/docs/custom-dashboards/

1. **Navigate to:** Dashboards → Create New Dashboard

2. **Dashboard Name:** "OTel Demo - Full Stack Observability"

#### **Required Visualizations (Minimum 3):**

**Widget 1: Request Rate Over Time (Logs)**
- Type: Line Chart
- Query: PromQL or DataPrime
- Shows: HTTP requests per second

**Widget 2: Service Map (Traces)**
- Type: Service Map
- Shows: Service dependencies and latency

**Widget 3: Error Rate by Service (Logs)**
- Type: Bar Chart
- Query: Filter for error status codes (4xx, 5xx)
- Group by: service.name

**Widget 4: Infrastructure Metrics (Metrics)**
- Type: Gauge or Line Chart
- Shows: CPU, Memory usage of pods

**Widget 5: Geographic Distribution (Enriched Logs)**
- Type: Map Widget
- Shows: Requests by country (from GeoIP enrichment)

**Widget 6: Events2Metrics Visualization (PromQL)**
- Type: Graph
- Query: 
```promql
rate(http_requests_total[5m])
```
- Shows: Request rate from Events2Metrics

---

## **PHASE 8: Create Alerts**

### Step 10: Create 3 Required Alerts

**Links:**
- **Alerts Documentation:** https://coralogix.com/docs/alerts/

#### **Alert 1: Ratio of Faulty Requests**

```yaml
Name: High Error Rate Alert
Type: Ratio (Metric or Logs)
Condition: 
  - Numerator: Count of logs where http.status_code >= 400
  - Denominator: Count of all HTTP logs
  - Threshold: > 5%
Timeframe: Last 5 minutes
Notification: Email/Slack
```

**In Coralogix UI:**
1. Navigate to **Alerts → New Alert**
2. Select **Ratio** alert type
3. Configure:
   - Query A: `http.status_code >= 400`
   - Query B: `http.status_code exists`
   - Alert when: `(A/B) > 0.05`

#### **Alert 2: Error Text Alert**

```yaml
Name: Error Message Alert
Type: New Value / Immediate
Query: text contains "Error" OR text contains "error"
Condition: Trigger immediately
Notification: Email/Slack
```

#### **Alert 3: Metric-Based Alert (Events2Metrics)**

```yaml
Name: High Response Time Alert
Type: Metric (PromQL)
Query: 
  avg(http_response_duration_seconds) > 1.0
Condition: Average response time > 1 second
Timeframe: Last 5 minutes
Notification: Email/Slack
```

**PromQL Example:**
```promql
avg(http_response_duration_seconds{service_name="frontend"}) > 1.0
```

---

## **PHASE 9: Documentation & Submission**

### Step 11: Prepare Submission Materials

Create a document with the following:

#### **1. Environment Setup Details**

```markdown
## Environment Setup

### Infrastructure
- **Cloud Provider:** AWS
- **Region:** us-east-1
- **Kubernetes:** EKS 1.28
- **Node Type:** t3.large
- **Node Count:** 2-3 nodes

### Installation Commands
```bash
# EKS Cluster creation
eksctl create cluster -f cluster.yaml

# OTel Demo installation
helm install my-otel-demo open-telemetry/opentelemetry-demo \
  --namespace otel-demo \
  --values coralogix-values.yaml

# Coralogix integration
helm install coralogix-integration coralogix/coralogix \
  --namespace coralogix \
  --values coralogix-integration-values.yaml
```
```

#### **2. Configuration Files**

Include:
- `cluster.yaml`
- `coralogix-values.yaml`
- `coralogix-integration-values.yaml`

#### **3. Links to Coralogix Resources**

```markdown
## Coralogix Resources

### Parsing Rules
- [Link to Parsing Rule 1]: https://cs-task-<YOUR_NAME>.coralogix.com/rules/...
- [Link to Parsing Rule 2]: https://cs-task-<YOUR_NAME>.coralogix.com/rules/...

### Events2Metrics
- [Link to E2M]: https://cs-task-<YOUR_NAME>.coralogix.com/events2metrics/...

### Custom Dashboards
- [Full Stack Dashboard]: https://cs-task-<YOUR_NAME>.coralogix.com/dashboards/...

### Alerts
- [Alert 1 - Error Ratio]: https://cs-task-<YOUR_NAME>.coralogix.com/alerts/...
- [Alert 2 - Error Text]: https://cs-task-<YOUR_NAME>.coralogix.com/alerts/...
- [Alert 3 - Metric Alert]: https://cs-task-<YOUR_NAME>.coralogix.com/alerts/...
```

#### **4. Screenshots**

Take screenshots of:
- Dashboard with visualizations
- Parsing rules in action
- Enriched logs showing GeoIP data
- Events2Metrics configuration
- Alert configurations
- Data flowing (logs, traces, metrics)

---

## **Quick Reference: Essential Links**

### **Official Documentation**
- **OpenTelemetry Demo:** https://github.com/open-telemetry/opentelemetry-demo
- **Coralogix Docs:** https://coralogix.com/docs/
- **K8s Dashboard:** https://coralogix.com/docs/kubernetes-dashboard/
- **Parsing Rules:** https://coralogix.com/docs/log-parsing-rules/
- **Events2Metrics:** https://coralogix.com/docs/events2metrics/
- **Custom Dashboards:** https://coralogix.com/docs/custom-dashboards/
- **Alerts:** https://coralogix.com/docs/alerts/

### **Helpful Tools**
- **Regex Tester:** https://regex101.com/
- **PromQL Docs:** https://prometheus.io/docs/prometheus/latest/querying/basics/
- **OTel Collector Config:** https://opentelemetry.io/docs/collector/configuration/

---

## **Troubleshooting Tips**

### **Issue: Pods not starting**
```bash
# Check pod status
kubectl describe pod <pod-name> -n otel-demo

# Check logs
kubectl logs <pod-name> -n otel-demo
```

### **Issue: No data in Coralogix**
1. Verify API key is correct
2. Check Coralogix domain matches your region
3. Verify network connectivity from EKS to Coralogix
4. Check OTel Collector logs:
```bash
kubectl logs -n otel-demo -l app=opentelemetry-collector
```

### **Issue: K8s metadata missing**
- Ensure `k8sattributes` processor is configured
- Verify service account has proper RBAC permissions

---

## **Timeline Recommendation**

- **Day 1:** Setup EKS, deploy OTel Demo (2-3 hours)
- **Day 2:** Create parsing rules, enrichment (2 hours)
- **Day 3:** Events2Metrics, dashboards (2-3 hours)
- **Day 4:** Alerts, documentation (2 hours)
- **Day 5:** Review, screenshots, submit

---

**Remember:** The assignment says to reach out with questions, blockers, and clarifications. Don't hesitate to contact them if you get stuck!

Good luck! 🚀
