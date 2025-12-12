# PROMETHEUS & GRAFANA - Complete Guide

## What is Prometheus?

### Overview
Prometheus is an open-source monitoring and alerting toolkit designed for reliability and scalability.

### Key Features
- ✅ Time-series database - Stores metrics with timestamps
- ✅ Pull-based model - Scrapes metrics from targets
- ✅ Powerful query language - PromQL for data analysis
- ✅ Service discovery - Automatically finds targets in Kubernetes
- ✅ Alerting - Built-in alerting rules
- ✅ No dependencies - Single binary, easy to deploy

### How It Works
```
1. Prometheus SCRAPES metrics from targets (every 15s)
2. Stores them in TIME-SERIES DATABASE
3. You QUERY data using PromQL
4. ALERTS fire when conditions are met
```

### Architecture
```
┌─────────────────────────────────────┐
│     KUBERNETES CLUSTER              │
│                                     │
│  Node Exporter  ← Scrapes ← ┐     │
│  Kube-State-Metrics ← ┐      │     │
│  Application Pods ← ┐  │      │     │
└────────────────────┼──┼──────┼─────┘
                     │  │      │
              ┌──────┴──┴──────┴────┐
              │   PROMETHEUS        │
              │   - Scrapes         │
              │   - Stores          │
              │   - Queries         │
              │   - Alerts          │
              └──────────┬──────────┘
                         │
              ┌──────────┴──────────┐
              │   GRAFANA           │
              │   (Visualize)       │
              └─────────────────────┘
```

---

## What is Grafana?

### Overview
Grafana is an open-source analytics and visualization platform that turns time-series data into beautiful dashboards.

### Key Features
- ✅ Beautiful dashboards - Interactive and customizable
- ✅ Multiple data sources - Prometheus, Elasticsearch, MySQL, etc.
- ✅ Alerting - Visual alerts and notifications
- ✅ Templating - Dynamic dashboards with variables
- ✅ Sharing - Share dashboards with team
- ✅ Plugins - Extensive plugin ecosystem

### What It Does
```
1. CONNECTS to Prometheus (data source)
2. QUERIES metrics using PromQL
3. VISUALIZES data in graphs, charts, tables
4. SENDS alerts based on thresholds
```



---

## Types of Metrics (Organized by Source/Component)

Prometheus collects metrics from 3 main sources:

---

### 1. Node Exporter (Hardware/OS Metrics)

What: Runs on every node (DaemonSet) to collect hardware and OS-level metrics.

Location: 
- Runs on: Every Kubernetes node
- Port: 9100
- Deployment: DaemonSet (1 pod per node)

What it monitors:
```
🖥️ Hardware & Operating System Level

CPU:
- node_cpu_seconds_total           → CPU usage per core
- node_load1, node_load5           → Load average

Memory:
- node_memory_MemTotal_bytes       → Total RAM
- node_memory_MemAvailable_bytes   → Available RAM
- node_memory_MemFree_bytes        → Free RAM
- node_memory_Cached_bytes         → Cached memory

Disk:
- node_filesystem_size_bytes       → Total disk space
- node_filesystem_avail_bytes      → Available disk space
- node_disk_read_bytes_total       → Bytes read from disk
- node_disk_written_bytes_total    → Bytes written to disk
- node_disk_io_time_seconds_total  → Disk I/O time

Network:
- node_network_receive_bytes_total → Bytes received
- node_network_transmit_bytes_total → Bytes transmitted
- node_network_receive_errors_total → Network errors

System:
- node_boot_time_seconds           → System boot time
- node_time_seconds                → Current time
- node_uname_info                  → System info (kernel, OS)
```

Example Queries:
```promql
# Node CPU usage (%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Node Memory usage (%)
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Node Disk usage (%)
(1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100

# Network traffic (bytes/sec)
rate(node_network_receive_bytes_total[5m])
```

Who uses it: Infrastructure monitoring, capacity planning, hardware alerts

---

### 2. kube-state-metrics (Kubernetes Object Metrics)

What: Monitors the state of Kubernetes objects (Pods, Deployments, Nodes, etc.)

Location:
- Runs as: Single Deployment in monitoring namespace
- Port: 8080
- Watches: Kubernetes API Server

