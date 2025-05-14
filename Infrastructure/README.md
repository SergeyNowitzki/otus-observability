# Kubernetes Observability Infrastructure
This repository contains configuration files and instructions for setting up a complete observability stack in Kubernetes, including Prometheus, Grafana, MinIO, and WordPress with MariaDB.

## Prerequisites
### NTP Synchronization
```bash
apt-get install -y chrony
systemctl enable --now chrony
```
Verify NTP synchronization:
```bash
chronyc -a sources
chronyc -a tracking  # Check the "Last offset" row
```

## Monitoring Stack
### Prometheus Installation
1. Add the Prometheus Helm repository:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm search repo prometheus
helm repo update
```

2. Install Prometheus:
```bash
helm install prometheus prometheus-community/prometheus
```

3. Verify the installation:
```bash
root@node1:~# kubectl get svc | egrep "^prometheus"
prometheus-kube-state-metrics                  ClusterIP   10.233.51.189   <none>        8080/TCP            36d
prometheus-prometheus-node-exporter            ClusterIP   10.233.4.195    <none>        9100/TCP            36d
prometheus-prometheus-pushgateway              ClusterIP   10.233.20.44    <none>        9091/TCP            36d
prometheus-server                              ClusterIP   10.233.30.177   <none>        80/TCP              36d
```
DNS service record: `prometheus.default.svc.cluster.local`

4. Customize Prometheus configuration
```bash
helm show values prometheus-community/prometheus > prom-custom-values.yaml
```

5. Edit the values file to configure node exporter:
```yaml
...
prometheus-node-exporter:
  enabled: true
...
```
```yaml
...
nodeExporter:
  extraArgs:
    - '--path.procfs=/host/proc'
    - '--path.sysfs=/host/sys'
  volumeMounts:
    - name: proc
      mountPath: /host/proc
      readOnly: true
    - name: sys
      mountPath: /host/sys
      readOnly: true
  volumes:
    - name: proc
      hostPath:
        path: /proc
    - name: sys
      hostPath:
        path: /sys
...
```

6. Apply custom configuration:
```bash
helm upgrade prometheus prometheus-community/prometheus -f prom-custom-values.yaml -n default
```

7. Monitor the rollout:
```bash
kubectl rollout status deploy/prometheus-server -n default
```

8. Disable the built-in Alertmanager (since Mimir will be used) in `prom-custom-values.yaml`:
```yaml
alertmanager:
  enabled: false
```

9. Clean up old resources in case the Grafana Alertmanager was installed:
```bash
kubectl delete pvc storage-prometheus-alertmanager-0 -n default
kubectl delete svc prometheus-alertmanager -n default
```

### Grafana Installation
1. Add the Grafana Helm repository:
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm search repo grafana/grafana
```

2. Install Grafana:
```bash
helm install grafana grafana/grafana
```

3. Verify the installation:
```bash
helm list
NAME      	NAMESPACE	REVISION	UPDATED                                  STATUS  	 CHART             APP VERSION
grafana   	default  	1       	2025-04-05 17:26:31.977360633 +0000 UTC  deployed grafana-8.11.3    11.6.0
prometheus	default  	1       	2025-04-04 21:14:50.275440836 +0000 UTC  deployed	 prometheus-27.8.0 v3.2.1

kubectl get all | grep grafana
```
DNS: `grafana.default.svc.cluster.local`

5. Create a NodePort service for external access (otional for testing):
```bash
kubectl expose service grafana --type=NodePort --target-port=3000 --name=grafana-ext 
kubectl get svc
```

6. Retrieve Grafana admin credentials:
```bash
kubectl get secret --namespace default grafana -o yaml
# Decode the password
echo "c3BMTEkzdUxEejdnR3hnUWtVemVBb3p6dk5SWDJuQVUyd1VST3Nzeg==" | openssl base64 -d ; echo
```

7. Enable persistent storage for Grafana:
```bash
helm show values grafana/grafana > grafana-custom-values.yaml
```

8. Edit the values file to enable persistence:
By default, persistent storage is disabled, which means that Grafana uses ephemeral storage, and all data will be stored within the container’s file system. This data will be lost if the container is stopped, restarted, or if the container crashes.
It is highly recommended that you enable persistent storage in Grafana Helm charts if you want to ensure that your data persists and is not lost in case of container restarts or failures.
```yaml
persistence:
  type: pvc
  enabled: true
  # storageClassName: default
```

9. Preview changes (optional):
```bash
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade grafana grafana/grafana -f grafana-custom-values.yaml -n default
```

10. Apply the changes:
```bash
helm upgrade grafana grafana/grafana -f grafana-custom-values.yaml -n default
```

## MinIO Object Storage
`otus-observability/minio/README.md`
MiniO is an open-source object storage server. It will be used as an object storage solution for storing logs, metrics, and traces for Mimir.
MinIO is deployed as a StatefulSet with persistent storage for object storage needs. The configuration includes:
- A single replica for test environments
- Persistent storage via PVC
- Resource limits for CPU and memory
- Health checks via liveness and readiness probes
- Accessible via both API (port 9000) and Console (port 9001)

