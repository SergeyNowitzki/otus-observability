mimir ruler mimirtools:
https://grafana.com/docs/mimir/latest/manage/tools/mimirtool/#load-rule-group

kubectl run curlpod -n mimir --rm -i -t --restart=Never --image=alpine -- sh
apk add --no-cache curl jq
curl -fLo mimirtool https://github.com/grafana/mimir/releases/latest/download/mimirtool-linux-amd64
chmod +x mimirtool

./mimirtool rules load ./example_rules_one.yaml --address=http://mimir-nginx.mimir.svc.cluster.local --id=prod
or 
./mimirtool rules load ./example_rules_one.yaml ./example_rules_two.yaml

./mimirtool rules delete-namespace my_namespace --address=http://mimir-nginx.mimir.svc.cluster.local --id=prod

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

curl -s -H "X-Scope-OrgID: prod" http://mimir-ruler.mimir.svc.cluster.local:8080/prometheus/api/v1/alerts | jq


aws s3 --profile mimir --endpoint=http://minio.local/ ls s3://mimir-ruler/rules/prod/bXlfbmFtZXNwYWNl/
2025-05-03 02:11:12        119 ZXhhbXBsZQ==

Which is:
    • bXlfbmFtZXNwYWNl = my_namespace (Base64)
    • ZXhhbXBsZQ== = example (Base64)


Prometheus Alerts
https://samber.github.io/awesome-prometheus-alerts/
