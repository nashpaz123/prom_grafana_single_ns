To use this setup for different namespaces, you would:

1. Set the `NAMESPACE` variable to the desired namespace name.
2. Run through these steps, which will create a separate Prometheus and Grafana installation for each namespace, each monitoring only its own namespace.

Remember to adjust resource limits, persistence, and security settings according to your specific needs and environment. Also, ensure that your applications in the namespace are properly configured to expose metrics for Prometheus to scrape.

```bash
```

Here's the short process:

1. Clone this repo, Enter the namespace var, Create the namespace:

```bash
git clone https://github.com/nashpaz123/prom_grafana_single_ns.git
export NAMESPACE=your-target-namespace
kubectl create namespace $NAMESPACE
```

2. Install Prometheus:

```bash
helm install prometheus-$NAMESPACE ./prometheus -n $NAMESPACE -f prometheus-values.yaml --set namespace=$NAMESPACE
```

3. Install Grafana:

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

kubectl -n ${NAMESPACE} get svc |grep grafana
```

