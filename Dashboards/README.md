# Monitoring Dashboards

This directory contains a collection of Grafana dashboards for monitoring various components of our Kubernetes infrastructure and applications. These dashboards provide comprehensive visibility into the health, performance, and status of the systems.

## Dashboard Categories

The dashboards are organized into three main categories:

### 1. Kubernetes (k8s)

Dashboards for monitoring Kubernetes cluster components:

- **k8s_API_srv.json** - Kubernetes API Server monitoring
  - Tracks API server availability, request rates, latency, and resource usage
  - Includes metrics for work queue depth and error rates

- **k8s_coredns.json** - CoreDNS monitoring
  - DNS query performance and error rates
  - Cache hit/miss ratios and resource usage

- **k8s_global.json** - Global cluster overview
  - High-level view of cluster health and resource utilization
  - Node status and capacity planning metrics

- **k8s_namespaces.json** - Namespace resource utilization
  - CPU, memory, and storage usage by namespace
  - Pod distribution and status by namespace

- **k8s_nodes.json** - Node-level monitoring
  - Detailed node resource utilization
  - System metrics (CPU, memory, disk, network)
  - Node conditions and status

- **k8s_pods.json** - Pod monitoring
  - Pod resource usage and status
  - Container metrics and health checks

- **k8s_prom_exporter.json** - Prometheus exporters overview
  - Status and metrics of various Prometheus exporters
  - Scrape durations and success rates

- **Ingress_NGINX.json** - NGINX Ingress Controller
  - Request rates, latencies, and status codes
  - Connection metrics and resource usage

### 2. Applications (app)

Dashboards for monitoring specific applications and services:

- **BlackBox_Exporter.json** - External endpoint monitoring
  - HTTP, HTTPS, DNS, TCP, and ICMP probe results
  - Response times and availability metrics

- **NGINX_Exporter.json** - NGINX web server metrics
  - Connection rates and active connections
  - Request processing metrics and error rates

- **PHP-FPM.json** - PHP-FPM performance monitoring
  - Process manager metrics
  - Request handling and queue metrics
  - Resource utilization

### 3. Infrastructure (infra)

Dashboards for monitoring underlying infrastructure:

- **Node_Exporter_Full.json** - Comprehensive host metrics
  - Detailed system metrics for CPU, memory, disk, and network
  - Process and service status
  - Hardware health indicators

## Usage

### Importing Dashboards

To import these dashboards into Grafana instance:

1. Log in to your Grafana instance
2. Navigate to Dashboards > Import
3. Either upload the JSON file or paste the dashboard JSON content
4. Select the appropriate Prometheus data source
5. Click Import

### Dashboard Variables

Most dashboards include the following variables that can be adjusted:

- `datasource` - The Prometheus data source to use
- `cluster` - Filter by specific Kubernetes cluster
- `namespace` - Filter by specific Kubernetes namespace
- `resolution` - Adjust the metric resolution/sampling rate

### Customization

These dashboards can be customized to suit your specific monitoring needs:

- Adjust thresholds for alerts and visualizations
- Add or remove panels as needed
- Modify queries to include additional metrics or filters

### Drilldown Dashboards Summary
![image](https://github.com/user-attachments/assets/1b01be8b-fb9e-4c4e-afa9-1427663f0975)

## Dashboard Sources

These dashboards are based on or inspired by:

- [dotdc/grafana-dashboards-kubernetes](https://github.com/dotdc/grafana-dashboards-kubernetes)
- [kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx/tree/main/deploy/grafana/dashboards)
- [rfmoz/grafana-dashboards](https://github.com/rfmoz/grafana-dashboards)

