we can use custom s3 service:
minio-local:9001/login

buckets should be correlated with the config

kubectl -n kube-system get svc | grep -E 'dns|coredns'
coredns          ClusterIP   10.233.0.3    <none>        53/UDP,53/TCP,9153/TCP   42d

coredns.kube-system.svc.cluster.local

we can use custom s3 service:
minio-local:9001/login

buckets should be correlated with the config
`mimir/mimir-distributed-values.yml`

awscli should be installed to connect to minio s3:
aws s3 ls --profile mimir --endpoint=http://minio.local/
2025-04-27 23:06:37 mimir-ruler
2025-04-27 23:06:57 mimir-tsdb