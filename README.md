# CockroachDB TPCC Benchmark on EKS

1. First set some variables.
```
export aws_region="eu-west-1"
export aws_vpc_cidr="10.2.0.0/16"
export clus2="mb-tpcc-cluster-1"
export kubernetes_version=1.21
```

2. Create you eks cluster of the desired size here I am using three instances of type `c5d.4xlarge`.
```
eksctl create cluster \
--name $clus2 \
--nodegroup-name standard-workers \
--node-type c5d.4xlarge \
--nodes 3 \
--region $aws_region \
--vpc-cidr $aws_vpc_cidr \
--version $kubernetes_version
```


3. Download the stateful set configuration and edit to meet your requirements.
```
curl -O https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/bring-your-own-certs/cockroachdb-statefulset.yaml
```

4. Make this directories for our certificates.
```
mkdir certs my-safe-directory
```

5. Use the cockroach CLI to create the ca.
```
cockroach cert create-ca --certs-dir=certs --ca-key=my-safe-directory/ca.key
```

6. Create a client certificate and key pair for the root user.
```
cockroach cert create-client \
root \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

7. Upload the client certificate and key to the Kubernetes cluster as a secret.
```
kubectl create secret \
generic cockroachdb.client.root \
--from-file=certs
```

8. Create the certificate and key pair for your CockroachDB nodes.
```
cockroach cert create-node \
localhost 127.0.0.1 \
cockroachdb-public \
cockroachdb-public.default \
cockroachdb-public.default.svc.cluster.local \
'*.cockroachdb' \
'*.cockroachdb.default' \
'*.cockroachdb.default.svc.cluster.local' \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

9. Upload the node certificate and key to the Kubernetes cluster as a secret.
```
kubectl create secret \
generic cockroachdb.node \
--from-file=certs
```

10. Create the kubernetes resources.
```
kubectl create -f cockroachdb-statefulset.yaml
kubectl create -f aws-svc-admin-ui.yaml
```

11. Run cockroach init on one of the pods to complete the node startup process and have them join together as a cluster.
```
kubectl exec -it cockroachdb-0 \
-- /cockroach/cockroach init \
--certs-dir=/cockroach/cockroach-certs
```

12. To use the CockroachDB SQL client, first launch a secure pod running the cockroach binary.
```
kubectl create -f https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/bring-your-own-certs/client.yaml
```

```
kubectl exec -it cockroachdb-client-secure \
-- ./cockroach sql \
--certs-dir=/cockroach-certs \
--host=cockroachdb-public
```

13. Create a user to log in to the UI with.
```
CREATE USER craig WITH PASSWORD 'cockroach';

GRANT admin TO craig;
\q
```
14. Now shell into the secure client again but this time just with `sh` as we need to pull down our dataset form the internet.

```
kubectl exec -it cockroachdb-client-secure -- sh
```


15. Download the CockroachDB archive for Linux, extract the binary, and copy it into the PATH.
```
curl https://binaries.cockroachdb.com/cockroach-v22.1.1.linux-amd64.tgz | tar -xz
```
```
cp -i cockroach-v22.1.1.linux-amd64/cockroach /usr/local/bin/
```
If you get a permissions error, prefix the command with sudo.

16. Import the TPC-C dataset.
```
cockroach workload fixtures import tpcc --warehouses=2500 'postgresql://root@cockroachdb-0.cockroachdb:26257?sslmode=verify-full&sslrootcert=/cockroach-certs/ca.crt&sslcert=/cockroach-certs/client.root.crt&sslkey=/cockroach-certs/client.root.key'
```

