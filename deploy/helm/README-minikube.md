# AI Observability Metric Summarizer - Minikube Deployment

This guide explains how to deploy the AI Observability Metric Summarizer on Minikube instead of OpenShift.

## Prerequisites

1. **Minikube** with NGINX Ingress Controller enabled
2. **Prometheus** already running in your cluster
3. **VLLM or compatible LLM service** already running in your cluster
4. **Helm 3.x**

## Setup

### 1. Enable NGINX Ingress in Minikube

```bash
minikube addons enable ingress
```

### 2. Configure Values

Edit the Minikube-specific values files to point to your existing services:

#### For metric-mcp (`deploy/helm/metric-mcp/values-minikube.yaml`):

```yaml
config:
  # Update this to your Prometheus URL
  prometheusUrl: "http://prometheus.monitoring.svc.cluster.local:9090"

llm:
  # Update this to your VLLM/LLM service URL
  url: "http://vllm-service.default.svc.cluster.local:8000"
  apiToken: ""  # Add if your LLM requires authentication
```

#### For UI (`deploy/helm/ui/values-minikube.yaml`):

```yaml
env:
  # Update these to match your services
  PROMETHEUS_URL: "http://prometheus.monitoring.svc.cluster.local:9090"
  LLM_URL: "http://vllm-service.default.svc.cluster.local:8000"
  LLM_API_TOKEN: ""  # Add if your LLM requires authentication
```

### 3. Deploy

Deploy the metric-mcp service first:

```bash
helm install metric-mcp deploy/helm/metric-mcp -f deploy/helm/metric-mcp/values-minikube.yaml
```

Then deploy the UI:

```bash
helm install metric-ui deploy/helm/ui -f deploy/helm/ui/values-minikube.yaml
```

### 4. Access the Application

#### Option A: Local Minikube (Local Development)

Add the following entries to your `/etc/hosts` file:

```bash
echo "$(minikube ip) ai-metrics-ui.local" | sudo tee -a /etc/hosts
echo "$(minikube ip) ai-metrics-mcp.local" | sudo tee -a /etc/hosts
```

Then access the application at:
- **UI**: http://ai-metrics-ui.local
- **API**: http://ai-metrics-mcp.local

#### Option B: EC2 Instance (Cloud Deployment)

If you're running on an EC2 instance, you have two options:

**Option B1: Use EC2-specific deployment (Recommended)**

Use the EC2-specific Makefile targets that automatically configure ingress for public IP access:

```bash
# Deploy both services for EC2
make install-ec2 \
  NAMESPACE=ai-metrics \
  PROMETHEUS_URL=http://prometheus.monitoring.svc.cluster.local:9090 \
  LLM_URL=http://vllm-service.default.svc.cluster.local:8000

# Get your EC2 public IP
EC2_PUBLIC_IP=$(curl -s http://checkip.amazonaws.com/)
echo "Your public IP is: $EC2_PUBLIC_IP"
```

Then access via your EC2 public IP:
- **UI**: http://YOUR-EC2-PUBLIC-IP
- **API**: http://YOUR-EC2-PUBLIC-IP

**Option B2: Manual configuration**

If you already have the services deployed with Minikube values, remove host restrictions:

```bash
# Remove host restrictions for EC2 deployment
kubectl patch ingress metric-mcp-ingress -n ai-metrics --type='json' -p='[{"op": "remove", "path": "/spec/rules/0/host"}]'
kubectl patch ingress ui-ingress -n ai-metrics --type='json' -p='[{"op": "remove", "path": "/spec/rules/0/host"}]'
```

  **Option B3: Use custom domain/hosts**

Get your EC2 public IP and add to your local machine's `/etc/hosts`:

```bash
EC2_PUBLIC_IP=$(curl -s http://checkip.amazonaws.com/)
echo "$EC2_PUBLIC_IP ai-metrics-ui.local" | sudo tee -a /etc/hosts
echo "$EC2_PUBLIC_IP ai-metrics-mcp.local" | sudo tee -a /etc/hosts
```

