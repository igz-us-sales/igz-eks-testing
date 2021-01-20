# Iguazio EKS Testing

## Additional Setup Steps
### Nvidia Daemonset Override

Updates nvidia daemon for correct GPU discovery.

```bash
kubectl delete -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.7.3/nvidia-device-plugin.yml
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.7.0/nvidia-device-plugin.yml
```

### MLRun Cleanup Override

Updates MLRun to immediately release resources once a job is complete. Allows for GPU nodes to scale down after job.

```bash
vi cleanup_override.yaml

apiVersion: v1
data:
  MLRUN_RUNTIME_RESOURCES_DELETION_GRACE_PERIOD: "0"
  MLRUN_RUNTIMES_CLEANUP_INTERVAL: "60"
kind: ConfigMap
metadata:
  name: mlrun-override-env
  namespace: default-tenant
  
kubectl apply -f cleanup_override.yaml

kubectl -n default-tenant delete pod <mlrun-api-pod>
```

### Horovod Demo Additional Steps

Git clone updated demos repo.

```
git clone https://github.com/mlrun/demos.git
```

Update model to ResNet50. Replaces EffecientNetB7.

```
horovod-training.py

from tensorflow.keras.applications import ResNet50
model = ResNet50(include_top=False, input_shape=IMAGE_SHAPE)
```

Add limits/requests to job for best practices.

```
horovod-project.ipynb

trainer.with_requests(cpu=1, mem="3G")
trainer.with_limits(cpu=2, mem="5G")
```

## Testing Results
- Scale up
    - ~1 min to kick off scaling
    - ~4 mins to add node to cluster
    - ~Additional 4.5 mins to pull image and ititialize launcher
    - Total: ~10m
- Scale down
    - ~10 mins to start shutting down
    - ~5 mins to completely remove from K8s
    - ~2 additional mins to completely remove from UI
    - Total: ~15/17m