Example Output:
```
I220616 09:49:47.527128 1 ccl/workloadccl/fixture.go:318  [-] 1  starting import of 9 tables
I220616 09:49:49.874109 42 ccl/workloadccl/fixture.go:483  [-] 2  imported 2.5 MiB in district table (25000 rows, 0 index entries, took 2.27973529s, 1.08 MiB/s)
I220616 09:49:50.389726 41 ccl/workloadccl/fixture.go:483  [-] 3  imported 133 KiB in warehouse table (2500 rows, 0 index entries, took 2.795448522s, 0.05 MiB/s)
I220616 09:49:50.490800 47 ccl/workloadccl/fixture.go:483  [-] 4  imported 7.9 MiB in item table (100000 rows, 0 index entries, took 2.896393934s, 2.72 MiB/s)
I220616 09:50:15.562040 46 ccl/workloadccl/fixture.go:483  [-] 5  imported 340 MiB in new_order table (22500000 rows, 0 index entries, took 27.967682747s, 12.16 MiB/s)
I220616 09:52:52.223157 45 ccl/workloadccl/fixture.go:483  [-] 6  imported 4.1 GiB in order table (75000000 rows, 75000000 index entries, took 3m4.628814502s, 22.55 MiB/s)
I220616 09:53:10.150226 44 ccl/workloadccl/fixture.go:483  [-] 7  imported 5.4 GiB in history table (75000000 rows, 0 index entries, took 3m22.555905007s, 27.20 MiB/s)
I220616 09:59:07.335145 43 ccl/workloadccl/fixture.go:483  [-] 8  imported 43 GiB in customer table (75000000 rows, 75000000 index entries, took 9m19.740752911s, 78.92 MiB/s)
I220616 10:02:36.886581 49 ccl/workloadccl/fixture.go:483  [-] 9  imported 43 GiB in order_line table (750022630 rows, 0 index entries, took 12m49.292182104s, 57.13 MiB/s)
I220616 10:02:40.529454 48 ccl/workloadccl/fixture.go:483  [-] 10  imported 75 GiB in stock table (250000000 rows, 0 index entries, took 12m52.935031273s, 99.82 MiB/s)
I220616 10:02:40.554137 1 ccl/workloadccl/fixture.go:326  [-] 11  imported 171 GiB bytes in 9 tables (took 12m53.026841034s, 226.76 MiB/s)
I220616 10:02:41.646193 1 ccl/workloadccl/cliccl/fixtures.go:343  [-] 12  fixture is restored; now running consistency checks (ctrl-c to abort)
I220616 10:02:41.682992 1 workload/tpcc/tpcc.go:517  [-] 13  check 3.3.2.1 took 36.744891ms
I220616 10:02:59.700697 1 workload/tpcc/tpcc.go:517  [-] 14  check 3.3.2.2 took 18.017641828s
I220616 10:03:02.729988 1 workload/tpcc/tpcc.go:517  [-] 15  check 3.3.2.3 took 3.02924782s
I220616 10:05:39.619676 1 workload/tpcc/tpcc.go:517  [-] 16  check 3.3.2.4 took 2m36.889645739s
I220616 10:06:42.635846 1 workload/tpcc/tpcc.go:517  [-] 17  check 3.3.2.5 took 1m3.016103174s
I220616 10:10:47.545607 1 workload/tpcc/tpcc.go:517  [-] 18  check 3.3.2.7 took 4m4.909679287s
I220616 10:11:11.850450 1 workload/tpcc/tpcc.go:517  [-] 19  check 3.3.2.8 took 24.304779361s
I220616 10:11:42.313118 1 workload/tpcc/tpcc.go:517  [-] 20  check 3.3.2.9 took 30.46262877s
```

17. Create the a file called `addrs` on the file system with the command below.
```
cat > addrs<<EOF
postgresql://root@cockroachdb-0.cockroachdb:26257?sslmode=verify-full&sslrootcert=/cockroach-certs/ca.crt&sslcert=/cockroach-certs/client.root.crt&sslkey=/cockroach-certs/client.root.key postgresql://root@cockroachdb-1.cockroachdb:26257?sslmode=verify-full&sslrootcert=/cockroach-certs/ca.crt&sslcert=/cockroach-certs/client.root.crt&sslkey=/cockroach-certs/client.root.key postgresql://root@cockroachdb-2.cockroachdb:26257?sslmode=verify-full&sslrootcert=/cockroach-certs/ca.crt&sslcert=/cockroach-certs/client.root.crt&sslkey=/cockroach-certs/client.root.key
EOF
```

18. Run TPC-C for 30 minutes.
```
cockroach workload run tpcc \
--warehouses=2500 \
--ramp=1m \
--duration=30m \
$(cat addrs)
```

Example Output
```
_elapsed_______tpmC____efc__avg(ms)__p50(ms)__p90(ms)__p95(ms)__p99(ms)_pMax(ms)
 1800.0s    28185.8  87.7%   2407.9    570.4   7247.8  15032.4  28991.0 103079.2
 ```