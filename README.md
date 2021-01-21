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

```bash
git clone https://github.com/mlrun/demos.git
```

Update model to ResNet50. Replaces EffecientNetB7.

```python
# src-tfv1/horovod-training.py
# src-tfv2/horovod-training.py

from tensorflow.keras.applications import ResNet50
model = ResNet50(include_top=False, input_shape=IMAGE_SHAPE)
```

Add limits/requests to job for best practices.

```python
# horovod-project.ipynb

# Assuming being run with p3.2xlarge
if use_gpu:
    trainer.gpus(1)
    trainer.with_requests(cpu=1, mem="3G")
    trainer.with_limits(cpu=6, mem="15G")
else:
    trainer.with_requests(cpu=1, mem="3G")
    trainer.with_limits(cpu=2, mem="5G")
```

Create image for training - TF v1 (only if running TF v1)

```python
# Any notebook

image = f"docker-registry.{os.getenv('IGZ_NAMESPACE_DOMAIN')}:80/tf-v1-image"

# Build Docker Image (only needs to be run once)
build_image = new_function(name="build-image", kind="job")
build_image.build_config(
    image=image, base_image="mlrun/ml-models-gpu:0.5.4-py36", commands=["pip uninstall tensorflow horovod -y",
                                                                        "conda install tensorflow-gpu==1.14.0 -y",
                                                                        "pip install keras==2.3.1 horovod==0.16.4"]
)
build_image.deploy(with_mlrun=False)
```

```python
# horovod-project.ipynb

if tf_ver == 'v1':
    trainer.spec.image = f"docker-registry.{os.getenv('IGZ_NAMESPACE_DOMAIN')}:80/tf-v1-image"
else:
    trainer.spec.image = image(use_gpu)
```

Update image for serving - TF v1 (only if running TF v1)

```python
# horovod-project.ipynb

if tf_ver == 'v1':
    hvdproj.set_function('hub://tf1_serving', 'serving')
    hvdproj.func("serving").spec.base_spec['spec']['build']['commands'].append("pip install 'h5py<3.0.0'")
else:
    hvdproj.set_function('hub://tf2_serving', 'serving')
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