Then access via the custom hostnames:
- **UI**: http://ai-metrics-ui.local
- **API**: http://ai-metrics-mcp.local

**Important EC2 Configuration:**

1. **Security Group**: Ensure your EC2 security group allows inbound traffic on port 80:
   ```bash
   # Allow HTTP traffic (port 80) from your IP or 0.0.0.0/0
   aws ec2 authorize-security-group-ingress \
     --group-id sg-your-security-group-id \
     --protocol tcp \
     --port 80 \
     --cidr 0.0.0.0/0
   ```

2. **Get your EC2 public IP**:
   ```bash
   curl -s http://checkip.amazonaws.com/
   # OR
   curl -s http://169.254.169.254/latest/meta-data/public-ipv4
   ```

## Key Differences from OpenShift Deployment

### Removed OpenShift-Specific Features:

1. **Routes** → Replaced with **Ingress**
2. **Service Account Tokens** → Uses environment variables or no authentication
3. **Trusted CA Bundles** → Uses default certificate verification
4. **OpenShift Monitoring Integration** → Uses your existing Prometheus

### Configuration Changes:

- **Prometheus URL**: No longer uses `thanos-querier.openshift-monitoring.svc.cluster.local`
- **Authentication**: Simplified to work without OpenShift service accounts
- **Networking**: Uses standard Kubernetes Ingress instead of OpenShift Routes

## Troubleshooting

### 1. Ingress Not Working

Check if NGINX Ingress Controller is running:

```bash
kubectl get pods -n ingress-nginx
```

### 2. Service Discovery Issues

Verify your Prometheus and LLM services are accessible:

```bash
# Test Prometheus
kubectl run test-pod --image=curlimages/curl --rm -it --restart=Never -- curl http://prometheus.monitoring.svc.cluster.local:9090/api/v1/status/config

# Test LLM service
kubectl run test-pod --image=curlimages/curl --rm -it --restart=Never -- curl http://vllm-service.default.svc.cluster.local:8000/health
```

### 3. Authentication Issues

If your Prometheus requires authentication, you may need to:

1. Create a service account with appropriate permissions
2. Mount the service account token in the deployment
3. Update the `THANOS_TOKEN` environment variable

### 4. DNS Resolution

If services can't resolve each other, check:

```bash
kubectl get svc -A  # List all services
kubectl get endpoints -A  # Check service endpoints
```

### 5. EC2-Specific Issues

**Cannot access via public IP:**

1. Check if ingress has host restrictions:
   ```bash
   kubectl get ingress -n ai-metrics -o yaml
   ```

2. Verify NGINX ingress controller is running:
   ```bash
   kubectl get pods -n ingress-nginx
   ```

3. Check security group allows port 80:
   ```bash
   # List security groups for your instance
   aws ec2 describe-instances --instance-ids $(curl -s http://169.254.169.254/latest/meta-data/instance-id) --query 'Reservations[*].Instances[*].SecurityGroups[*].GroupId' --output text
   
   # Check rules for a security group
   aws ec2 describe-security-groups --group-ids sg-your-group-id
   ```

4. Test ingress directly:
   ```bash
   # Test without host header
   curl -v http://YOUR-EC2-PUBLIC-IP
   
   # Test with host header (if using hostnames)
   curl -H 'Host: ai-metrics-ui.local' http://YOUR-EC2-PUBLIC-IP
   ```

**Services returning 404:**

This usually means the ingress rules still have host restrictions. Run:
```bash
make configure-ec2-ingress NAMESPACE=ai-metrics
```

## Customization

You can customize the deployment by modifying the values files:

- **Image tags**: Update `image.tag` in values files
- **Resource limits**: Add `resources` section to values files
- **Ingress hosts**: Change `ingress.host` to your preferred domain
- **Service types**: Change `service.type` to `NodePort` or `LoadBalancer` if needed

## Monitoring

The application exposes health endpoints:

- **MCP Health**: http://ai-metrics-mcp.local/health
- **UI**: Access through the web interface

Monitor the pods:

```bash
kubectl get pods
kubectl logs -f deployment/metric-mcp-app
kubectl logs -f deployment/ui
``` 