In a Kubernetes environment, you can use Prometheus to monitor MySQL and MariaDB database using the open-source MySQL exporter.
https://github.com/prometheus/mysqld_exporter
# Add the Prometheus community Helm repository if you haven't already
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo mysql

# Install the MySQL exporter with your custom values
helm install mysql-exporter prometheus-community/prometheus-mysql-exporter \
  -f mariadb-exporter-values.yaml \
  -n default


helm upgrade mysql-exporter prometheus-community/prometheus-mysql-exporter \
 -f mariadb-exporter-values.yaml


# Add the Prometheus community Helm repository if you haven't already
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo nginx
# Install the Nginx exporter with your custom values
helm install nginx-exporter prometheus-community/prometheus-nginx-exporter \
  -f nginx-exporter-values.yaml \
  -n default


NGINX Ingress exporter:
The Ingress-Nginx Controller should already be deployed according to the deployment
helm upgrade ingress-nginx ingress-nginx \
--repo https://kubernetes.github.io/ingress-nginx \
--namespace ingress-nginx \
--set controller.metrics.enabled=true \
--set-string controller.podAnnotations."prometheus\.io/scrape"="true" \
--set-string controller.podAnnotations."prometheus\.io/port"="10254"

helm get values ingress-nginx --namespace ingress-nginx


blackbox exporter:
The blackbox exporter allows blackbox probing of endpoints over HTTP, HTTPS, DNS, TCP, ICMP and gRPC.
https://github.com/prometheus/blackbox_exporter

uncomment in prom-custom-values.yaml
```
scrape_configs:
  - job_name: 'blackbox'

```
helm upgrade prometheus prometheus-community/prometheus -f prom-custom-values.yaml -n default


# PHP‑FPM Exporter
git clone https://github.com/brunowego/prometheus-php-fpm-exporter.git
cd prometheus-php-fpm-exporter

helm install php-fpm-exporter . -f values-exporter.yaml -n default

prometheus job: > prom-custom-values.yaml
extraScrapeConfigs: |
- job_name: 'wordpress-phpfpm'
  metrics_path: /metrics
  static_configs:
    - targets:
        - <php-fpm-svc>.default.svc.cluster.local:9253

helm upgrade prometheus prometheus-community/prometheus -f prom-custom-values.yaml -n default