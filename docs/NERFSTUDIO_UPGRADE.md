# Upgrading to Nerfstudio 1.1.5

The original extension was built for nerfstudio 0.3.4. This document covers the changes required to support models trained with nerfstudio 1.1.5.

## Summary of Changes

Two main issues needed to be fixed:

1. **Missing `three_js_perspective_camera_focal_length` function** - This utility was removed from nerfstudio's public API
2. **State dict size mismatch for training-specific parameters** - `camera_optimizer` and `embedding_appearance` tensors have sizes dependent on the number of training images

## Code Changes

### 1. `nerfstudio_renderer/src/nerfstudio_renderer/utils.py`

Added local implementation of the focal length calculation function:

```python
import math

def three_js_perspective_camera_focal_length(fov: float, image_height: int) -> float:
    """Compute focal length from vertical FOV and image height.

    This function was previously imported from nerfstudio.viewer.server.utils
    but was removed in newer versions of nerfstudio.

    Args:
        fov: Vertical field of view in degrees
        image_height: Height of the image in pixels

    Returns:
        Focal length in pixels
    """
    return image_height / (2.0 * math.tan(math.radians(fov) / 2.0))
```

Updated the import in `get_camera()` function to use the local implementation instead of:
```python
# Old (broken in nerfstudio 1.1.5):
from nerfstudio.viewer.server.utils import three_js_perspective_camera_focal_length
```

### 2. `nerfstudio_renderer/src/nerfstudio_renderer/renderer.py`

Added filtering of training-specific parameters when loading the model state dict. These parameters have tensor sizes that depend on the number of training images, which causes size mismatches at inference time.

In the `__init__` method, after loading the checkpoint and before `load_state_dict`:

```python
# Drop training-specific parameters that have size mismatches at inference time:
# - embedding_appearance: requires number of training images
# - camera_optimizer: per-image pose adjustments from training
# Ref: https://github.com/nerfstudio-project/nerfstudio/blob/c87ebe34ba8b11172971ce48e44b6a8e8eb7a6fc/nerfstudio/fields/nerfacto_field.py#L112
model_state = { key: value for key, value in model_state.items()
               if 'embedding_appearance' not in key and 'camera_optimizer' not in key }
```

Also using `strict=False` when loading:
```python
self.model.load_state_dict(model_state, strict=False)
```

## Docker Image Update

The Dockerfile should be updated to use nerfstudio 1.1.5:

### Option 1: Use Pre-built Image (if available)

```dockerfile
FROM j3soon/nerfstudio-renderer:v1.1.5
```

### Option 2: Build from Newer Base

Update `nerfstudio_renderer/Dockerfile`:

```dockerfile
# Change from:
FROM dromni/nerfstudio:0.3.4

# To a newer nerfstudio image or build nerfstudio 1.1.5:
FROM nvidia/cuda:11.8.0-devel-ubuntu22.04

# Install nerfstudio 1.1.5
RUN pip install nerfstudio==1.1.5

# ... rest of setup
```

### Option 3: Upgrade In-Container

If using the existing image, upgrade nerfstudio inside the container:

```dockerfile
FROM dromni/nerfstudio:0.3.4

ARG SERVER_PORT
ENV SERVER_PORT=$SERVER_PORT

# Upgrade nerfstudio to 1.1.5
RUN pip install --upgrade nerfstudio==1.1.5

RUN sudo pip install rpyc

WORKDIR /src

ENTRYPOINT cp -r /src ~/src \
           && cd ~/src \
           && sudo pip install . \
           && python3 --version \
           && rpyc_classic --host 0.0.0.0 --port $SERVER_PORT
```

## Remote Renderer Setup

For running the renderer on a remote machine (e.g., via Tailscale):

1. Update `compose.yaml` on the remote machine to mount your NeRF outputs:
   ```yaml
   volumes:
     - /path/to/your/nerf/outputs:/workspace:ro
   ```

2. Update `extension.py` to point to the remote host:
   ```python
   def init_rpyc(self):
       host = '100.122.215.70'  # Remote machine IP (e.g., Tailscale)
       port = 10001
       model_config_path = '/workspace/outputs/your-model/nerfacto/DATE_TIME/config.yml'
       model_checkpoint_path = '/workspace/outputs/your-model/nerfacto/DATE_TIME/nerfstudio_models/step-XXXXXX.ckpt'
   ```

## Troubleshooting

### Error: `No module named 'nerfstudio.viewer.server.utils'`

The `three_js_perspective_camera_focal_length` function was moved/removed. Use the local implementation in `utils.py`.

### Error: `size mismatch for camera_optimizer.pose_adjustment`

The checkpoint contains per-image camera pose adjustments from training. Filter these out when loading (see renderer.py changes above).

### Error: `size mismatch for field.embedding_appearance`

Similar to above - filter out `embedding_appearance` keys from the state dict.

### Error: `No module named 'nerfstudio.data.datamanagers.parallel_datamanager'`

Your nerfstudio version is too old. Models trained with nerfstudio 1.1.5 require the renderer to also use 1.1.5+.

## Why These Parameters Can Be Filtered

- **`camera_optimizer`**: Stores per-image pose refinements learned during training. At inference time, we provide explicit camera poses, so these aren't needed.

- **`embedding_appearance`**: Stores per-image appearance embeddings to handle lighting variations across training images. At inference, we render with a default/neutral appearance.

Both are training artifacts that the neural field doesn't need for rendering novel views.
