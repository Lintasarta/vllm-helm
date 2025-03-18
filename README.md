# Helm Chart for vLLM Deployment on DekaGPU

This Helm chart deploys AI models (e.g., Qwen2.5-72B-Instruct) using **vLLM** on a Kubernetes cluster with DekaGPU infrastructure. It provides full flexibility via `values.yaml` for configuring model path, resources, arguments, probes, PVC, and Service settings.

## Features

- Deploy AI models like Qwen, LLaMA, or other Hugging Face models using vLLM
- Configurable:
  - `replicaCount`, `affinity`, `tolerations`, `resources`
  - `image` and `model path`
  - `extraArgs` for `vllm serve`
  - Probes, PVC, and Service settings
- Optional PVC and Service creation (can reuse existing ones)
- Customizable Service annotations (e.g., MetalLB or DNS annotations)

---

## Prerequisites

- Helm 3.x
- Kubernetes cluster with GPU-enabled nodes
- Nvidia GPU Operator deployed (for `nvidia` runtime class)
- DekaGPU platform

---

## Folder Structure
```plaintext
vllm-helm/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── pvc.yaml
    └── service.yaml
```

## Installation Steps
### 1️⃣ Clone the repository
```bash
git clone https://github.com/<your-org>/vllm-helm.git
cd vllm-helm
```

### 2️⃣ Customize your values.yaml
```yaml
app: qwen-25-72b-instruct

modelName: Qwen/Qwen2.5-72B-Instruct

replicaCount: 1

image:
  repository: dekaregistry.cloudeka.id/cloudeka-system/vllm
  tag: v0.7.3
  pullPolicy: IfNotPresent

affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: nvidia.com/gpu.product
          operator: In
          values:
          - NVIDIA-L40S

tolerations: {}

pvc:
  enabled: true
  name: ""  
  storageClassName: storage-nvme-c1
  size: 250Gi

service:
  enabled: true
  name: ""
  type: LoadBalancer
  annotations:
    metallb.universe.tf/address-pool: address-pool
  ports:
    - name: http-vllm
      port: 80
      targetPort: 8000
      protocol: TCP
  sessionAffinity: None

resources:
  limits:
    cpu: "32"
    memory: "64Gi"
    nvidia.com/gpu: "8"
  requests:
    cpu: "4"
    memory: "16Gi"
    nvidia.com/gpu: "8"

# Additional arguments passed to vllm serve
# Example:
# extraArgs:
#   - "--gpu-memory-utilization"
#   - "0.95"
#   - "--tensor-parallel-size"
#   - "8"
#   - "--enforce-eager"
extraArgs: {}

# Optional additional env vars
# Example:
# env:
#   - name: HUGGING_FACE_HUB_TOKEN
#     value: <your-token>
env: {}

# Probe configurations for health checks
probes:
  liveness:
    path: /health
    port: 8000
    initialDelaySeconds: 600
    periodSeconds: 10
  readiness:
    path: /health
    port: 8000
    initialDelaySeconds: 600
    periodSeconds: 5
```
💡 Tip:
If you want to reuse existing PVC/Service, set pvc.enabled: false and service.enabled: false.

### 3️⃣ Install the Helm chart
```
helm install <release-name> . -f values.yaml -n <namespace> --create-namespace
```

Example:
```
helm install qwen-vllm . -f values.yaml -n vllm --create-namespace
```



Notes
- This chart mounts shared memory (/dev/shm) as required by vLLM tensor-parallel inference.
- Supports NVIDIA GPUs with node affinity.
- MetalLB annotations included for bare-metal load balancers.

TODO / Suggestions
- Support Horizontal Pod Autoscaler (HPA)
- ConfigMap/Secrets support for environment variables
- Optional ServiceMonitor for Prometheus scraping