What it monitors:
```
☸️ Kubernetes Objects State

Pods:
- kube_pod_info                    → Pod metadata (name, namespace, labels)
- kube_pod_status_phase            → Pod phase (Running, Pending, Failed)
- kube_pod_container_status_ready  → Container ready status
- kube_pod_container_status_restarts_total → Container restart count
- kube_pod_container_resource_requests → CPU/Memory requests
- kube_pod_container_resource_limits → CPU/Memory limits

Nodes:
- kube_node_info                   → Node metadata
- kube_node_status_condition       → Node conditions (Ready, DiskPressure)
- kube_node_status_capacity        → Node capacity (pods, CPU, memory)
- kube_node_status_allocatable     → Allocatable resources

Deployments:
- kube_deployment_status_replicas  → Desired replicas
- kube_deployment_status_replicas_available → Available replicas
- kube_deployment_status_replicas_updated → Updated replicas

Services:
- kube_service_info                → Service metadata
- kube_service_spec_type           → Service type (ClusterIP, NodePort)

Namespaces:
- kube_namespace_status_phase      → Namespace status

Jobs:
- kube_job_status_succeeded        → Successful jobs
- kube_job_status_failed           → Failed jobs
```

Example Queries:
```promql
# Total Pods
count(kube_pod_info)

# Running Pods
count(kube_pod_status_phase{phase="Running"})

# Pod Restarts (last 24h)
sum(increase(kube_pod_container_status_restarts_total[24h])) by (namespace, pod)

# Deployment replica status
kube_deployment_status_replicas_available / kube_deployment_status_replicas

# Nodes not ready
count(kube_node_status_condition{condition="Ready",status!="true"})
```

Who uses it: Kubernetes cluster monitoring, object health, capacity planning

---

### 3. cAdvisor / Container Metrics (Pod/Container Resource Usage)

What: Built into kubelet, monitors actual resource usage of containers.

Location:
- Runs on: Every node (part of kubelet)
- Port: 10250 (kubelet metrics)
- Endpoint: /metrics/cadvisor

What it monitors:
```
📦 Container Runtime Metrics (Actual Usage)

CPU:
- container_cpu_usage_seconds_total → Actual CPU used by container
- container_cpu_system_seconds_total → CPU used in kernel mode
- container_cpu_user_seconds_total → CPU used in user mode

Memory:
- container_memory_usage_bytes     → Current memory usage
- container_memory_working_set_bytes → Working set size
- container_memory_rss             → Resident set size
- container_memory_cache           → Cache memory
- container_memory_swap            → Swap usage

Network:
- container_network_receive_bytes_total → Bytes received
- container_network_transmit_bytes_total → Bytes transmitted
- container_network_receive_packets_total → Packets received
- container_network_transmit_packets_total → Packets transmitted

Filesystem:
- container_fs_usage_bytes         → Filesystem usage
- container_fs_limit_bytes         → Filesystem limit
- container_fs_reads_bytes_total   → Bytes read
- container_fs_writes_bytes_total  → Bytes written

Process:
- container_processes               → Number of processes
- container_threads                 → Number of threads
```

Example Queries:
```promql
# Container CPU usage (cores)
rate(container_cpu_usage_seconds_total{container!=""}[5m])

# Container Memory usage (bytes)
container_memory_usage_bytes{container!=""}

# CPU usage by namespace
sum(rate(container_cpu_usage_seconds_total{namespace!=""}[5m])) by (namespace)

# Memory usage by pod
sum(container_memory_usage_bytes{pod!=""}) by (namespace, pod)

# Network I/O
rate(container_network_receive_bytes_total[5m])
rate(container_network_transmit_bytes_total[5m])
```

Who uses it: Resource monitoring, pod performance, cost allocation

---



---

## Installation Scripts

### Method 1: Separate Installation (Simple)

File: `install-prometheus-grafana.sh`

