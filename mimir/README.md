# Grafana Mimir

This guide covers the installation and configuration of Grafana Mimir, a highly scalable and distributed time series database for Prometheus metrics.
## Prerequisites

- Kubernetes cluster
- Helm 3+
- MinIO (or other S3-compatible storage) already installed: `minio/README.md`
- `kubectl` configured to communicate with your cluster

### 1. Create a dedicated namespace

```bash
kubectl create namespace mimir
```
### 2. Add the Grafana Helm repository
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm search repo mimir
```
### 3. Configure Mimir
Download the default values file and customize it for your environment:
```bash
helm show values grafana/mimir-distributed > mimir-distributed-values.yml
```
Key configuration considerations:
- Set replicas to 1 for testing environments (query_scheduler and querier)
- Disable zoneAwareReplication for test environments
- Configure S3 storage (using MinIO)
- Set appropriate retention periods

### 4. Install Mimir
```bash
helm install mimir -n mimir grafana/mimir-distributed -f mimir-distributed-values.yml --create-namespace

Welcome to Grafana Mimir!
Remote write endpoints for Prometheus or Grafana Agent:
Ingress is not enabled, see the nginx.ingress values.
From inside the cluster:
  http://mimir-nginx.mimir.svc:80/api/v1/push

Read address, Grafana data source (Prometheus) URL:
Ingress is not enabled, see the nginx.ingress values.
From inside the cluster:
  http://mimir-nginx.mimir.svc:80/prometheus

```
For redeployment or upgrades:
```bash
helm upgrade mimir grafana/mimir-distributed -n mimir -f mimir-distributed-values.yml
kubectl rollout status deployment/mimir-nginx -n mimir
```

### 5. Verify the installation
```bash
kubectl -n mimir get pods
NAME                                        READY   STATUS    RESTARTS       AGE
mimir-alertmanager-0                        1/1     Running   0              5d19h
mimir-compactor-0                           1/1     Running   0              5d19h
mimir-distributor-c8b5dbf74-bstgj           1/1     Running   0              5d19h
mimir-ingester-0                            1/1     Running   0              5d19h
mimir-ingester-1                            1/1     Running   0              12d
mimir-ingester-2                            1/1     Running   0              5d19h
mimir-nginx-755dbcc49d-zrl6x                1/1     Running   0              5d19h
mimir-overrides-exporter-6f95754bff-4p265   1/1     Running   1 (6d7h ago)   12d
mimir-querier-7dd75cc78d-pwknn              1/1     Running   0              5d19h
mimir-query-frontend-7b9cd49655-kvks9       1/1     Running   0              5d19h
mimir-query-scheduler-584c88579b-4f9qn      1/1     Running   0              12d
mimir-rollout-operator-64bdb67ffc-h4gcg     1/1     Running   0              5d19h
mimir-ruler-79454d6d96-gvhj5                1/1     Running   0              5d19h
mimir-store-gateway-0                       1/1     Running   0              5d19h
```

## Configuring Prometheus to Write to Mimir
Add the following to your Prometheus configuration `Infrastructure/prom-custom-values.yaml` to send metrics to Mimir:
```yaml
remote_write:
  - url: http://mimir-distributor.mimir.svc.cluster.local:8080/api/v1/push
    headers:
      X-Scope-OrgID: prod
    write_relabel_configs:
      - action: replace
        target_label: site
        replacement: prod
```
Alternatively, we can use the NGINX gateway (in this case):
```yaml
remote_write:
  - url: http://mimir-nginx.mimir.svc.cluster.local/api/v1/push
    headers:
      X-Scope-OrgID: prod
```

## Configuring Grafana
Add Mimir as a Prometheus data source in your Grafana configuration ( `grafana-custom-values.yaml` ):
```yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Mimir-Prod-Tenant
      type: prometheus
      access: proxy
      orgId: 1
      url: http://mimir-nginx.mimir.svc.cluster.local/prometheus
      version: 1
      editable: true
      jsonData:
        httpHeaderName1: "X-Scope-OrgID"
        alertmanagerUid: "alertmanager"
      secureJsonData:
        httpHeaderValue1: "prod"
```
The `X-Scope-OrgID` header which defines a tenant.

Apply the changes:
```bash
# Check configuration
helm upgrade grafana grafana/grafana -n default -f grafana-custom-values.yaml --dry-run

# Apply configuration
helm upgrade grafana -n default grafana/grafana -f grafana-custom-values.yaml
```
## Data Retention
Mimir's data retention can be configured in the `mimir-distributed-values.yml` file:
```yaml
mimir:
  structuredConfig:
    limits:
      compactor_blocks_retention_period: 14d  # 14 days retention
