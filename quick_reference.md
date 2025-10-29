# Quick Reference - OpenTelemetry Demo → Coralogix

## 🚀 Quick Start (Copy-Paste Ready)

### Step 1: Edit Configuration
```bash
# Download the files I created for you
# Edit coralogix-override.yaml and replace:
# 1. YOUR_CORALOGIX_SEND_DATA_API_KEY_HERE → Your actual API key
# 2. domain: "coralogix.com" → Your region's domain
# 3. Update the endpoint URLs to match your region
```

### Step 2: Get Your Coralogix Details
```
Login: https://cs-task-<YOUR_NAME>.coralogix.com/
Go to: Settings → Send Your Data
Copy: Send Your Data API Key
Note: Your region domain
```

### Step 3: Apply the Configuration
```bash
# Make sure you're in the right directory
cd ~/coralogix_tam_task  # or wherever you saved the files

# Upgrade your deployment
helm upgrade my-otel-demo open-telemetry/opentelemetry-demo \
  --namespace otel-demo \
  --values coralogix-override.yaml \
  --reuse-values

# Wait for pods to restart (2-3 minutes)
kubectl get pods -n otel-demo -w
```

### Step 4: Verify Everything Works
```bash
# Check collector is running
kubectl get pods -n otel-demo -l app.kubernetes.io/name=opentelemetry-collector

# Check collector logs (look for "Everything is ready")
kubectl logs -n otel-demo -l app.kubernetes.io/name=opentelemetry-collector --tail=50 | grep -i "ready\|error\|coralogix"

# Wait 2-3 minutes, then check Coralogix UI
# Logs: Explore → Logs → filter by application_name:otel-demo
# Traces: Explore → Tracing
# Metrics: Explore → Metrics
```

---

## 🔧 Most Common Issues & Fixes

### "No data in Coralogix"
```bash
# 1. Check API key is correct
kubectl get secret -n otel-demo

# 2. Check collector logs for auth errors
kubectl logs -n otel-demo -l app.kubernetes.io/name=opentelemetry-collector | grep -i "error\|failed\|auth"

# 3. Restart collector
kubectl rollout restart deployment/otel-collector -n otel-demo
```

### "Configuration error after upgrade"
```bash
# View the actual config being used
kubectl get configmap -n otel-demo -o yaml | grep -A 20 "coralogix"

# If config looks wrong, rollback
helm rollback my-otel-demo -n otel-demo
```

### "Pods won't start"
```bash
# See what's wrong
kubectl describe pod -n otel-demo -l app.kubernetes.io/name=opentelemetry-collector

# Check events
kubectl get events -n otel-demo --sort-by='.lastTimestamp'
```

---

## 📍 Coralogix Endpoints by Region

### US Region
```yaml
domain: "coralogix.com"
traces:
  endpoint: "https://ingress.coralogix.com:443"
metrics:
  endpoint: "https://ingress.coralogix.com:443"
logs:
  endpoint: "https://ingress.coralogix.com:443"
```

### EU Region  
```yaml
domain: "coralogix.com"
traces:
  endpoint: "https://ingress.coralogix.com:443"
metrics:
  endpoint: "https://ingress.coralogix.com:443"
logs:
  endpoint: "https://ingress.coralogix.com:443"
```

### India Region
```yaml
domain: "coralogix.in"
traces:
  endpoint: "https://ingress.coralogix.in:443"
metrics:
  endpoint: "https://ingress.coralogix.in:443"
logs:
  endpoint: "https://ingress.coralogix.in:443"
```

### Singapore Region
```yaml
domain: "coralogixsg.com"
traces:
  endpoint: "https://ingress.coralogixsg.com:443"
metrics:
  endpoint: "https://ingress.coralogixsg.com:443"
logs:
  endpoint: "https://ingress.coralogixsg.com:443"
```

---

## 🎯 What to Verify in Coralogix UI

### Logs (Explore → Logs)
```
Filter: application_name:otel-demo
Look for fields:
  ✓ k8s.namespace.name
  ✓ k8s.pod.name
  ✓ k8s.deployment.name
  ✓ k8s.node.name
  ✓ service.name
  ✓ Text field with actual log content
```