```bash
#!/bin/bash
# ============================================
# INSTALL PROMETHEUS + GRAFANA SEPARATELY
# ============================================

echo "🚀 Installing Prometheus & Grafana..."

# 1. Add Helm repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 2. Create namespace
kubectl create namespace monitoring

# 3. Install Prometheus
echo "📊 Installing Prometheus..."
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --set server.persistentVolume.enabled=false \
  --set alertmanager.persistence.enabled=false \
  --set server.retention=7d \
  --set nodeExporter.enabled=true \
  --set kubeStateMetrics.enabled=true

# 4. Wait for Prometheus
echo "⏳ Waiting for Prometheus..."
sleep 30

# 5. Install Grafana
echo "📈 Installing Grafana..."
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=false \
  --set adminPassword=admin123

# 6. Wait for Grafana
echo "⏳ Waiting for Grafana..."
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=grafana -n monitoring --timeout=300s

# 7. Expose services as NodePort
echo "🌐 Exposing services..."
kubectl expose service prometheus-server \
  --type=NodePort \
  --target-port=9090 \
  --name=prometheus-nodeport \
  -n monitoring

kubectl expose service grafana \
  --type=NodePort \
  --target-port=3000 \
  --name=grafana-nodeport \
  -n monitoring

# 8. Get access information
echo ""
echo "============================================"
echo "✅ INSTALLATION COMPLETE!"
echo "============================================"
echo ""
echo "📊 Prometheus:"
echo "   NodePort: $(kubectl get svc prometheus-nodeport -n monitoring -o jsonpath='{.spec.ports[0].nodePort}')"
echo "   URL: http://<node-ip>:$(kubectl get svc prometheus-nodeport -n monitoring -o jsonpath='{.spec.ports[0].nodePort}')"
echo ""
echo "📈 Grafana:"
echo "   NodePort: $(kubectl get svc grafana-nodeport -n monitoring -o jsonpath='{.spec.ports[0].nodePort}')"
echo "   URL: http://<node-ip>:$(kubectl get svc grafana-nodeport -n monitoring -o jsonpath='{.spec.ports[0].nodePort}')"
echo "   Username: admin"
echo "   Password: admin123"
echo ""
echo "🔑 To get Grafana password:"
echo "   kubectl get secret grafana -n monitoring -o jsonpath=\"{.data.admin-password}\" | base64 --decode"
echo ""
echo "============================================"
```

---

### Method 2: All-in-One (kube-prometheus-stack)

File: `install-kube-prometheus-stack.sh`

```bash
#!/bin/bash
# ============================================
# INSTALL COMPLETE MONITORING STACK
# Includes: Prometheus + Grafana + Alertmanager + Exporters
# ============================================

echo "🚀 Installing kube-prometheus-stack..."

# 1. Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 2. Install complete stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=7d \
  --set prometheus.prometheusSpec.storageSpec= \
  --set alertmanager.alertmanagerSpec.storage= \
  --set grafana.persistence.enabled=false \
  --set grafana.adminPassword=admin123

# 3. Wait for pods
echo "⏳ Waiting for all pods to be ready..."
kubectl wait --for=condition=ready pod --all -n monitoring --timeout=300s

# 4. Expose services
echo "🌐 Exposing services..."

# Expose Grafana
kubectl expose service monitoring-grafana \
  --type=NodePort \
  --name=grafana-nodeport \
  -n monitoring

# Expose Prometheus
kubectl expose service monitoring-kube-prometheus-prometheus \
  --type=NodePort \
  --name=prometheus-nodeport \
  -n monitoring

# 5. Get access info
echo ""
echo "============================================"
echo "✅ INSTALLATION COMPLETE!"
echo "============================================"
echo ""
echo "📈 Grafana:"
echo "   URL: http://<node-ip>:$(kubectl get svc grafana-nodeport -n monitoring -o jsonpath='{.spec.ports[0].nodePort}')"
echo "   Username: admin"
echo "   Password: admin123"
echo ""
echo "📊 Prometheus:"
echo "   URL: http://<node-ip>:$(kubectl get svc prometheus-nodeport -n monitoring -o jsonpath='{.spec.ports[0].nodePort}')"
echo ""
echo "📦 Components Installed:"
echo "   ✓ Prometheus"
echo "   ✓ Grafana (with 20+ pre-built dashboards!)"
echo "   ✓ Alertmanager"
echo "   ✓ Node Exporter"
echo "   ✓ Kube-State-Metrics"
echo "   ✓ Prometheus Operator"
echo ""
echo "============================================"
```

---

## Dashboard JSON Format

Complete Kubernetes Monitoring Dashboard

This is your actual working dashboard with 14 panels (from json.json file):

