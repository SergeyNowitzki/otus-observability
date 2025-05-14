# Prometheus Exporters Guide
This guide explains how to set up and configure various exporters for Prometheus monitoring in our Kubernetes environment.

## Overview
Prometheus exporters are specialized applications that collect metrics from various services and expose them in a format Prometheus can scrape. Our setup includes:

- Node Exporter (installed with Prometheus stack)
- MariaDB Exporter
- NGINX Exporter
- PHP-FPM Exporter
- Blackbox Exporter

## Node Exporter
Prometheus was installed as part of the infrastructure. Along with the Prometheus stack, Node Exporters are automatically deployed on each worker node in the cluster, and the Prometheus configuration is preconfigured to scrape metrics from these exporters by default `infrastructure/prom-custom-values.yaml`:
```bash
root@node1:~# kubectl get pods -o wide | grep node-exporter
prometheus-prometheus-node-exporter-f5tx7                      1/1     Running   3 (9d ago)    38d   192.168.99.213   node3   <none>           <none>
prometheus-prometheus-node-exporter-l4nkb                      1/1     Running   2 (9d ago)    38d   192.168.99.212   node2   <none>           <none>
prometheus-prometheus-node-exporter-t5zlw                      1/1     Running   2 (25d ago)   38d   192.168.99.211   node1   <none>           <none>

root@node1:~# kubectl get svc | grep node-exporter
prometheus-prometheus-node-exporter            ClusterIP   10.233.4.195    <none>        9100/TCP            39d

root@node1:~# dig +short prometheus-prometheus-node-exporter.default.svc.cluster.local
10.233.4.195
```
![image](https://github.com/user-attachments/assets/a0092c9f-5a40-4135-bff4-660a8aa77cef)

A `curl` pod will be deployed for exporter testing purposes:
```bash
root@node1:~# kubectl run curlpod -n default --rm -i -t --restart=Never --image=alpine -- sh
/ # apk add --no-cache curl jq
/ # curl prometheus-prometheus-node-exporter.default.svc.cluster.local:9100/metrics -s | grep node_os_info
# HELP node_os_info A metric with a constant '1' value labeled by build_id, id, id_like, image_id, image_version, name, pretty_name, variant, variant_id, version, version_codename, version_id.
# TYPE node_os_info gauge
node_os_info{build_id="",id="ubuntu",id_like="debian",image_id="",image_version="",name="Ubuntu",pretty_name="Ubuntu 20.04.6 LTS",variant="",variant_id="",version="20.04.6 LTS (Focal Fossa)",version_codename="focal",version_id="20.04"} 1
```
## MySQL Exporter
The MySQL Exporter collects metrics from MariaDB/MySQL databases.

### Installation
```bash
# Add the Prometheus community Helm repository if you haven't already
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo mysql

# Install the MySQL exporter with custom values
helm install mysql-exporter prometheus-community/prometheus-mysql-exporter \
  -f mariadb-exporter-values.yaml \
  -n default

# If we need to upgrade
helm upgrade mysql-exporter prometheus-community/prometheus-mysql-exporter \
  -f mariadb-exporter-values.yaml \
  -n default
```

### Configuration
The `mariadb-exporter-values.yaml` file contains the configuration for connecting to MariaDB:
```yaml
mysql:
  host: "mariadb"
  port: 3306
  user: "root"
  pass: "rootpassword"

service:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9104"
```

The `prometheus.io/scrape: "true"` annotation enables **automatic discovery** and scraping of the exporter by Prometheus.

### Verification
```bash
root@node1:~# kubectl get pods | grep mysql
mysql-exporter-prometheus-mysql-exporter-656c7c4f57-tftd7      1/1     Running   0             9d
root@node1:~# kubectl get svc | grep mysql
mysql-exporter-prometheus-mysql-exporter       ClusterIP   10.233.46.144   <none>        9104/TCP            29d

/ # curl mysql-exporter-prometheus-mysql-exporter.default.svc.cluster.local:9104/metrics -s | grep mysql_up
# HELP mysql_up Whether the MySQL server is up.
# TYPE mysql_up gauge
mysql_up 1
```

## NGINX Exporter
The NGINX Exporter collects metrics from NGINX web servers.

### Installation
```bash
# Add the Prometheus community Helm repository if you haven't already
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo nginx

# Install the Nginx exporter with your custom values
helm install nginx-exporter prometheus-community/prometheus-nginx-exporter \
  -f nginx-exporter-values.yaml \
  -n default
```
### Configuration
The `nginx-exporter-values.yaml` file contains:
```yaml
nginxServer: "http://nginx.default.svc.cluster.local:80/stub_status"

service:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9113"
```
### Verification
```bash
root@node1:~# kubectl get svc | grep nginx-exporter
nginx-exporter-prometheus-nginx-exporter       ClusterIP   10.233.35.231   <none>        9113/TCP            29d
root@node1:~# kubectl get pod | grep nginx-exporter
nginx-exporter-prometheus-nginx-exporter-7c5d99bd65-gvkhn      1/1     Running   0             9d

curl nginx-exporter-prometheus-nginx-exporter.default.svc.cluster.local:9113/metrics -s | grep nginx_up
# HELP nginx_up Status of the last metric scrape
# TYPE nginx_up gauge
nginx_up 1
```

## NGINX Ingress Controller Metrics

### Deploy the Ingress Controller as a LoadBalancer:
Install an Ingress controller (NGINX Ingress) with a LoadBalancer service type:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace --set controller.service.type=LoadBalancer
```

### Enable metrics collection for the NGINX Ingress Controller:
```bash
helm upgrade ingress-nginx ingress-nginx \
--repo https://kubernetes.github.io/ingress-nginx \
--namespace ingress-nginx \
--set controller.metrics.enabled=true \
--set-string controller.podAnnotations."prometheus\.io/scrape"="true" \
--set-string controller.podAnnotations."prometheus\.io/port"="10254"

helm get values ingress-nginx --namespace ingress-nginx
USER-SUPPLIED VALUES:
controller:
  metrics:
    enabled: true
  podAnnotations:
    prometheus.io/port: "10254"
    prometheus.io/scrape: "true"
```

## Blackbox Exporter
The Blackbox Exporter allows probing of endpoints over HTTP, HTTPS, DNS, TCP, ICMP, and gRPC.
https://github.com/prometheus/blackbox_exporter

### Installation
```bash
helm search repo blackbox
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
Install the Blackbox exporter using the values `blackbox-exporter-values.yaml` file:
```bash
helm install blackbox prometheus-community/prometheus-blackbox-exporter \
  -f exporters/blackbox-exporter-values.yaml \
  -n default
```

### Configuration
In the `infrastructure/prom-custom-values.yaml` file, add the following scrape configuration:
```yaml
extraScrapeConfigs: |
  - job_name: 'prometheus-blackbox-exporter'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - google.com
          - wordpress.local
          - grafana.local
          - prometheus.local
          - minio.local
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-prometheus-blackbox-exporter:9115
```


```bash
# Apply the changes to the Prometheus Helm chart
helm upgrade prometheus prometheus-community/prometheus -f prom-custom-values.yaml -n default
```


## PHP-FPM Exporter
The PHP-FPM Exporter collects metrics from PHP-FPM.

### Installation
```bash
git clone https://github.com/brunowego/prometheus-php-fpm-exporter.git
cd prometheus-php-fpm-exporter
helm install php-fpm-exporter . -f values-exporter.yaml -n default
```
### Configuration
Add the PHP-FPM exporter to Prometheus by updating the `extraScrapeConfigs` section in the `infrastructure/prom-custom-values.yaml` :
```yaml
extraScrapeConfigs: |
  - job_name: 'wordpress-phpfpm'
    metrics_path: /metrics
    static_configs:
      - targets:
          - 'php-fpm-exporter-prometheus-php-fpm-exporter.default.svc.cluster.local:9253'
```
Apply the changes:
```bash
helm upgrade prometheus prometheus-community/prometheus -f prom-custom-values.yaml -n default
```

## Service Discovery in Prometheus
Prometheus uses Kubernetes service discovery to automatically find and scrape metrics from pods and services.

### Annotations-Based Discovery
Prometheus is configured to discover services with specific annotations:

- `prometheus.io/scrape: "true"` : Only scrape services with this annotation set to true
- `prometheus.io/port: "<port>"` : Specifies which port to scrape
- `prometheus.io/path: "<path>"` : Specifies the metrics path (defaults to /metrics )
This allows for automatic discovery of new exporters when they're deployed with the correct annotations.

### Verification
To verify that all exporters are running:
```bash
root@node1:~# kubectl get pods | grep exporter
blackbox-prometheus-blackbox-exporter-55d65fddfc-hnqlz         1/1     Running   1 (60m ago)    20d
mysql-exporter-prometheus-mysql-exporter-656c7c4f57-tftd7      1/1     Running   0              12d
nginx-exporter-prometheus-nginx-exporter-7c5d99bd65-gvkhn      1/1     Running   0              12d
php-fpm-exporter-prometheus-php-fpm-exporter-b7bb76476-w97pn   1/1     Running   0              12d
prometheus-prometheus-node-exporter-f5tx7                      1/1     Running   3 (12d ago)    42d
prometheus-prometheus-node-exporter-l4nkb                      1/1     Running   2 (12d ago)    42d
prometheus-prometheus-node-exporter-t5zlw                      1/1     Running   3 (60m ago)    42d
```