### Traces (Explore → Tracing)
```
Should see:
  ✓ Service map with multiple services
  ✓ Traces with multiple spans
  ✓ Can click through service → traces → spans
  ✓ Kubernetes metadata on each span
```

### Metrics (Explore → Metrics)
```
Search for:
  ✓ http_server_duration (application metrics)
  ✓ system_cpu_utilization (infrastructure)
  ✓ system_memory_utilization (infrastructure)
  ✓ postgresql_* (database metrics)
  ✓ http_* (HTTP metrics)
```

---

## 📊 Demo Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    OTel Demo Apps                       │
│  Frontend │ Cart │ Checkout │ Payment │ Shipping │ ... │
└────────────────────┬────────────────────────────────────┘
                     │ OTLP Protocol
                     ↓
┌─────────────────────────────────────────────────────────┐
│              OpenTelemetry Collector                    │
│  ┌───────────┐   ┌────────────┐   ┌─────────────┐     │
│  │ Receivers │ → │ Processors │ → │  Exporters  │     │
│  └───────────┘   └────────────┘   └─────────────┘     │
│                                           │              │
└───────────────────────────────────────────┼──────────────┘
                                            │
                    ┌───────────────────────┴───────────────┐
                    │                                       │
                    ↓                                       ↓
            ┌──────────────┐                      ┌───────────────┐
            │  Coralogix   │                      │ Local Tools   │
            │ (Production) │                      │ (Development) │
            │              │                      │               │
            │ • Logs       │                      │ • Jaeger      │
            │ • Metrics    │                      │ • Prometheus  │
            │ • Traces     │                      │ • OpenSearch  │
            └──────────────┘                      └───────────────┘
```

---

## 🔗 Important Links

**Coralogix Docs:**
- Dashboard: https://coralogix.com/docs/custom-dashboards/
- Parsing Rules: https://coralogix.com/docs/log-parsing-rules/
- Events2Metrics: https://coralogix.com/docs/events2metrics/
- Alerts: https://coralogix.com/docs/alerts/

**OpenTelemetry:**
- Demo Docs: https://opentelemetry.io/docs/demo/
- Collector Config: https://opentelemetry.io/docs/collector/configuration/

**Tools:**
- Regex Tester: https://regex101.com/
- YAML Validator: https://www.yamllint.com/

---

## ⏱️ Estimated Time

- Configuration: 10 minutes
- Helm upgrade: 5 minutes
- Data to appear: 2-3 minutes
- **Total: ~20 minutes**

---

## ✅ Success Checklist

- [ ] Downloaded coralogix-override.yaml
- [ ] Updated API key in the file
- [ ] Updated domain/endpoints for your region
- [ ] Ran helm upgrade command
- [ ] All pods restarted successfully
- [ ] Collector logs show "Everything is ready"
- [ ] Logs visible in Coralogix with K8s metadata
- [ ] Traces visible in Coralogix
- [ ] Metrics visible in Coralogix
- [ ] Infrastructure metrics (CPU, memory) present
- [ ] Can see service map with all services

---

## 💡 Pro Tips

1. **Keep port-forward running** to test the demo app:
   ```bash
   kubectl port-forward svc/my-otel-demo-frontendproxy 8080:8080 -n otel-demo
   ```

2. **Generate more traffic** to see more data:
   ```bash
   # In another terminal, send requests
   for i in {1..100}; do curl -s http://localhost:8080 > /dev/null; sleep 1; done
   ```

3. **Monitor in real-time**:
   ```bash
   # Watch logs stream
   kubectl logs -n otel-demo -l app.kubernetes.io/component=frontend -f
   ```

4. **Export config for documentation**:
   ```bash
   helm get values my-otel-demo -n otel-demo > current-config.yaml
   ```

---

**Remember:** The assignment says to reach out if you get stuck! Don't hesitate to contact Coralogix with questions.

Good luck! 🎉