```json
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": {
          "type": "datasource",
          "uid": "grafana"
        },
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 0,
  "id": null,
  "links": [],
  "liveNow": false,
  "panels": [
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "short"
        }
      },
      "gridPos": {
        "h": 4,
        "w": 4,
        "x": 0,
        "y": 0
      },
      "id": 1,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "expr": "count(kube_node_info)",
          "refId": "A"
        }
      ],
      "title": "Total Nodes",
      "type": "stat"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "short"
        }
      },
      "gridPos": {
        "h": 4,
        "w": 4,
        "x": 4,
        "y": 0
      },
      "id": 2,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "expr": "count(kube_pod_info)",
          "refId": "A"
        }
      ],
      "title": "Total Pods",
      "type": "stat"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "short"
        }
      },
      "gridPos": {
        "h": 4,
        "w": 4,
        "x": 8,
        "y": 0
      },
      "id": 3,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "expr": "count(kube_pod_status_phase{phase=\"Running\"})",
          "refId": "A"
        }
      ],
      "title": "Running Pods",
      "type": "stat"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 1
              }
            ]
          },
          "unit": "short"
        }
      },
      "gridPos": {
        "h": 4,
        "w": 4,
        "x": 12,
        "y": 0
      },
      "id": 4,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "expr": "sum(kube_pod_status_phase{phase=\"Pending\"})",
          "refId": "A"
        }
      ],
      "title": "Pending Pods",
      "type": "stat"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 1
              }
            ]
          },
          "unit": "short"
        }
      },
      "gridPos": {
        "h": 4,
        "w": 4,
        "x": 16,
        "y": 0
      },
      "id": 5,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "expr": "sum(kube_pod_status_phase{phase=\"Failed\"})",
          "refId": "A"
        }
      ],
      "title": "Failed Pods",
      "type": "stat"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "short"
        }
      },
      "gridPos": {
        "h": 4,
        "w": 4,
        "x": 20,
        "y": 0
      },
      "id": 6,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "expr": "count(count(kube_pod_info) by (namespace))",
          "refId": "A"
        }
      ],
      "title": "Namespaces",
      "type": "stat"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": {
              "tooltip": false,
              "viz": false,
              "legend": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "percent"
        }
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 4
      },
      "id": 7,
      "options": {
        "legend": {
          "calcs": [
            "mean",
            "lastNotNull",
            "max"
          ],
          "displayMode": "table",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "multi",
          "sort": "none"
        }
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "expr": "sum(rate(container_cpu_usage_seconds_total{container!=\"\"}[5m])) by (namespace) * 100",
          "legendFormat": "{{namespace}}",
          "refId": "A"
        }
      ],
      "title": "CPU Usage by Namespace (%)",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": {
              "tooltip": false,
              "viz": false,
              "legend": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "bytes"
        }
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 4
      },
      "id": 8,
      "options": {
        "legend": {
          "calcs": [
            "mean",
            "lastNotNull",
            "max"
          ],
          "displayMode": "table",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "multi",
          "sort": "none"
        }
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "expr": "sum(container_memory_usage_bytes{container!=\"\"}) by (namespace)",
          "legendFormat": "{{namespace}}",
          "refId": "A"
        }
      ],
      "title": "Memory Usage by Namespace",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": {
              "tooltip": false,
              "viz": false,
              "legend": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "Bps"
        }
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 12
      },
      "id": 9,
      "options": {
        "legend": {
          "calcs": [
            "mean",
            "lastNotNull"
          ],
          "displayMode": "table",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "multi",
          "sort": "none"
        }
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "expr": "sum(rate(container_network_receive_bytes_total[5m])) by (namespace)",
          "legendFormat": "{{namespace}} - receive",
          "refId": "A"
        },
        {
          "expr": "sum(rate(container_network_transmit_bytes_total[5m])) by (namespace)",
          "legendFormat": "{{namespace}} - transmit",
          "refId": "B"
        }
      ],
      "title": "Network I/O by Namespace",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "custom": {
            "align": "auto",
            "cellOptions": {
              "type": "auto"
            },
            "inspect": false
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          }
        }
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 12
      },
      "id": 10,
      "options": {
        "cellHeight": "sm",
        "footer": {
          "countRows": false,
          "fields": "",
          "reducer": [
            "sum"
          ],
          "show": false
        },
        "showHeader": true
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "expr": "sum(kube_pod_container_status_restarts_total) by (namespace, pod)",
          "format": "table",
          "instant": true,
          "refId": "A"
        }
      ],
      "title": "Pod Restarts by Namespace",
      "transformations": [
        {
          "id": "organize",
          "options": {
            "excludeByName": {
              "Time": true
            },
            "indexByName": {},
            "renameByName": {
              "Value": "Restarts",
              "namespace": "Namespace",
              "pod": "Pod"
            }
          }
        }
      ],
      "type": "table"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "hideFrom": {
              "tooltip": false,
              "viz": false,
              "legend": false
            }
          },
          "mappings": []
        }
      },
      "gridPos": {
        "h": 8,
        "w": 8,
        "x": 0,
        "y": 20
      },
      "id": 11,
      "options": {
        "legend": {
          "displayMode": "table",
          "placement": "right",
          "showLegend": true,
          "values": [
            "value"
          ]
        },
        "pieType": "pie",
        "tooltip": {
          "mode": "single",
          "sort": "none"
        }
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "expr": "count(kube_pod_info) by (namespace)",
          "legendFormat": "{{namespace}}",
          "refId": "A"
        }
      ],
      "title": "Pods per Namespace",
      "type": "piechart"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "max": 100,
          "min": 0,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "yellow",
                "value": 70
              },
              {
                "color": "red",
                "value": 90
              }
            ]
          },
          "unit": "percent"
        }
      },
      "gridPos": {
        "h": 8,
        "w": 8,
        "x": 8,
        "y": 20
      },
      "id": 12,
      "options": {
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "showThresholdLabels": false,
        "showThresholdMarkers": true
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "expr": "(1 - (node_filesystem_avail_bytes{mountpoint=\"/\"} / node_filesystem_size_bytes{mountpoint=\"/\"})) * 100",
          "legendFormat": "{{node}}",
          "refId": "A"
        }
      ],
      "title": "Node Disk Usage (%)",
      "type": "gauge"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "max": 100,
          "min": 0,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "yellow",
                "value": 70
              },
              {
                "color": "red",
                "value": 90
              }
            ]
          },
          "unit": "percent"
        }
      },
      "gridPos": {
        "h": 8,
        "w": 8,
        "x": 16,
        "y": 20
      },
      "id": 13,
      "options": {
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "showThresholdLabels": false,
        "showThresholdMarkers": true
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "expr": "(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100",
          "legendFormat": "{{node}}",
          "refId": "A"
        }
      ],
      "title": "Node Memory Usage (%)",
      "type": "gauge"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": {
              "tooltip": false,
              "viz": false,
              "legend": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "never",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "normal"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "short"
        }
      },
      "gridPos": {
        "h": 8,
        "w": 24,
        "x": 0,
        "y": 28
      },
      "id": 14,
      "options": {
        "legend": {
          "calcs": [
            "mean"
          ],
          "displayMode": "table",
          "placement": "right",
          "showLegend": true
        },
        "tooltip": {
          "mode": "multi",
          "sort": "none"
        }
      },
      "pluginVersion": "10.0.0",
      "targets": [
        {
          "expr": "sum(kube_pod_status_phase{phase=\"Running\"}) by (namespace)",
          "legendFormat": "{{namespace}} - Running",
          "refId": "A"
        },
        {
          "expr": "sum(kube_pod_status_phase{phase=\"Pending\"}) by (namespace)",
          "legendFormat": "{{namespace}} - Pending",
          "refId": "B"
        },
        {
          "expr": "sum(kube_pod_status_phase{phase=\"Failed\"}) by (namespace)",
          "legendFormat": "{{namespace}} - Failed",
          "refId": "C"
        }
      ],
      "title": "Pod Status Over Time",
      "type": "timeseries"
    }
  ],
  "refresh": "30s",
  "schemaVersion": 38,
  "style": "dark",
  "tags": [
    "kubernetes",
    "monitoring",
    "prometheus"
  ],
  "templating": {
    "list": []
  },
  "time": {
    "from": "now-1h",
    "to": "now"
  },
  "timepicker": {},
  "timezone": "browser",
  "title": "Kubernetes Cluster Monitoring",
  "uid": "k8s-cluster-monitoring",
  "version": 1,
  "weekStart": ""
}
```

