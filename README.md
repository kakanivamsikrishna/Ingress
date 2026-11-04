# Kubernetes NGINX Ingress: Path-Based Routing on AWS (kops)

A hands-on project that builds a Kubernetes cluster on AWS EC2 with **kops**, deploys several demo applications, and exposes them through a single NGINX Ingress Controller using path-based routing behind an AWS load balancer.

> **Note:** The upstream `kubernetes/ingress-nginx` project was retired in March 2026 and no longer receives security fixes. This repo is for learning and practice. For production, use a maintained controller (for example Traefik) or the Gateway API.

## Architecture

```
Client
  |
  v
AWS Elastic Load Balancer (created by the ingress controller Service)
  |
  v
NGINX Ingress Controller (ingress-nginx namespace)
  |
  |-- /flight --> myservice-1 --> mydeploy-1 (2 pods)
  |-- /pubg   --> myservice-2 --> mydeploy-2 (2 pods)
  |-- /yatra  --> myservice-3 --> mydeploy-3 (2 pods)
```

Cluster layout (kops, `us-east-2`):

- 1 control plane node (`m7i-flex.large`, 30 GB)
- 2 worker nodes (`c7i-flex.large`, 30 GB)
- Across zones `us-east-2a` and `us-east-2b`
- Gossip-based DNS cluster name (`*.k8s.local`), state stored in S3

## Repository Structure

| File | Description |
|------|-------------|
| `deploy1.yml` | Deployment (2 replicas) and ClusterIP Service for the **flight** app |
| `deploy2.yml` | Deployment (2 replicas) and ClusterIP Service for the **pubg** app |
| `deploy3.yml` | Deployment (2 replicas) and ClusterIP Service for the **yatra** app |
| `ingress.yml` | Ingress resource with path-based routing rules |

## Prerequisites

- An AWS account and an EC2 instance (or workstation) with the AWS CLI configured
- IAM permissions to create EC2, VPC, IAM, Auto Scaling, ELB, and S3 resources
- An S3 bucket for the kops state store

## Setup

### 1. Install kubectl and kops

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x kops
sudo mv kops /usr/local/bin/kops
```

### 2. Create the cluster

```bash
export KOPS_STATE_STORE=s3://<your-bucket-name>

kops create cluster \
  --name <cluster-name>.k8s.local \
  --zones us-east-2a,us-east-2b \
  --master-count 1 --master-size m7i-flex.large --master-volume-size 30 \
  --node-count 2 --node-size c7i-flex.large --node-volume-size 30 \
  --image=<ami-id-for-your-region>

kops update cluster --name <cluster-name>.k8s.local --yes --admin
kops validate cluster --wait 10m
```

The AMI ID is region-specific. Use one valid for your region and supported by your kops version.

### 3. Install the NGINX Ingress Controller

```bash
kubectl create -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.1/deploy/static/provider/cloud/deploy.yaml

kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
kubectl get ingressclass
```

Because kops provides the AWS cloud integration, the controller's `LoadBalancer` Service gets an ELB hostname under `EXTERNAL-IP`. The IngressClass created is named `nginx`.

### 4. Deploy the applications

```bash
kubectl apply -f deploy1.yml -f deploy2.yml -f deploy3.yml
kubectl get pods,svc
```

### 5. Create the Ingress

```bash
kubectl apply -f ingress.yml
kubectl get ing
kubectl describe ing vamsi-ingress
```

Key points in `ingress.yml`:

- `ingressClassName: nginx` must match the installed IngressClass.
- `pathType` is required in `networking.k8s.io/v1`.
- The rewrite annotation `/$2` needs a second capture group in each path, for example `/flight(/|$)(.*)`, so `/flight/abc` reaches the app as `/abc`.

## Testing

The Ingress only matches host `vamsi.com` and the configured paths. Requesting the load balancer hostname directly at `/` returns `404 Not Found` from NGINX, which is expected.

```bash
curl -H "Host: vamsi.com" http://<elb-hostname>/flight
curl -H "Host: vamsi.com" http://<elb-hostname>/pubg
curl -H "Host: vamsi.com" http://<elb-hostname>/yatra
```

To use a browser, either add `<elb-ip> vamsi.com` to your hosts file, or create a DNS record (for example in Route 53) pointing your domain to the ELB.

## Optional: Monitoring with Prometheus and Grafana

Installed with Helm using the kube-prometheus-stack chart:

```bash
helm install prometheus oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack

kubectl patch svc prometheus-grafana -p '{"spec":{"type":"LoadBalancer"}}'
kubectl patch svc prometheus-kube-prometheus-prometheus -p '{"spec":{"type":"LoadBalancer"}}'

# Grafana admin password
kubectl get secret prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

Exposing Grafana and Prometheus through public load balancers is fine for a lab, but not for anything real. Restrict access or use an authenticated ingress.

## Troubleshooting

| Symptom | Likely cause and fix |
|---------|----------------------|
| `pathType must be specified` | Add `pathType` to every path in the Ingress. |
| ADDRESS column is empty | `ingressClassName` does not match an existing IngressClass, or the controller Service has no external address yet. Check `kubectl get ingressclass` and `kubectl get svc -n ingress-nginx`. |
| `404 Not Found` (nginx) | Host or path did not match any rule. Send the `Host: vamsi.com` header and use a configured path. |
| `503 Service Temporarily Unavailable` | Backend pods are not running or not ready. Check `kubectl get pods` and `kubectl get endpoints`. |
| Pages load but CSS/images break | Check the rewrite annotation and capture groups in the paths. |

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
kubectl get endpoints myservice-1 myservice-2 myservice-3
```

## Cleanup

Delete the load-balancer-backed resources first so AWS removes the ELBs, then delete the cluster:

```bash
kubectl delete -f ingress.yml
kubectl delete -f deploy1.yml -f deploy2.yml -f deploy3.yml
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.1/deploy/static/provider/cloud/deploy.yaml
helm uninstall prometheus   # if installed

kops delete cluster --name <cluster-name>.k8s.local --yes
```

Running EC2 instances, EBS volumes, and load balancers all cost money, so delete the cluster when you finish practicing.

## What I Learned

- Creating a Kubernetes cluster on AWS with kops and an S3 state store
- How Ingress, Services, and Deployments connect
- Path-based routing with NGINX rewrite rules and regex paths
- Why IngressClass names must match between the controller and the Ingress
- How a `LoadBalancer` Service provisions an AWS ELB on a kops cluster

## License

MIT
