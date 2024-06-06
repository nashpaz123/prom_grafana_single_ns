Sure! Here is the detailed guide in markdown format:

---

# Monitoring a Specific Namespace with Prometheus and Grafana

This guide walks you through installing Prometheus and Grafana on a Kubernetes cluster using Helm, configuring Prometheus to scrape all resources in a specific namespace using a `ServiceMonitor`, and setting up Grafana to visualize those metrics.

## Prerequisites

1. **Kubernetes Cluster**: Ensure you have a running Kubernetes cluster.
2. **Helm**: Ensure Helm is installed and configured to use your Kubernetes cluster.

## Step-by-Step Guide

### Step 1: Add Helm Repositories

First, add the Prometheus and Grafana Helm chart repositories:

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### Step 2: Create a Namespace

Create a namespace for monitoring tools:

```sh
kubectl create namespace monitoring
```

### Step 3: Install Prometheus Operator

Use Helm to install Prometheus Operator in the `monitoring` namespace:

```sh
helm install prometheus-operator prometheus-community/kube-prometheus-stack --namespace monitoring
```

### Step 4: Create a Namespace to Monitor

Create the namespace you want to monitor (if it doesn't exist already):

```sh
kubectl create namespace target-namespace
```

### Step 5: Create a ServiceMonitor

A `ServiceMonitor` custom resource specifies how Prometheus should monitor services within a namespace. Here’s an example `ServiceMonitor` YAML file:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: target-namespace-servicemonitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      monitor: 'true'
  namespaceSelector:
    matchNames:
      - target-namespace
  endpoints:
    - port: web
      interval: 30s
      scrapeTimeout: 10s
```

In this configuration:
- `selector.matchLabels` specifies that Prometheus should monitor services with the label `monitor: 'true'`.
- `namespaceSelector.matchNames` specifies the namespace to monitor (`target-namespace`).
- `endpoints` defines how Prometheus should scrape the metrics.

Apply this `ServiceMonitor`:

```sh
kubectl apply -f servicemonitor.yaml
```

### Step 6: Label Services in the Target Namespace

Label the services in the `target-namespace` that you want Prometheus to monitor:

```sh
kubectl label service your-service monitor=true -n target-namespace
```

### Step 7: Install Grafana

Use Helm to install Grafana in the `monitoring` namespace:

```sh
helm install grafana grafana/grafana --namespace monitoring
```

### Step 8: Access Grafana

1. **Get the Grafana Admin Password**:

   ```sh
   kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
   ```

2. **Port-forward to access Grafana UI**:

   ```sh
   kubectl port-forward --namespace monitoring svc/grafana 3000:80
   ```

   Access Grafana by opening your browser and navigating to `http://localhost:3000`. Log in with the username `admin` and the password retrieved above.

### Step 9: Configure Prometheus as a Data Source in Grafana

1. In Grafana, go to **Configuration** > **Data Sources**.
2. Click **Add data source** and select **Prometheus**.
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
