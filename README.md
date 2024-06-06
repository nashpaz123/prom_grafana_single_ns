Sure! Here is the detailed guide in markdown format:

---

# Monitoring a Specific Namespace with Prometheus and Grafana

This guide walks you through installing Prometheus and Grafana on a Kubernetes cluster using Helm, configuring Prometheus to scrape all resources in a specific namespace using a `ServiceMonitor`, and setting up Grafana to visualize those metrics.

## Prerequisites

Kubernetes cluster.
2. **Helm**: 

```sh
wget https://get.helm.sh/helm-v3.14.4-linux-amd64.tar.gz
 tar -zxvf helm-v3.14.4-linux-amd64.tar.gz
 sudo mv linux-amd64/helm /usr/local/bin/helm

alias k=kubectl
alias kk="kubectl -n target-namespace"
```

### Step 1: Add Helm Repositories

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### Step 2: Create a Namespace if it doesnt exist

```sh
kubectl create namespace monitoring
```

### Step 3: Install Prometheus Operator in the ns #TODO: check if can have extra monitoring ns for each ns
Certainly! You can automate the configuration of Grafana and the addition of Prometheus as a data source using configuration files and Kubernetes resources. Here’s how you can achieve this.

### Step 7: Configure Prometheus as a Data Source in Grafana (Automated)

You can use a ConfigMap to automate the configuration of Prometheus as a data source in Grafana.

Create a file named `grafana-datasource.yaml` with the following content:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: target-namespace
  labels:
    grafana_datasource: "1"
data:
  prometheus-datasource.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      access: proxy
      url: http://prometheus-operated.target-namespace.svc.cluster.local:9090
      isDefault: true
      editable: true
```

Apply the ConfigMap to your Kubernetes cluster:

```sh
kubectl apply -f grafana-datasource.yaml
```

This ConfigMap will automatically configure Grafana to use Prometheus as a data source.

### Step 8: Create Dashboards (Automated)

You can create Grafana dashboards by using ConfigMaps or JSON files and apply them to Grafana.

Create a file named `grafana-dashboard-configmap.yaml` with the following content:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: target-namespace
  labels:
    grafana_dashboard: "1"
data:
  sample-dashboard.json: |
    {
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": "-- Grafana --",
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "gnetId": null,
      "graphTooltip": 0,
      "id": null,
      "links": [],
      "panels": [
        {
          "datasource": "prometheus",
          "fieldConfig": {
            "defaults": {
              "thresholds": {
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              },
              "mappings": [],
              "units": "percent"
            }
          },
          "gridPos": {
            "h": 9,
            "w": 12,
            "x": 0,
            "y": 0
          },
          "id": 2,
          "options": {
            "colorMode": "value",
            "graphMode": "area",
            "justifyMode": "auto",
            "orientation": "auto",
            "reduceOptions": {
              "calcs": [
                "mean"
              ],
              "fields": "",
              "values": false
            }
          },
          "targets": [
            {
              "expr": "sum(rate(container_cpu_usage_seconds_total{namespace=\"target-namespace\"}[5m])) by (pod)",
              "interval": "",
              "legendFormat": "{{pod}}",
              "refId": "A"
            }
          ],
          "title": "CPU Usage",
          "type": "stat"
        }
      ],
      "schemaVersion": 27,
      "style": "dark",
      "tags": [],
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-6h",
        "to": "now"
      },
      "timepicker": {
        "refresh_intervals": [
          "5s",
          "10s",
          "30s",
          "1m",
          "5m",
          "15m",
          "30m",
          "1h",
          "2h",
          "1d"
        ],
        "time_options": [
          "5m",
          "15m",
          "1h",
          "6h",
          "12h",
          "24h",
          "2d",
          "7d",
          "30d"
        ]
      },
      "timezone": "",
      "title": "pod Dashboard",
      "uid": null,
      "version": 1
    }
```

Apply the ConfigMap to your Kubernetes cluster:

```sh
kubectl apply -f grafana-dashboard-configmap.yaml
```

### Applying the Configuration

1. Ensure the Grafana deployment picks up the ConfigMap. The Grafana Helm chart usually has options to specify the data source and dashboard directories. Update the Helm release to include these ConfigMaps:

```sh
helm upgrade grafana grafana/grafana \
  --namespace target-namespace \
  --set datasources.enabled=true \
  --set datasources.configMapName=grafana-datasources \
  --set dashboards.enabled=true \
  --set dashboards.configMapName=grafana-dashboards
```

### nodeport for the damned vmdev
```
#nodeport-grafana.yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana  # Service name
  namespace: target-namespace  # Target namespace
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: grafana # Target pods with label app.kubernetes.io/name=grafana
  ports:
  - protocol: TCP
    port: 80  # External port for access
    targetPort: 3000  # Port where Grafana listens in pods
#    nodePort: 31000  # Optional: Specify a specific NodePort (change if desired)
#kubectl apply -f nodeport-grafana.yaml && kk get svc |grep -v node
```
that Grafana is ready to visualize metrics from your specific namespace without any manual setup steps.

--- 
### Manually if getting syntax errors at grafana helm upgrade:

   Access Grafana by opening your browser and navigating to `http://localhost:3000`. Log in with the username `admin` and the password retrieved above.

### Step 9: Configure Prometheus as a Data Source in Grafana

1. In Grafana, go to **Configuration** > **Data Sources**.
2. Click **Add data source** and select **prometheus**. (note the lowercase p that is also required in the json for the dashboard, for datasource: "prometheus")
3. Set the URL to `http://prometheus-operated.monitoring.svc.cluster.local:9090`.
4. Click **Save & Test** to verify the configuration.

### Step 10: Create Dashboards

You can now create dashboards in Grafana to visualize metrics from your specific namespace.

1. Go to **Create** > **Dashboard**.
2. Add panels to your dashboard and use Prometheus queries to fetch metrics from the `target-namespace`. For example:

   ```prometheus
   sum(rate(container_cpu_usage_seconds_total{namespace="target-namespace"}[5m])) by (pod)
   ```

## Summary

This guide provides a robust monitoring solution tailored to your specific namespace requirements by installing Prometheus Operator and Grafana on a Kubernetes cluster, configuring Prometheus to scrape all resources in a specific namespace using a `ServiceMonitor`, and setting up Grafana to visualize those metrics.

---

You can use this markdown content in your documentation or any markdown-supported platform.
