minio should have been already installed.

helm chart: https://grafana.com/docs/helm-charts/mimir-distributed/latest/get-started-helm-charts/

grafana mimir will be as a datasource in grafana

kubectl create namespace mimir

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm search repo mimir

helm show values grafana/mimir-distributed > mimir-values.yml
adjust to our needs

pay attention to the default replicas - 1 is enough for testing:
query_scheduler and querier
zoneaware replication - we can disable it in test env
if set minio false it wont be installed, but we have to define S3 buckets in config instead


helm -n mimir install mimir grafana/mimir-distributed -f mimir-disrtibuted-values-new.yml

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


helm install mimir -n mimir grafana/mimir-distributed -f mimir-scenario/mimir-distributed-values.yml --create-namespace
redeployment:
helm upgrade mimir grafana/mimir-distributed -n mimir -f mimir-distributed-values.yml
kubectl rollout status deployment/mimir-nginx -n mimir

after mimr installed message:
```Welcome to Grafana Mimir!
Remote write endpoints for Prometheus or Grafana Agent:
Ingress is not enabled, see the nginx.ingress values.
From inside the cluster:
  http://mimir-nginx.mimir.svc:80/api/v1/push

Read address, Grafana data source (Prometheus) URL:
Ingress is not enabled, see the nginx.ingress values.
From inside the cluster:
  http://mimir-nginx.mimir.svc:80/prometheus```

datasource values should be added to `grafana-custom-values.yaml`
`X-Scope-OrgID` header defines a tenant.

to check updates:
helm upgrade grafana grafana/grafana -n default  -f grafana-custom-values.yaml --dry-run

upgrade config and apply:
helm upgrade grafana -n default grafana/grafana -f grafana-custom-values.yaml

Point Prometheus at Mimir
remote_write:
  - url: http://mimir-distributor.mimir.svc.cluster.local:8080/api/v1/push
    headers:
      X-Scope-OrgID: prod
    write_relabel_configs:
      - action: replace        # “replace” means “add or overwrite”
        target_label: site
        replacement: prod


If you prefer to go through the NGINX gateway, use
http://mimir-nginx.mimir.svc.cluster.local/api/v1/push.

in the backets come up folders with metrics.

## Prometheus data retention period (default if not specified is 15 days)
  ##
  retention: "1d"

  ## Prometheus' data retention size. Supported units: B, KB, MB, GB, TB, PB, EB.
  ##
  retentionSize: "1GB"



Alertmanager
So, to cleanly centralize all your alerting in Mimir:
	1. Disable the Grafana built-in Alertmanager.
	2. Leave only your Mimir Alertmanager entry enabled.

How to set up Slack alerts:
https://grafana.com/blog/2020/02/25/step-by-step-guide-to-setting-up-prometheus-alertmanager-with-slack-pagerduty-and-gmail/
To set up alerting in your Slack workspace, you’re going to need a Slack API URL. Go to Slack -> Administration -> Manage apps.

In the Manage apps directory, search for Incoming WebHooks and add it to your Slack workspace.

Test the link:
curl -X POST -H 'Content-type: application/json' \
     --data '{"text": "Hello, World!"}' \
     https://hooks.slack.com/services/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


To recieve 'warnings' in telegram we should create a bot and add it to the new channel.

alertmanager is configured in `mimir-distributed-values.yml` in alertmanager section.

