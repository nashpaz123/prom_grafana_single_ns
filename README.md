To use this setup for different namespaces, you would:

1. Set the `NAMESPACE` variable to the desired namespace name.
2. Run through these steps, which will create a separate Prometheus and Grafana installation for each namespace, each monitoring only its own namespace.

Remember to adjust resource limits, persistence, and security settings according to your specific needs and environment. Also, ensure that your applications in the namespace are properly configured to expose metrics for Prometheus to scrape.

```bash
export NAMESPACE=your-target-namespace
```

Here's the complete process:

1. Create the namespace:

```bash
kubectl create namespace $NAMESPACE
```

2. Install Prometheus:

Pull and untar the Prometheus Helm chart:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm pull prometheus-community/prometheus
tar -xvf prometheus-*.tgz
```

Create a values file for Prometheus (prometheus-values.yaml):

```yaml
server:
  global:
    scrape_interval: 15s
  persistentVolume:
    enabled: false

serviceAccounts:
  server:
    create: true
    name: "prometheus-server-{{ .Values.namespace }}"

prometheus:
  prometheusSpec:
    serviceMonitorSelector:
      matchLabels:
        prometheus: "{{ .Values.namespace }}"
    serviceMonitorNamespaceSelector:
      matchNames:
        - "{{ .Values.namespace }}"

rbac:
  create: true
```

Install Prometheus:

```bash
helm install prometheus-$NAMESPACE ./prometheus -n $NAMESPACE -f prometheus-values.yaml --set namespace=$NAMESPACE
```

3. Install Grafana:

Pull and untar the Grafana Helm chart:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm pull grafana/grafana
tar -xvf grafana-*.tgz
```

Create a values file for Grafana (grafana-values.yaml):

```yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-{{ .Values.namespace }}-server
      access: proxy
      isDefault: true

dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      editable: true
      options:
        path: /var/lib/grafana/dashboards

dashboards:
  default:
    cpu-usage:
      json: |
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
              "aliasColors": {},
              "bars": false,
              "dashLength": 10,
              "dashes": false,
              "datasource": null,
              "fieldConfig": {
                "defaults": {
                  "custom": {}
                },
                "overrides": []
              },
              "fill": 1,
              "fillGradient": 0,
              "gridPos": {
                "h": 9,
                "w": 12,
                "x": 0,
                "y": 0
              },
              "hiddenSeries": false,
              "id": 2,
              "legend": {
                "avg": false,
                "current": false,
                "max": false,
                "min": false,
                "show": true,
                "total": false,
                "values": false
              },
              "lines": true,
              "linewidth": 1,
              "nullPointMode": "null",
              "options": {
                "alertThreshold": true
              },
              "percentage": false,
              "pluginVersion": "7.2.0",
              "pointradius": 2,
              "points": false,
              "renderer": "flot",
              "seriesOverrides": [],
              "spaceLength": 10,
              "stack": false,
              "steppedLine": false,
              "targets": [
                {
                  "expr": "sum(rate(container_cpu_usage_seconds_total{namespace=\"NAMESPACE_PLACEHOLDER\"}[5m])) by (pod)",
                  "interval": "",
                  "legendFormat": "",
                  "refId": "A"
                }
              ],
              "thresholds": [],
              "timeFrom": null,
              "timeRegions": [],
              "timeShift": null,
              "title": "CPU Usage",
              "tooltip": {
                "shared": true,
                "sort": 0,
                "value_type": "individual"
              },
              "type": "graph",
              "xaxis": {
                "buckets": null,
                "mode": "time",
                "name": null,
                "show": true,
                "values": []
              },
              "yaxes": [
                {
                  "format": "short",
                  "label": null,
                  "logBase": 1,
                  "max": null,
                  "min": null,
                  "show": true
                },
                {
                  "format": "short",
                  "label": null,
                  "logBase": 1,
                  "max": null,
                  "min": null,
                  "show": true
                }
              ],
              "yaxis": {
                "align": false,
                "alignLevel": null
              }
            }
          ],
          "schemaVersion": 26,
          "style": "dark",
          "tags": [],
          "templating": {
            "list": []
          },
          "time": {
            "from": "now-6h",
            "to": "now"
          },
          "timepicker": {},
          "timezone": "",
          "title": "CPU Usage Dashboard",
          "uid": null,
          "version": 1
        }

admin:
  existingSecret: ""
  userKey: admin-user
  passwordKey: admin-password

auth:
  disable_login_form: true
  anonymous:
    enabled: true
    org_role: Admin

# This ensures that the admin user is created but with an empty password
adminUser: admin
adminPassword: ""

# Explicitly disable the secret creation
createSecret: false

# Add this to ensure the changes take effect
env:
  GF_AUTH_ANONYMOUS_ENABLED: "true"
  GF_AUTH_ANONYMOUS_ORG_ROLE: "Admin"
  GF_AUTH_DISABLE_LOGIN_FORM: "true"
```

Install Grafana:

```bash
sed "s/NAMESPACE_PLACEHOLDER/$NAMESPACE/g" grafana-values.yaml > grafana-values-$NAMESPACE.yaml #needed because the json panel query inside the yaml doesnt receive the {{ Values.namespace }} value
helm install grafana-$NAMESPACE ./grafana -n $NAMESPACE -f grafana-values-$NAMESPACE.yaml --set namespace=$NAMESPACE
```

4. Create a ServiceMonitor for Prometheus:

Apply the ServiceMonitor:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: ${NAMESPACE}-monitor
  namespace: ${NAMESPACE}
  labels:
    prometheus: ${NAMESPACE}
spec:
  namespaceSelector:
    matchNames:
      - ${NAMESPACE}
  selector: {}
  endpoints:
    - port: metrics
      path: /metrics
      interval: 15s
EOF
```

5. (optional) Stupid nodeport for grafana if you aint got no ingress but want to get to the gui without proxy:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: ${NAMESPACE}
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: grafana
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
EOF
```

