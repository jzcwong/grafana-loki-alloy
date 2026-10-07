# Cross-cluster pod logs: Grafana Alloy → Grafana Loki → Grafana on EKS (S3-backed)

This demo shows **Grafana Alloy** running on one EKS cluster collecting pod logs and
shipping them to **Grafana Loki** running on a different EKS cluster, with Loki storing
its chunks, index and ruler data in **Amazon S3**. **Grafana** runs on the same cluster
as Loki and queries it for visualization.

> **POC / demo only.** Loki is deployed in *monolithic* (SingleBinary) mode with a
> single replica and replication factor 1, and is exposed over plain HTTP without
> authentication. Grafana is a single replica reached through `kubectl port-forward`.
> This is intended to demonstrate the log flow, not to serve as a
> production setup. For production, use a distributed Loki deployment and the
> improvements listed under [Future improvements](#future-improvements).

## Architecture

```
 demo-vpc (10.0.0.0/16, us-west-2)
┌──────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│  demo-argo-cluster                        demo-eks-cluster                       │
│  ┌─────────────────────────┐              ┌──────────────────────────────────┐   │
│  │ node                    │              │ namespace: loki                  │   │
│  │  /var/log/pods/*  ──┐   │              │                                  │   │
│  │                     ▼   │   HTTP push  │  ┌──────────────┐   ┌──────────┐ │   │
│  │  Alloy (DaemonSet) ─────┼──► internal ─┼─►│ loki-gateway │──►│  loki-0  │ │   │
│  │  ns: alloy              │      NLB     │  │   (nginx)    │   │ (single  │ │   │
│  └─────────────────────────┘    :80       │  └──────▲───────┘   │  binary) │ │   │
│                                           │         │           └────┬─────┘ │   │
│                                           │╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌│   │
│                                           │ namespace: grafana       │       │   │
│                                           │         │ LogQL (HTTP)   │       │   │
│                                           │  ┌──────┴───────┐        │       │   │
│                                           │  │   Grafana    │        │       │   │
│                                           │  └──────▲───────┘        │       │   │
│                                           └─────────┼────────────────┼───────┘   │
└─────────────────────────────────────────────────────┼────────────────┼───────────┘
                                                      │                │ IRSA
                                     kubectl          │  ┌─────────────┴─────────────┐
                                   port-forward ──────┘  │ S3: <CHUNKS_BUCKET>       │
                                   :3000 (browser)       │     (chunks + TSDB index) │
                                                         │ S3: <RULER_BUCKET>        │
                                                         │     (ruler)               │
                                                         └───────────────────────────┘
```

- Both clusters share the same VPC, so no VPC peering or Transit Gateway is needed.
- Loki's gateway is exposed through an **internal** NLB that only accepts traffic from
  the VPC CIDR.
- Loki accesses S3 through an IAM role bound to its service account (IRSA). No access
  keys are stored in the Helm values.
- Alloy adds a `cluster="demo-argo-cluster"` label to every log line so logs from
  different source clusters can be told apart in Loki.
- Grafana queries Loki in-cluster through the `loki-gateway` Service, not the NLB. It
  has no load balancer of its own; you reach it with `kubectl port-forward`, which goes
  through the EKS API server.

## Files

| File | Purpose |
|---|---|
| [`loki-mono-aws.yaml`](loki-mono-aws.yaml) | Helm values used in this demo: Loki SingleBinary on `demo-eks-cluster`, S3 storage, IRSA, internal NLB. |
| [`alloy-argo.yaml`](alloy-argo.yaml) | Helm values used in this demo: Alloy DaemonSet on `demo-argo-cluster` that tails pod logs and pushes to Loki. |
| [`grafana-eks.yaml`](grafana-eks.yaml) | Helm values used in this demo: minimal Grafana on `demo-eks-cluster` with Loki provisioned as the default data source. |
| [`storage-class.yaml`](storage-class.yaml) | Default encrypted `gp3` StorageClass for `demo-eks-cluster` (EBS CSI driver). |

The committed values files contain placeholders such as `<ACCOUNT_ID>`. To deploy, copy
each one to a `.local.yaml` file (for example `cp loki-mono-aws.yaml loki-mono-aws.local.yaml`),
fill in your values there and pass that file to `-f`. `.gitignore` excludes `*.local.yaml`
so your real values are never committed.

## Environment

| Item | Value |
|---|---|
| AWS region | `us-west-2` |
| Loki + Grafana cluster | `demo-eks-cluster` (2 × t3.medium Spot, private subnets, fixed at 2 nodes) |
| Log source cluster | `demo-argo-cluster` (Bottlerocket, public subnets) |
| Loki Helm chart | `grafana/loki` 7.3.0 (Loki 3.6.x) |
| Grafana Helm chart | `grafana-community/grafana` 13.2.7 (Grafana 13.2.3) |
| Alloy Helm chart | `grafana/alloy` 1.13.0 (Alloy v1.20.0) |
| S3 buckets | `<CHUNKS_BUCKET>` (chunks), `<RULER_BUCKET>` (ruler) |

## Prerequisites

- `aws` CLI, `kubectl` and `helm` installed, with access to both clusters.
- Kubeconfig contexts for both clusters, with their names kept in variables. The
  commands in this guide always pass an explicit context so nothing is installed on
  the wrong cluster:

  ```sh
  aws eks update-kubeconfig --region us-west-2 --name demo-eks-cluster
  aws eks update-kubeconfig --region us-west-2 --name demo-argo-cluster

  EKS=arn:aws:eks:us-west-2:<ACCOUNT_ID>:cluster/demo-eks-cluster
  ARGO=arn:aws:eks:us-west-2:<ACCOUNT_ID>:cluster/demo-argo-cluster
  ```

- The two S3 buckets already created in the same region as the clusters.
- An IAM OIDC provider associated with `demo-eks-cluster` (required for IRSA).
- The **Amazon EBS CSI driver** add-on (`aws-ebs-csi-driver`) enabled on
  `demo-eks-cluster`. Loki's persistent volume uses the `gp3` StorageClass, and
  without the driver the PVC stays `Pending` and `loki-0` never starts.
- A default `gp3` StorageClass on `demo-eks-cluster`, which `loki-mono-aws.yaml`
  expects. Once the EBS add-on is installed, create it with
  [`storage-class.yaml`](storage-class.yaml):

  ```sh
  kubectl --context $EKS apply -f storage-class.yaml
  kubectl --context $EKS get storageclass
  ```

  gp3 is cheaper than gp2 and has a fixed baseline of 3000 IOPS and 125 MB/s
  regardless of volume size. The class encrypts volumes, allows volume expansion and
  uses `reclaimPolicy: Retain`, so the EBS volume is kept after its PVC is deleted
  and has to be removed by hand. If `gp2` is also marked `(default)`, remove its
  annotation so there is only one default:

  ```sh
  kubectl --context $EKS annotate storageclass gp2 \
    storageclass.kubernetes.io/is-default-class-
  ```

  If you'd rather keep using `gp2`, set `singleBinary.persistence.storageClass: gp2`
  in `loki-mono-aws.yaml` instead.
- An IAM role for Loki (in this demo `demo-lk-role`) with:
  - a trust policy allowing `system:serviceaccount:loki:loki` via the cluster's OIDC
    provider. Get `<OIDC_ID>` from the part of the issuer URL after `/id/`:

    ```sh
    aws eks describe-cluster --name demo-eks-cluster --region us-west-2 \
      --query 'cluster.identity.oidc.issuer' --output text
    ```

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Principal": {
          "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/<OIDC_ID>"
        },
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
          "StringEquals": {
            "oidc.eks.us-west-2.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:loki:loki",
            "oidc.eks.us-west-2.amazonaws.com/id/<OIDC_ID>:aud": "sts.amazonaws.com"
          }
        }
      }]
    }
    ```

    The `sub` value has to match Loki's service account. It's `loki` when the Helm
    release is named `loki`. If you use a different release name, check the actual name
    with `kubectl -n loki get sa`.

  - this S3 permissions policy:

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Action": ["s3:ListBucket", "s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
        "Resource": [
          "arn:aws:s3:::<CHUNKS_BUCKET>", "arn:aws:s3:::<CHUNKS_BUCKET>/*",
          "arn:aws:s3:::<RULER_BUCKET>", "arn:aws:s3:::<RULER_BUCKET>/*"
        ]
      }]
    }
    ```

