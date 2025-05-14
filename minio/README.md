# MinIO for Grafana Mimir
This document describes the MinIO setup used as object storage for Grafana Mimir in our observability stack.

## Overview
MinIO provides S3-compatible object storage that Grafana Mimir uses for long-term storage of metrics, rules, and alerts. Our setup uses a single-node MinIO instance with persistent storage.

## Installation

### 1. Deploy MinIO
```bash
# Apply the MinIO configuration
kubectl apply -f minio.yaml -n default
```

### 2. Verify the deployment
```bash
root@node1:~# kubectl get pods -n default -l app=minio
NAME      READY   STATUS    RESTARTS   AGE
minio-0   1/1     Running   0          8d
```

### 3. Configure Ingress
Ensure the ingress configuration `infrastructure/ingress/ingress.yaml` includes entries for MinIO hosts: 
```yaml
- host: minio.local
- host: minio-console.local
```
<img width="1756" alt="image" src="https://github.com/user-attachments/assets/44ce01fe-8258-48c0-b8bd-ed97f400f501" />

## Configuring AWS CLI for MinIO
### 1. Install AWS CLI
https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

### 2. Configure AWS credentials
Create a file `~/.aws/credentials` with the following content:
```ini
[mimir]
aws_access_key_id=access_key_id_value
aws_secret_access_key=secret_access_key_value
```
### 3. Test the connection
```bash
aws s3 ls --profile mimir --endpoint=http://minio.local/
2025-04-27 23:06:37 mimir-ruler
2025-04-27 23:06:57 mimir-tsdb
```
Buckets should be correlated with the config `mimir/mimir-distributed-values.yml`

## Bucket Structure for Mimir
Mimir uses two main buckets:

### 1. mimir-ruler : Stores alerting and recording rules
```bash
aws s3 --profile mimir --endpoint=http://minio.local/ ls s3://mimir-ruler
                           PRE alertmanager/
                           PRE alerts/
                           PRE rules/
```
### 2. mimir-tsdb : Stores time series data
```bash
aws s3 --profile mimir --endpoint=http://minio.local/ ls s3://mimir-tsdb
                           PRE __mimir_cluster/
                           PRE anonymous/
                           PRE prod/
```

## Tenant Structure
Mimir uses multi-tenancy, with each tenant having its own directory:
```bash
aws s3 --profile mimir --endpoint=http://minio.local/ ls s3://mimir-ruler/rules/prod/
                           PRE YXBw/
                           PRE YmxhY2tib3g=/
                           PRE ZGF0YWJhc2U=/
                           PRE a3ViZXJuZXRlcw==/
                           PRE aGFyZHdhcmU=/
                           PRE dGVzdF9uYW1lc3BhY2U=/
```
Note: The folder names are base64 encoded. To decode them:
```bash
echo "ZGF0YWJhc2U=" | base64 -d
database
```

## Mimir Configuration for MinIO
In the Mimir configuration ( `mimir/mimir-distributed-values.yml` ), ensure the following settings:
```yaml
...
mimir:
  structuredConfig:
    common:
      storage:
        backend: s3
        s3:
          access_key_id: access_key_id_value
          bucket_name: mimir-ruler
          endpoint: minio.default.svc.cluster.local:9000
          insecure: true
          secret_access_key: secret_access_key_value
    blocks_storage:
      s3:
        bucket_name: mimir-tsdb

    alertmanager_storage:
      s3:
        bucket_name: mimir-ruler

    ruler_storage:
      s3:
        bucket_name: mimir-ruler
...
```
The full DNS name for services `minio.mimir.svc.cluster.local`

To verify that Mimir is correctly storing data:
```bash
# Check ruler bucket
aws s3 --profile mimir --endpoint=http://minio.local/ ls s3://mimir-ruler --recursive | head

# Check TSDB bucket
aws s3 --profile mimir --endpoint=http://minio.local/ ls s3://mimir-tsdb --recursive | head
```