### Dashboard Panels Summary

This dashboard includes 14 panels:

Row 1 - Overview (Stats):
1. Total Nodes
2. Total Pods
3. Running Pods
4. Pending Pods
5. Failed Pods
6. Namespaces

Row 2 - Resource Usage (Time Series):
7. CPU Usage by Namespace (%)
8. Memory Usage by Namespace

Row 3 - Network & Restarts:
9. Network I/O by Namespace
10. Pod Restarts by Namespace (Table)

Row 4 - Node Health (Gauges):
11. Pods per Namespace (Pie Chart)
12. Node Disk Usage (%)
13. Node Memory Usage (%)

Row 5 - Pod Status:
14. Pod Status Over Time (Stacked graph)

### How to Import This Dashboard

In Grafana:
1. Click "+" → "Import"
2. Copy and paste this entire JSON
3. Select "prometheus" as data source
4. Click "Import"

Done! Instant dashboard with all panels! 🎉

---

## Basic Queries

```promql
# Total Nodes
count(kube_node_info)

# Total Pods
count(kube_pod_info)

# Running Pods
count(kube_pod_status_phase{phase="Running"})

# Pending Pods
sum(kube_pod_status_phase{phase="Pending"})

# Failed Pods
sum(kube_pod_status_phase{phase="Failed"})

# Pods by Namespace
count(kube_pod_info) by (namespace)
```