Add the Helm chart repositories:

```sh
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
```

The Grafana chart in the `grafana` repo is deprecated and no longer updated; it moved
to `grafana-community`. Loki and Alloy still come from the `grafana` repo.

## Deployment

### 1. Deploy Loki on `demo-eks-cluster`

In `loki-mono-aws.yaml`, replace `<ACCOUNT_ID>`, `<CHUNKS_BUCKET>` and `<RULER_BUCKET>`
and make sure the region and IAM role name match your environment, then install:

```sh
helm install loki grafana/loki -n loki --create-namespace --version 7.3.0 \
  -f loki-mono-aws.yaml --kube-context $EKS

kubectl --context $EKS -n loki get pods
```

Expected pods: `loki-0`, `loki-gateway-*`, `loki-chunks-cache-0`,
`loki-results-cache-0` and one `loki-canary-*` per node, all `Running`.

Key settings in `loki-mono-aws.yaml`:

- `deploymentMode: SingleBinary` with `singleBinary.replicas: 1` and
  `replication_factor: 1`. The chart places one Loki pod per node, so more replicas need
  more nodes.
- `singleBinary.resources` requests 250m CPU / 1Gi memory (2Gi limit). Without a
  request the scheduler doesn't count Loki's memory and may pack other pods onto its
  node, and Loki's memory grows once Grafana starts sending queries.