## Application Stack
### WordPress with MariaDB and Nginx
`infrastructure/wordpress_mariadb_nginx/`
The label is important and will allow our service to correctly identify which pod to route traffic to.
1. Deploy the application stack:
```bash
kubectl apply -f mariadb.yaml
kubectl apply -f wp_manifest/
kubectl apply -f nginx.yaml
```
2. Verify the deployment:
```bash
kubectl logs -f -l app=nginx
```
The **WordPress** setup includes:
- MariaDB database with persistent storage
- WordPress using PHP-FPM
- Nginx as a web server and reverse proxy
- Persistent storage for both database and WordPress files

In `nginx.yaml` configuration file `location /stub_status` is used to expose metrics for NGINX.
If any changes in the config has to be applied, the `kubectl apply -f nginx.yaml` command should be used.
To reload the NGINX pods and apply the config: `kubectl rollout restart deployment nginx`

## Network Configuration
### MetalLB Load Balancer
A LoadBalancer service for Ingress controller (MetalLB).
On many cloud providers, a LoadBalancer service automatically gets an external IP with ports 80/443.
On bare metal or lab clusters MetalLB can be deployed to provide external IPs for services.
MetalLB allows to create a LoadBalancer service that listens on standard ports.

1. Install MetalLB for bare-metal load balancing:
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
```

2. Configure MetalLB with an IP Pool:
- `ipaddresspool.yaml`
- `l2advertisement.yaml`
Then apply it:
```bash
kubectl apply -f ipaddresspool.yaml
kubectl apply -f l2advertisement.yaml
```
The IP pool `192.168.99.200-192.168.99.210` is configured to work with the node subnet `192.168.99.0/24`.

3. Verify the configuration:
```bash
kubectl get configmap config -n metallb-system -o yaml
kubectl get pods -n metallb-system
NAME                          READY   STATUS    RESTARTS        AGE
controller-5456bd6d98-d98n8   1/1     Running   0               6d15h
speaker-dwvmz                 1/1     Running   3 (22d ago)     35d
speaker-qq966                 1/1     Running   6 (6d14h ago)   35d
speaker-vn4x4                 1/1     Running   4 (6d15h ago)   35d

kubectl logs -n metallb-system -l component=controller
```

### Ingress Controller
1. Install the NGINX Ingress Controller with a LoadBalancer service type:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace --set controller.service.type=LoadBalancer
```

2. Deploy the Ingress resource `otus-observability/Infrastructure/ingress/ingress.yaml`:
```bash
kubectl apply -f ingress.yaml
```

3. Verify the Ingress configuration:
```bash
root@node1:~# kubectl get ingress -n default
NAME          CLASS    HOSTS                                                        ADDRESS          PORTS   AGE
lab-ingress   <none>   wordpress.local,prometheus.local,grafana.local + 2 more...   192.168.99.200   80      35d

root@node1:~# kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.233.13.189   192.168.99.200   80:30487/TCP,443:32244/TCP   35d
ingress-nginx-controller-admission   ClusterIP      10.233.22.7     <none>           443/TCP                      35d
ingress-nginx-controller-metrics     ClusterIP      10.233.57.174   <none>           10254/TCP                    26d
```

4. Add host entries to your local machine:
```bash
sudo vim /etc/hosts
192.168.99.200 wordpress.local prometheus.local grafana.local minio.local minio-console.local
```

5. Test the Ingress configuration:
```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
```
```
root@node1:~# kubectl get svc
NAME                                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
grafana                               ClusterIP   10.233.48.143   <none>        80/TCP         25h
prometheus-server                     ClusterIP   10.233.30.177   <none>        80/TCP         45h
wordpress                             ClusterIP   10.233.34.247   <none>        80/TCP         7h23m
```

6. Restart ingres pods if needed(optional):
```bash
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
```

7. Validate default backends:
```bash
kubectl describe ingress lab-ingress -n default
Name:             lab-ingress
Labels:           <none>
Namespace:        default
Address:          192.168.99.200
Ingress Class:    <none>
Default backend:  <default>
Rules:
  Host                 Path  Backends
  ----                 ----  --------
  wordpress.local
                       /   nginx:80 (10.233.71.63:80,10.233.71.62:80,10.233.71.2:80)
  prometheus.local
                       /   prometheus-server:80 (10.233.71.4:9090)
  grafana.local
                       /   grafana:80 (10.233.75.12:3000)
  minio.local
                       /   minio:9000 (10.233.75.5:9000)
  minio-console.local
                       /   minio:9001 (10.233.75.5:9001)
Annotations:           kubernetes.io/ingress.class: nginx
                       nginx.ingress.kubernetes.io/proxy-body-size: 0
                       nginx.ingress.kubernetes.io/proxy-connect-timeout: 600
                       nginx.ingress.kubernetes.io/proxy-read-timeout: 600
                       nginx.ingress.kubernetes.io/proxy-send-timeout: 600
Events:                <none>
```

## Accessing Services
- WordPress: http://wordpress.local/wp-admin/install.php
- Prometheus: http://prometheus.local/
- Grafana: http://grafana.local/
- MinIO API: http://minio.local/
- MinIO Console: http://minio-console.local/

## Remote Write Configuration
Prometheus is configured to remote write metrics to Mimir. The header `X-Scope-OrgID` is used to specify the tennant ID.:
```yaml
...
remoteWrite:
  - url: http://mimir-nginx.mimir.svc.cluster.local/api/v1/push
    headers:
      X-Scope-OrgID: prod
...
```

## Monitoring Resources
1. Check resource usage:
```bash
kubectl top node
kubectl top pod -n default
```
2. View Prometheus configuration:
```bash
kubectl get configmap prometheus-server -o yaml
```