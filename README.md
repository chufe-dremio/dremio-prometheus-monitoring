# Dremio Prometheus Monitoring

### Initial Prometheus Setup

#### Create a Namespace for Prometheus
Create a Namespace for monitoring 

Example:

`kubectl create ns monitoring`

#### Install Metrics Server
In order to use the default scaling metrics of CPU and memory usage, [metrics server](https://github.com/kubernetes-sigs/metrics-server) will need to be installed
This step it not required for Azure Kubernetes Service (AKS).

```
# Add the kubernetes-sigs repo
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/

# Install Metrics Server
helm install  metrics-server metrics-server/metrics-server -n monitoring
```

#### Install Prometheus Stack
In order to have the required Custom Resource Definitions (CRD), we will need 
[Prometheus Stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) installed.

Use the Prometheus stack template file [values_prometheus.hc.55.7.0.yml](./kube-prometheus-stack/values_prometheus.hc.55.7.0.yml) and create a copy.
Walk through the file and replace all values which are marked with a 'TODO'. This includes node selectors, storage classes and hostnames.

```
# Add the prometheus-community repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm update

# Install kube-prometheus-stack
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack -n monitoring --version 55.7.0 -f kube-prometheus-stack/values_prometheus.hc.55.7.0.yml
```

### Configure Dremio for Prometheus Monitoring
Dremio needs to be configured, so that it exposes the metrics to Prometheus. The guide assumes that Dremio's official Helm Charts have been used: https://github.com/dremio/dremio-cloud-tools

#### Configure Dremio
When Dremio is deployed using the official Helm Charts the `values.yml` needs to be copied and modified.
The following needs to be added to the `values.yml`:

```
coordinator:
  extraStartParams: >-
    -Dservices.web-admin.port=9010
    -Dservices.web-admin.enabled=true
    -Dservices.web-admin.host=0.0.0.0

executor:
  extraStartParams: >-
    -Dservices.web-admin.port=9010
    -Dservices.web-admin.enabled=true
    -Dservices.web-admin.host=0.0.0.0
```

Additionally, it is required to patch `templates/dremio-executor.yml` and `templates/dremio-master.yaml` and add the container port for monitoring:

```
        - containerPort: 9010
          name: prometheus
```

#### Configure Zookeeper
To enable Prometheus metrics for Zookeeper, the `templates/zookeeper.yml` file needs to be patched:

```
        # An additional env variable needs to be added (around line 110).
        - name: ZOO_CFG_EXTRA
          value: metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider metricsProvider.httpPort=9010
        # Web port needs to be added (around line 126)
        - containerPort: 9010
          name: prometheus
```

#### Add pod monitors
To add the pod monitors so that Prometheus starts scraping the metrics from Dremio and Zookeeper, run the following command:
```
kubectl apply -n <dremio namespace> -f https://raw.githubusercontent.com/chufe-dremio/dremio-prometheus-monitoring/main/specs/dremio-pod-monitor.yaml
kubectl apply -n <dremio namespace> -f https://raw.githubusercontent.com/chufe-dremio/dremio-prometheus-monitoring/main/specs/zookeeper-pod-monitor.yaml
```

#### Install the Grafana Dashboard

Login into Grafana and click the '+' icon at the top right and select "Import dashboard". Upload the file from this repository: [Dremio_Monitoring.json](./grafana/Dremio_Monitoring.json).