- `loki.storage` points chunks and ruler storage at the two S3 buckets. Credentials
  come from the IRSA annotation on the service account.
- `chunksCache.allocatedMemory: 1024` replaces the chart default of 8192 MB (a ~9.8Gi
  pod request), which doesn't fit on t3.medium nodes.
- 28-day retention, enforced by the compactor.
- `gateway.service` is `type: LoadBalancer` with annotations that make EKS's built-in
  service controller create an **internal NLB**. Source ranges are set with the
  `service.beta.kubernetes.io/load-balancer-source-ranges` annotation because chart
  7.3.0 doesn't render `spec.loadBalancerSourceRanges` for the gateway.

### 2. Get the Loki push endpoint

```sh
LOKI_HOST=$(kubectl --context $EKS -n loki get svc loki-gateway \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "http://$LOKI_HOST/loki/api/v1/push"
```

Put this URL in the `loki.write` block in `alloy-argo.yaml`, replacing
`<LOKI_NLB_HOSTNAME>` with `$LOKI_HOST`. The NLB hostname changes
if the `loki-gateway` service is ever deleted and recreated.

### 3. Check connectivity from the source cluster

```sh
kubectl --context $ARGO run loki-check --rm -i --restart=Never \
  --image=curlimages/curl:8.10.1 -- \
  curl -s -m 5 http://$LOKI_HOST/loki/api/v1/status/buildinfo
```

Getting Loki's version JSON back means the network path works.

### 4. Deploy Alloy on `demo-argo-cluster`

```sh
helm install alloy grafana/alloy -n alloy --create-namespace --version 1.13.0 \
  -f alloy-argo.yaml --kube-context $ARGO

kubectl --context $ARGO -n alloy get pods -o wide
```

The Alloy pipeline in `alloy-argo.yaml`:

1. `discovery.kubernetes` finds running pods on the local node (`K8S_NODE_NAME`).
2. `discovery.relabel` adds `namespace`, `pod`, `container`, `app`, `node` and `job`
   labels and builds the `/var/log/pods/...` file path.
3. `loki.source.file` tails the log files (the chart mounts host `/var/log` into the
   pod).
4. `loki.process` with `stage.cri` strips the containerd log prefix.
5. `loki.write` pushes to Loki through the internal NLB with the external label
   `cluster="demo-argo-cluster"`.

On first start, Alloy reads existing log files from the beginning. Loki rejects lines
older than 7 days (`reject_old_samples_max_age`), so you'll see a burst of
`400 ... timestamp too old` errors in the Alloy logs. These stop within seconds. Alloy
records how far it has read in each file, so restarts don't resend old lines.

To change the config later, edit `alloy-argo.yaml` and run the same command with
`helm upgrade` instead of `helm install`.

### 5. Deploy Grafana on `demo-eks-cluster`

If Loki was installed before `singleBinary.resources` was added to
`loki-mono-aws.yaml`, apply it first. This restarts `loki-0`; ingestion pauses for
about a minute while Alloy retries, and Loki replays its write-ahead log on startup:

