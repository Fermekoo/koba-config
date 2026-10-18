# koba-config

Kubernetes manifests for the `koba-app` microservice, managed with Kustomize.

## Manifests

| File              | Purpose                                          |
|-------------------|--------------------------------------------------|
| `deployment.yaml` | 2-replica Deployment with probes + resource limits |
| `service.yaml`    | LoadBalancer Service with AWS NLB annotations   |
| `hpa.yaml`        | HPA scaling 2–10 replicas (75% CPU / 80% RAM)   |
| `kustomization.yaml` | Kustomize root — groups all resources         |

## Local Validation

```bash
# Render all manifests
kubectl kustomize .

# Client-side dry run (no cluster needed)
kubectl kustomize . | kubectl apply --dry-run=client -f - --validate=false
```

## Deploying to EKS

```bash
# Configure kubectl context
aws eks --region us-east-1 update-kubeconfig --name koba-eks-cluster

# Apply all manifests
kubectl apply -k .

# Verify deployment
kubectl get deployments,svc,hpa -n default
```

## CI/CD Pipeline

Triggered on every push/PR to `main`:

- Renders manifests with `kubectl kustomize .`
- Runs a client-side dry-run apply to catch schema errors

## Updating the Image Tag

After `koba-app` CI pushes a new image to ECR, manually update `deployment.yaml`:

```yaml
# Before
image: <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/koba-app:latest

# After (use the git SHA from koba-app CI output)
image: 123456789.dkr.ecr.us-east-1.amazonaws.com/koba-app:abc1234
```

Then commit and push — the validate workflow will confirm the manifests are still valid.

## Post-Setup Notes

### Install Metrics Server (required for HPA)

After the EKS cluster is provisioned via `koba-service`, install metrics-server:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl get deployment metrics-server -n kube-system
```

### AWS Load Balancer Controller (required for NLB Service)

The NLB annotations in `service.yaml` require the AWS Load Balancer Controller to be installed on the cluster. See the [AWS docs](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html) for installation steps.
