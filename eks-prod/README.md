# EKS GameWarden Deployment Guide

## Prerequisites

1. **EKS Cluster Setup (without CNI)**
   ```bash
   eksctl create cluster \
     --name gamewarden-cluster \
     --region us-east-1 \
     --nodegroup-name standard-workers \
     --node-type m5.xlarge \
     --nodes 3 \
     --nodes-min 3 \
     --nodes-max 5 \
     --without-nodegroup \
     --max-pods-per-node 100
   ```

2. **Install Cilium CNI**
   ```bash
   # Install Cilium CLI
   curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
   tar xzvf cilium-linux-amd64.tar.gz
   sudo mv cilium /usr/local/bin

   # Install Cilium on EKS
   cilium install --version 1.14.5 \
     --set egressGateway.enabled=true \
     --set hubble.relay.enabled=true \
     --set hubble.ui.enabled=true

   # Verify Cilium installation
   cilium status --wait
   ```

3. **Install AWS EBS CSI Driver**
   ```bash
   eksctl create iamserviceaccount \
     --name ebs-csi-controller-sa \
     --namespace kube-system \
     --cluster gamewarden-cluster \
     --role-name AmazonEKS_EBS_CSI_DriverRole \
     --role-only \
     --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
     --approve

   eksctl create addon \
     --name aws-ebs-csi-driver \
     --cluster gamewarden-cluster \
     --service-account-role-arn arn:aws:iam::$ACCOUNT_ID:role/AmazonEKS_EBS_CSI_DriverRole \
     --force
   ```

4. **Create S3 Bucket for Loki**
   ```bash
   aws s3api create-bucket \
     --bucket gamewarden-loki-storage \
     --region us-east-1
   ```

5. **Install Flux**
   ```bash
   curl -s https://fluxcd.io/install.sh | sudo bash
   flux check --pre
   ```

## Deployment Steps

1. **Install SOPS Age Key**

   If you have an existing Age key file (e.g., `keys.txt`):
   ```bash
   # Create the bigbang namespace
   kubectl create namespace bigbang

   # Create the secret from your existing key file
   kubectl create secret generic sops-age \
     --namespace=bigbang \
     --from-file=age.agekey=keys.txt
   ```

   If you need to generate a new Age key:
   ```bash
   # Generate new key
   age-keygen -o age.agekey

   # Create Kubernetes secret
   kubectl create namespace bigbang
   cat age.agekey | kubectl create secret generic sops-age \
     --namespace=bigbang \
     --from-file=age.agekey=/dev/stdin
   ```

   **Important**:
    - Store the Age private key file securely (never commit to Git)
    - The public key in `.sops.yaml` must match your private key
    - You'll need this key to encrypt/decrypt secrets in the future

2. **Update Configuration**
    - Edit `eks-prod/configmap.yaml`:
        - Update S3 region and bucket name for Loki
        - Update domain names and URLs
        - Update SSO client secrets
    - Update `bootstrap.yaml` to point to `./eks-prod` instead of `./localdev`

3. **Bootstrap GitOps**
   ```bash
   # Fork this repository and update the URL in bootstrap.yaml

   # Apply bootstrap configuration
   kubectl apply -f bootstrap.yaml

   # Watch Flux sync status
   flux get all -A --watch
   ```

4. **Configure DNS**
    - Get the Load Balancer URL:
      ```bash
      kubectl get svc -n istio-system istio-ingressgateway
      ```
    - Create Route53 records or update your DNS to point to the NLB

5. **Verify Installation**
   ```bash
   # Check all pods are running
   kubectl get pods -A

   # Check Flux resources
   flux get all -A

   # Access Grafana
   echo "https://grafana.gamewarden.example.com"

   # Access Keycloak
   echo "https://login.gamewarden.example.com"
   ```

## Key Differences from Local Dev

1. **Storage**: Uses EBS gp3 storage class instead of local-path
2. **Networking**: Uses AWS NLB for ingress instead of local load balancer
3. **Loki Backend**: Uses S3 instead of MinIO for log storage
4. **Resources**: Higher resource limits suitable for production
5. **High Availability**: Multiple replicas for critical components
6. **Network Policies**: Enabled with Cilium CNI

## Customization

- **TLS Certificates**: Add cert-manager or use AWS Certificate Manager
- **IAM Roles**: Configure IRSA for pods needing AWS access
- **Monitoring**: Consider adding CloudWatch integration
- **Backup**: Add Velero for cluster backup to S3

## Troubleshooting

```bash
# Check Flux logs
flux logs --all-namespaces --follow

# Check events
kubectl get events -A --sort-by='.lastTimestamp'

# Reconcile Flux resources
flux reconcile source git environment-repo -n bigbang
flux reconcile kustomization environment -n bigbang
```