```sh
helm upgrade loki grafana/loki -n loki --version 7.3.0 \
  -f loki-mono-aws.yaml --kube-context $EKS
kubectl --context $EKS -n loki rollout status statefulset/loki
```

Install Grafana:

```sh
helm install grafana grafana-community/grafana -n grafana --create-namespace \
  --version 13.2.7 -f grafana-eks.yaml --kube-context $EKS

kubectl --context $EKS -n grafana get pods,pvc
```

Expected: one `grafana-*` pod `Running` and the `grafana` PVC `Bound`.

Key settings in `grafana-eks.yaml`:

- `datasources` provisions Loki at startup as the default data source (`uid: loki`),
  pointing at `http://loki-gateway.loki.svc.cluster.local`. No `X-Scope-OrgID` header
  is needed because Loki runs with `auth_enabled: false`.
- `resources` requests 100m CPU / 128Mi memory with a 512Mi limit.
- `persistence` keeps Grafana's SQLite database (saved dashboards, users) on a 2Gi
  `gp3` volume, so it survives pod restarts and Spot node replacements.
- `service.type: ClusterIP`: Grafana has no load balancer.

Log in:

```sh
kubectl --context $EKS -n grafana get secret grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d; echo

kubectl --context $EKS -n grafana port-forward svc/grafana 3000:80
```

Open <http://localhost:3000> and sign in as `admin` with the password above. The
password is generated at install time and stored in the `grafana` Secret.

#### Capacity

Measured on the 2 × t3.medium nodes (1930m CPU / ~3.2Gi memory allocatable each)
after this step:

| Node | Main pods | CPU requested | Memory requested |
|---|---|---|---|
| `ip-10-0-142-156` | grafana, loki-results-cache, coredns | 1040m (53%) | 1833Mi (55%) |
| `ip-10-0-157-204` | loki-0, loki-chunks-cache, loki-gateway | 990m (51%) | 2589Mi (78%) |

This is fine for a demo. The node group has a fixed size and no autoscaler, so if you
need heavier querying (wide time ranges, many dashboards), add a node or move to
t3.large. Otherwise raising Loki's memory request may leave it unschedulable.

## Verification

### End-to-end test

Write a unique line from a pod on the source cluster and look for it in Loki:

```sh
M="e2e-marker-$(date +%s)"
kubectl --context $ARGO run log-marker --restart=Never --image=busybox:1.36 \
  -- sh -c "echo $M; sleep 30"

sleep 15
kubectl --context $EKS -n loki port-forward svc/loki-gateway 8080:80 &
curl -s -G localhost:8080/loki/api/v1/query_range \
  --data-urlencode "query={cluster=\"demo-argo-cluster\", pod=\"log-marker\"} |= \"$M\"" \
  --data-urlencode since=5m

kubectl --context $ARGO delete pod log-marker
```

If the marker line comes back, the full path works: pod log file → Alloy → NLB →
Loki.

### Sender side (Alloy metrics)

```sh
kubectl --context $ARGO -n alloy port-forward ds/alloy 12345:12345 &
curl -s localhost:12345/metrics | \
  grep -E '^loki_write_(sent_entries_total|dropped_entries_total|request_duration_seconds_count)'
```

- `loki_write_sent_entries_total` and `..._count{status_code="204"}` should keep
  increasing.
- `loki_write_dropped_entries_total` and `status_code="400"` should stay flat after the
  initial backlog.
- `http://localhost:12345` serves the Alloy UI, which shows the health of each
  component in the pipeline.

### Receiver side (Loki)

```sh
# Namespaces from the source cluster seen recently
curl -s -G localhost:8080/loki/api/v1/label/namespace/values \
  --data-urlencode 'query={cluster="demo-argo-cluster"}' --data-urlencode since=15m

# Ingestion and rejection counters
kubectl --context $EKS -n loki port-forward pod/loki-0 3100:3100 &
curl -s localhost:3100/metrics | \
  grep -E '^loki_(distributor_lines_received_total|discarded_samples_total)'
```

`loki_discarded_samples_total{reason="greater_than_max_sample_age"}` reflects the
startup backlog and should stay flat. Growth, or a new `reason` such as `rate_limited`,
means Loki is rejecting live logs.

Useful LogQL queries:

```logql
{cluster="demo-argo-cluster"}
{cluster="demo-argo-cluster", namespace="argocd"}
sum by (namespace) (count_over_time({cluster="demo-argo-cluster"}[5m]))
```

### Grafana

With the port-forward from step 5 running:

```sh
GRAFANA_PW=$(kubectl --context $EKS -n grafana get secret grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d)

# Grafana itself
curl -s localhost:3000/api/health

# Grafana → Loki connectivity; expect "Data source successfully connected."
curl -s -u admin:$GRAFANA_PW localhost:3000/api/datasources/uid/loki/health
```

In the UI, go to **Explore**, pick **Loki**, and run the LogQL queries above. To check
the read path without depending on Alloy, query Loki's own canary, which writes a
line every second from each node: `{pod=~"loki-canary.*"}`.

## Known limitations

- **No high availability.** One Loki replica with replication factor 1: restarting
  `loki-0` briefly pauses ingestion. Data already received is kept, because Loki
  replays its write-ahead log on restart.
- **No authentication or TLS.** `auth_enabled: false` (single tenant `fake`) and the
  gateway has no basic auth. Access is restricted only by the internal NLB and the
  `10.0.0.0/16` source range on the node security group.
- **Built-in service controller.** The NLB is created by EKS's built-in controller
  (instance targets via NodePort). Because the subnets aren't tagged for load balancer
  discovery, it placed one NLB node in a public subnet. The NLB is still internal and
  has no public IP.
- **Hard-coded endpoint.** The NLB hostname is hard-coded in `alloy-argo.yaml`.
- **Pod logs only.** Kubernetes events and node/system logs are not collected.
- **Grafana is single-replica with local SQLite.** Its `gp3` volume is tied to one
  Availability Zone; after a Spot interruption the pod waits until a node is back in
  that zone. Only the generated `admin` account exists, and there's no SSO.
- **Tight capacity.** The two t3.medium nodes are about half to three-quarters
  requested (see [Capacity](#capacity)), with no autoscaler.

## Future improvements

- [ ] **Install the AWS Load Balancer Controller** and expose the Loki gateway through a
      properly managed NLB (IP targets, explicit subnets, optional TLS via ACM), instead
      of EKS's built-in service controller.
- [ ] **Add authentication to Loki's load balancer**, for example gateway basic auth
      (`gateway.basicAuth` in the Loki chart), with Alloy sending the credentials from a
      Kubernetes Secret.
- [ ] **Tag the public and private subnets properly** (`kubernetes.io/role/elb=1` on
      public, `kubernetes.io/role/internal-elb=1` on private) so load balancers land in
      the intended subnets.
- [x] **Install Grafana** for visualization, with Loki as a data source.
- [ ] Expose Grafana for shared access (internal NLB or ALB with TLS) with SSO or at
      least a non-default admin password from a Kubernetes Secret, and provision
      dashboards as code (ConfigMaps with the dashboard sidecar).
- [ ] Give the endpoint a stable DNS name (for example a Route 53 private hosted zone
      record pointing to the NLB) so Alloy doesn't depend on the generated hostname.
- [ ] Add more Loki replicas (and nodes) with `replication_factor: 3`, or move to a
      distributed deployment (`deploymentMode: Distributed`).
- [ ] Collect Kubernetes events with a separate single-replica Alloy deployment
      (`loki.source.kubernetes_events`), to avoid one copy per node.
- [ ] Enable multi-tenancy (`auth_enabled: true`) and have each source cluster send its
      own `X-Scope-OrgID`.

## Cleanup

```sh
helm uninstall grafana -n grafana --kube-context $EKS
kubectl --context $EKS -n grafana delete pvc --all
kubectl --context $EKS delete namespace grafana

helm uninstall alloy -n alloy --kube-context $ARGO
kubectl --context $ARGO delete namespace alloy

helm uninstall loki -n loki --kube-context $EKS   # also deletes the NLB
kubectl --context $EKS -n loki delete pvc --all
kubectl --context $EKS delete namespace loki
```

The S3 buckets and the IAM role are not removed by Helm; delete them separately if
they're no longer needed. The `gp3` StorageClass uses `reclaimPolicy: Retain`, so the
EBS volumes behind the Loki and Grafana PVCs are also kept after the PVCs are deleted.
Find them with `kubectl --context $EKS get pv` (status `Released`), then delete the PVs
and the EBS volumes.