```
For comparison, Prometheus retention settings `Infrastructure/prom-custom-values.yaml`:
```yaml
retention: "1d"  # 1 day retention
retentionSize: "1GB"  # 1GB size limit
```

## Alertmanager Configuration
Mimir includes a built-in Alertmanager that can centralize all alerting. To use it effectively:
1. Disable the Grafana built-in Alertmanager
2. Configure only Mimir Alertmanager

![image](https://github.com/user-attachments/assets/ce51c69b-8208-4d48-883f-be5b5e813e1c)

![image](https://github.com/user-attachments/assets/c5450b9f-cab1-4773-b2ff-d1197103fe32)


### Slack Alerts Configuration
https://grafana.com/blog/2020/02/25/step-by-step-guide-to-setting-up-prometheus-alertmanager-with-slack-pagerduty-and-gmail/
1. Create a Slack webhook:
   - Go to Slack → Administration → Manage apps
   - Search for "Incoming WebHooks" and add it to your workspace
   - Copy the webhook URL

2. Test the webhook:
```bash
curl -X POST -H 'Content-type: application/json' \
     --data '{"text": "Hello, World!"}' \
     https://hooks.slack.com/services/YOUR_WEBHOOK_PATH
```
![image](https://github.com/user-attachments/assets/2b1b614c-f78e-418e-9e97-8cac71226fc9)

3. Configure Alertmanager in `mimir-distributed-values.yml`:
```yaml
alertmanager:
  fallbackConfig: |
    global:
      resolve_timeout: 5m
    route:
      group_by: ['alertname', 'cluster', 'service']
      repeat_interval: 1h
      group_wait: 30s
      group_interval: 5m
      receiver: 'slack-notifications'
      routes:
        - matchers:
            - severity="critical"
          receiver: slack-notifications
        - matchers:
            - severity="warning"
          receiver: telegram-notifications
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - send_resolved: true
        channel: "#alertmanager"
        api_url: "https://hooks.slack.com/services/YOUR_WEBHOOK_PATH"
        title: "{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}"
        text: "{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}"
```
Alerts with a severity of "critical" will be sent to the Slack channel "#alertmanager".

### Telegram Alerts Configuration
To recieve alerts with a severity 'warnings' in telegram we should create a bot and add it to the new channel:
1. Create a Telegram bot and add it to the channel
2. Configure in `mimir-distributed-values.yml` :
```yaml
    receivers:
...
    - name: 'telegram-notifications'
      telegram_configs:
      - chat_id: -111222333
        bot_token: 'Bot_Token'
```

![image](https://github.com/user-attachments/assets/0157d2bc-ee43-45b5-bdb3-6dd5eefd354b)


## Working with Mimir Rules
Mimir supports Prometheus-compatible alerting and recording rules.
**Mimirtools** is used to manage Mimir rules: https://grafana.com/docs/mimir/latest/manage/tools/mimirtool/
Alertrools are defined in `rules` directory. Some alerts are used for testing purposes and they are "always firing".
The Rules are classified by the following criteria:
- `app_alert_rules.yaml` - PHP-FPM alerts
- `blackbox_alert_rules.yaml` - Blackbox alerts
- `database_alert_rules.yaml` - Database alerts
- `example_rules_one.yaml` - Example alerts always firing
- `hardware_alert_rules.yaml` - Hardware alerts describing hardware metrics
- `kubernetes_alert_rules.yaml` - Kubernetes alerts

![image](https://github.com/user-attachments/assets/d14358b9-dd16-46f8-86fe-a9ab99cd8bf2)


1. Spin up a pod to run the `mimirtool` command:
```bash
kubectl run mimirtools -n mimir --rm -i -t --restart=Never --image=alpine -- sh
```
2. Install `mimirtool`:
```bash
apk add --no-cache curl jq
curl -fLo mimirtool https://github.com/grafana/mimir/releases/latest/download/mimirtool-linux-amd64
chmod +x mimirtool
```
3. Load rules:
```bash
./mimirtool rules load ./example_rules_one.yaml --address=http://mimir-nginx.mimir.svc.cluster.local --id=prod
```
4. Delete rules:
```bash
./mimirtool rules delete-namespace my_namespace --address=http://mimir-nginx.mimir.svc.cluster.local --id=prod
```
5. View current rules:
```bash
curl -s -H "X-Scope-OrgID: prod" http://mimir-ruler.mimir.svc.cluster.local:8080/prometheus/api/v1/rules | jq
{
  "status": "success",
  "data": {
    "groups": [
      {
        "name": "example",
        "file": "my_namespace",
        "rules": [
          {
            "name": "job:http_inprogress_requests:sum",
            "query": "sum by (job) (http_inprogress_requests)",
            "labels": {},
            "health": "ok",
            "lastError": "",
            "type": "recording",
            "lastEvaluation": "2025-05-03T00:20:26.835535222Z",
            "evaluationTime": 0.00674672
          }
        ],
        "interval": 300,
        "lastEvaluation": "2025-05-03T00:20:26.835462589Z",
        "evaluationTime": 0.006834254,
        "sourceTenants": null
      }
    ]
  },
  "errorType": "",
  "error": ""
}
```

## ### S3 Storage Structure
Rules and metrics are stored in S3 buckets with the following structure:
```
s3://mimir-ruler/
  ├── alertmanager/
  ├── alerts/
  └── rules/
      └── prod/
          ├── YXBw/  (app in base64)
          ├── YmxhY2tib3g=/  (blackbox in base64)
          ├── ZGF0YWJhc2U=/  (database in base64)
          └── ...

s3://mimir-tsdb/
  ├── __mimir_cluster/
  ├── anonymous/
  └── prod/
```
To decode base64 folder names:
```bash
echo "ZGF0YWJhc2U=" | base64 -d
# Output: database
```
