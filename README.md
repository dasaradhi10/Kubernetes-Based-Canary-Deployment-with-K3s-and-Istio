TITLE: Kubernetes-Based Canary Deployment with K3s and Istio

## Objective

Simulate modern canary deployments using Istio's traffic splitting capabilities on a K3s (or Minikube in this case) cluster to manage traffic between stable and new application versions.

## Prerequisites

*   **Ubuntu Linux:** The environment where commands are executed.
*   **Docker:** Required as the driver for Minikube.
*   **Minikube:** To run a local Kubernetes cluster. (The steps use Minikube, although K3s is mentioned in the title - the process is similar).
*   **kubectl:** Kubernetes command-line tool.
*   **curl:** Tool for making HTTP requests.
*   **(Optional) kubens:** A tool to switch between Kubernetes namespaces quickly.

## Setup

### 1. Prepare Workspace

Create a dedicated directory for the project.

```bash
mkdir canary
cd canary

2. Install Istio CLI (istioctl)
Download, extract, and add istioctl to your PATH.

# Download Istio
wget https://github.com/istio/istio/releases/download/1.21.1/istio-1.21.1-linux-amd64.tar.gz

# Extract
tar -xvf istio-1.21.1-linux-amd64.tar.gz

# Rename for simplicity
mv istio-1.21.1 istio

# Add istioctl to PATH (adjust path if necessary)
export PATH=$PATH:$(pwd)/istio/bin

# Verify installation
istioctl version
# Should show client version and indicate connection status to control plane (later)


3. Start Kubernetes Cluster (Minikube)
Start a Minikube cluster using the Docker driver.
# Start Minikube (may require sudo depending on Docker setup)
sudo minikube start --driver=docker --force

# Check cluster status
minikube status
# Example Output:
# minikube
# type: Control Plane
# host: Running
# kubelet: Running
# apiserver: Running
# kubeconfig: Configured

# Verify node(s)
kubectl get nodes
# Example Output (Node name might differ):
# NAME        STATUS   ROLES                  AGE   VERSION
# minikube    Ready    control-plane,master   ...   v1.xx.x

4. Install Istio on Kubernetes
Install Istio components onto the cluster using the demo profile.

# List available Istio profiles (optional)
istioctl profile list

# Install Istio using the demo profile
istioctl install --set profile=demo -y
# Example Output:
# ✔ Istio core installed
# ✔ Istiod installed
# ✔ Egress gateways installed
# ✔ Ingress gateways installed
# ✔ Installation complete

# Switch to the istio-system namespace (optional, for verification)
# If you don't have kubens, use: kubectl config set-context --current --namespace=istio-system
kubens istio-system

# Verify Istio components are running
kubectl get services
kubectl get pods
# Look for istiod, istio-ingressgateway, istio-egressgateway

# Switch back to the default namespace
# If you don't have kubens, use: kubectl config set-context --current --namespace=default
kubens default

5. Understanding Istio Sidecar Injection (Demonstration)
Istio works by injecting a sidecar proxy (Envoy) into your application pods. This can be done manually or automatically.
Manual Injection (Demonstration):
# Create a sample deployment (alpine.yml)
cat <<EOF > alpine.yml
apiVersion: v1
kind: Service
metadata:
  name: alpine
  labels:
    app: alpine
spec:
  ports:
    - port: 80
      name: http
  selector:
    app: alpine
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alpine
spec:
  selector:
    matchLabels:
      app: alpine
  template:
    metadata:
      labels:
        app: alpine
    spec:
      containers:
        - name: alpine
          image: alpine
          command: ["sleep"]
          args: ["100000"]
EOF

# See what manual injection would look like (doesn't apply)
# istioctl kube-inject --filename alpine.yml

# Apply the manifest WITHOUT automatic injection
kubectl apply -f alpine.yml

# Check the pod - notice READY 1/1 (no sidecar)
kubectl get pods
# NAME                      READY   STATUS    RESTARTS   AGE
# alpine-xxxxxxxxxx-xxxxx   1/1     Running   0          ...

# Clean up this demo deployment
kubectl delete -f alpine.yml

Automatic Injection:
Enable automatic sidecar injection for the default namespace (or any target namespace).

# Label the namespace for automatic injection
kubectl label namespace default istio-injection=enabled
# namespace/default labeled

# Verify the label
kubectl describe namespace default
# Look for Labels: istio-injection=enabled

# Re-create the deployment
kubectl apply -f alpine.yml

# Check the pod - notice READY 2/2 (app container + Istio sidecar)
kubectl get pods
# NAME                      READY   STATUS    RESTARTS   AGE
# alpine-yyyyyyyyyy-yyyyy   2/2     Running   0          ...

# Disable automatic injection for the default namespace
kubectl label namespace default istio-injection-
# namespace/default unlabeled

# Restart the deployment - new pods won't have the sidecar
kubectl rollout restart deployment alpine

# Check the pod - notice READY 1/1 again
kubectl get pods
# NAME                      READY   STATUS    RESTARTS   AGE
# alpine-zzzzzzzzzz-zzzzz   1/1     Running   0          ...

# Clean up the Alpine deployment
kubectl delete -f alpine.yml

Canary Deployment Simulation
1. Prepare Application Files
Organize your Kubernetes manifests. Based on the provided structure:


.
├── canary/
│   └── istio/              # Istio installation files (downloaded)
└── go-demo-7/
    └── k3s/
        └── istio/
            ├── gateway/
            │   ├── app/
            │   │   ├── deployment.yaml     # App v0.0.1 Deployment
            │   │   ├── service.yaml        # App Service
            │   │   └── istio.yaml          # Istio Gateway & VirtualService
            │   └── db/
            │       ├── deployment-standalone.yaml
            │       ├── pvc-standalone.yaml
            │       └── svc-standalone.yaml
            ├── split/
            │   └── excercise/
            │       ├── app-0-0-2-canary.yaml # App v0.0.2 Canary Deployment
            │       ├── app-0-0-2.yaml      # App v0.0.2 Deployment (for primary upgrade)
            │       ├── host20.yaml         # Services for primary/canary, initial VS
            │       ├── split40.yaml        # DR + VS for 40% canary traffic
            │       ├── split60.yaml        # VS for 60% canary traffic
            │       └── split100.yaml       # VS for 100% primary traffic + canary scale down
            └── metrics/ # (Not used in this flow, but listed)
                ├── metrics.yaml
                └── prom-ingress.yaml

# Note: You need to create these files with the appropriate content.
# Assuming `go-demo-7/k3s/istio` is the current directory for the next steps.

Make sure you have the YAML files mentioned above (like gateway/app/deployment.yaml, split/excercise/app-0-0-2-canary.yaml, etc.) populated with the correct Kubernetes/Istio resource definitions. The image used seems to be vfarcic/go-demo-7.

2. Deploy Initial Application Version (v0.0.1)
Create a dedicated namespace, enable Istio injection, and deploy the first version of the application along with its database and Istio gateway resources.

# Assume we are in the `go-demo-7/k3s/istio` directory
# If not: cd go-demo-7/k3s/istio

# Create namespace
kubectl create ns go-demo-7

# Enable automatic Istio sidecar injection
kubectl label namespace go-demo-7 istio-injection=enabled

# Switch to the new namespace
# If you don't have kubens: kubectl config set-context --current --namespace=go-demo-7
kubens go-demo-7

# Apply all resources in the gateway directory (App v0.0.1, DB, Service, Istio GW/VS)
kubectl apply -f gateway/ --recursive
# service/go-demo-7 created
# deployment.apps/go-demo-7 created
# gateway.networking.istio.io/go-demo-7 created
# virtualservice.networking.istio.io/go-demo-7 created
# deployment.apps/go-demo-7-db created
# persistentvolumeclaim/go-demo-7-db-pvc created
# service/go-demo-7-db created

# Verify pods (should be 2/2 ready)
kubectl get pods
# NAME                            READY   STATUS    RESTARTS   AGE
# go-demo-7-db-xxxxxxxxx-xxxxx    2/2     Running   0          ...
# go-demo-7-xxxxxxxxxx-xxxxx      2/2     Running   0          ...

# Verify Istio resources
kubectl get virtualservices
kubectl get gateways # Often abbreviated as gw

3. Access Application via Istio Ingress Gateway
Find the Ingress Gateway's external access point and test the application.

# Get the NodePort for the Istio Ingress Gateway's HTTP2 port
export INGRESS_PORT=$(kubectl \
    --namespace istio-system \
    get service istio-ingressgateway \
    --output jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')

# Get the Minikube IP
export MINIKUBE_IP=$(minikube ip)

# Construct the Ingress Host URL
export INGRESS_HOST=$MINIKUBE_IP:$INGRESS_PORT

# Test accessing the /version endpoint (should show 0.0.1)
# Use -H to set the Host header, which the Gateway uses for routing
for i in {1..5}; do
    curl -s -H "Host: go-demo-7.acme.com" "http://$INGRESS_HOST/version"
    echo "" # Add a newline for readability
done
# Example Output (repeated):
# Version: 0.0.1; Release: unknown


4. Deploy Canary Version (v0.0.2)
Deploy the new version (v0.0.2) of the application as a separate "canary" deployment. Initially, it won't receive traffic via the main VirtualService.

# Apply the canary deployment manifest
kubectl apply -f split/excercise/app-0-0-2-canary.yaml
# deployment.apps/go-demo-7-canary created

# Check the canary deployment and pod
kubectl get deployment.apps/go-demo-7-canary
kubectl get pods
# NAME                                READY   STATUS    RESTARTS   AGE
# go-demo-7-canary-yyyyyyyyy-yyyyy    2/2     Running   0          ...
# go-demo-7-db-xxxxxxxxx-xxxxx        2/2     Running   0          ...
# go-demo-7-xxxxxxxxxx-xxxxx          2/2     Running   0          ...

# Test access again - should still only show v0.0.1
# Traffic hasn't been routed to the canary yet.
for i in {1..5}; do
    curl -s -H "Host: go-demo-7.acme.com" "http://$INGRESS_HOST/version"
    echo ""
done
# Example Output (repeated):
# Version: 0.0.1; Release: unknown


5. Initiate Traffic Splitting (Canary Release)
Create separate Kubernetes Services for the primary and canary deployments and update the Istio VirtualService to split traffic.

# Apply resources to create primary/canary services and initial VS split
# host20.yaml likely defines:
# - Service/go-demo-7-primary (selector: app=go-demo-7, version=0.0.1 or similar stable label)
# - Service/go-demo-7-canary (selector: app=go-demo-7, version=0.0.2 or similar canary label)
# - VirtualService/go-demo-7 routing 100% to primary initially, or maybe a small % to canary
# Check the content of host20.yaml to be sure. Let's assume it starts with 100% to primary.
kubectl apply -f split/excercise/host20.yaml
# service/go-demo-7-primary created
# service/go-demo-7-canary created
# virtualservice.networking.istio.io/go-demo-7 configured

# Verify services
kubectl get svc
# NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
# go-demo-7           ClusterIP   ...             <none>        80/TCP      ...
# go-demo-7-canary    ClusterIP   ...             <none>        80/TCP      ...
# go-demo-7-db        ClusterIP   ...             <none>        27017/TCP   ...
# go-demo-7-primary   ClusterIP   ...             <none>        80/TCP      ...

# Test access - output depends on the exact VS definition in host20.yaml
# If it was 100% to primary, still only v0.0.1. If it started a split (e.g., 80/20), you'll see both.
# The original log shows a mix AFTER deploying canary but BEFORE host20.yaml,
# suggesting the original service selector might have matched both.
# Let's proceed assuming host20.yaml sets up the structure for splitting.

# Apply 40% split to Canary (requires a DestinationRule and updated VirtualService)
# split40.yaml should define:
# - DestinationRule/go-demo-7 defining subsets 'primary' and 'canary'
# - VirtualService/go-demo-7 routing 60% to subset 'primary', 40% to subset 'canary'
kubectl apply -f split/excercise/split40.yaml
# destinationrule.networking.istio.io/go-demo-7 created (or configured)
# virtualservice.networking.istio.io/go-demo-7 configured

# Test access (approx. 40% should be v0.0.2)
echo "--- Testing 60/40 Split ---"
for i in {1..10}; do curl -s -H "Host: go-demo-7.acme.com" "http://$INGRESS_HOST/version"; echo ""; done
# Example Output (mix of v0.0.1/primary and v0.0.2/canary):
# Version: 0.0.1; Release: primary
# Version: 0.0.2; Release: canary
# ... (ratio should approach 60/40 over many requests)

# Apply 60% split to Canary
# split60.yaml should update VirtualService weights (40% primary, 60% canary)
kubectl apply -f split/excercise/split60.yaml
# virtualservice.networking.istio.io/go-demo-7 configured

# Test access (approx. 60% should be v0.0.2)
echo "--- Testing 40/60 Split ---"
for i in {1..10}; do curl -s -H "Host: go-demo-7.acme.com" "http://$INGRESS_HOST/version"; echo ""; done
# Example Output (mix, now favouring v0.0.2/canary):
# Version: 0.0.2; Release: canary
# Version: 0.0.1; Release: primary
# ... (ratio should approach 40/60 over many requests)


6. Finalize Deployment (Promote Canary)
If the canary version (v0.0.2) is stable, upgrade the primary deployment and shift all traffic to it.

# Upgrade the primary deployment to version 0.0.2
# app-0-0-2.yaml updates the image and any relevant env vars/labels for the PRIMARY deployment
kubectl apply -f split/excercise/app-0-0-2.yaml
# deployment.apps/go-demo-7-primary configured

# Check rollout status
kubectl rollout status deployment go-demo-7-primary
# deployment "go-demo-7-primary" successfully rolled out

# Check pods - new primary pods running v0.0.2
kubectl get pods
# NAME                                    READY   STATUS    RESTARTS   AGE
# go-demo-7-canary-yyyyyyyyy-yyyyy        2/2     Running   0          ...
# go-demo-7-db-xxxxxxxxx-xxxxx            2/2     Running   0          ...
# go-demo-7-primary-zzzzzzzzz-zzzzz       2/2     Running   0          ... (new pods)
# go-demo-7-primary-zzzzzzzzz-aaaaa       2/2     Running   0          ... (new pods)
# go-demo-7-primary-zzzzzzzzz-bbbbb       2/2     Running   0          ... (new pods)

# Test access (should show a mix of v0.0.2/primary and v0.0.2/canary, based on 40/60 split)
echo "--- Testing Post-Primary Upgrade (Still 40/60 Split) ---"
for i in {1..10}; do curl -s -H "Host: go-demo-7.acme.com" "http://$INGRESS_HOST/version"; echo ""; done
# Example Output (All should be v0.0.2, but from different deployments):
# Version: 0.0.2; Release: primary
# Version: 0.0.2; Release: canary
# ...

# Shift 100% traffic to the upgraded primary deployment and scale down canary
# split100.yaml should:
# - Update VirtualService/go-demo-7 to route 100% to subset 'primary'
# - Optionally scale Deployment/go-demo-7-canary replicas to 0
kubectl apply -f split/excercise/split100.yaml
# deployment.apps/go-demo-7-canary configured (if scaled down)
# virtualservice.networking.istio.io/go-demo-7 configured

# Test access (should show only v0.0.2/primary)
echo "--- Testing 100% Primary Traffic ---"
for i in {1..10}; do curl -s -H "Host: go-demo-7.acme.com" "http://$INGRESS_HOST/version"; echo ""; done
# Example Output (repeated):
# Version: 0.0.2; Release: primary

# Verify final state - canary deployment might be scaled to 0
kubectl get pods
# NAME                                    READY   STATUS    RESTARTS   AGE
# go-demo-7-db-xxxxxxxxx-xxxxx            2/2     Running   0          ...
# go-demo-7-primary-zzzzzzzzz-zzzzz       2/2     Running   0          ...
# go-demo-7-primary-zzzzzzzzz-aaaaa       2/2     Running   0          ...
# go-demo-7-primary-zzzzzzzzz-bbbbb       2/2     Running   0          ...
# (Canary pod should be Terminating or gone if scaled to 0)

kubectl get deployments
# NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
# go-demo-7-canary     0/0     0            0           ... (if scaled down)
# go-demo-7-db         1/1     1            1           ...
# go-demo-7-primary    3/3     3            3           ...

The canary deployment is now complete. Version 0.0.2 is fully rolled out.
Cleanup
To remove the created resources:
# Delete application resources from the go-demo-7 namespace
kubectl delete ns go-demo-7

# Uninstall Istio from the cluster
istioctl uninstall --purge -y

# Delete the istio-system namespace (if it persists)
kubectl delete ns istio-system

# Stop the Minikube cluster
sudo minikube stop

# Delete the Minikube cluster (optional, frees up resources)
# sudo minikube delete

# Remove downloaded Istio files (optional)
# cd /path/to/canary # Navigate back to the initial directory
# rm -rf istio istio-1.21.1-linux-amd64.tar.gz

# Unset environment variables (optional)
# unset INGRESS_PORT INGRESS_HOST MINIKUBE_IP
# Remove the istio path export from your shell profile if added permanently
