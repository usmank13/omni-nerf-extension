# Using the Remote NeRF Renderer

This guide covers the setup for running the NeRF renderer on a remote machine (Beast) and viewing in Isaac Sim on your local machine.

## Architecture

```
┌─────────────────────┐         Tailscale          ┌─────────────────────┐
│   Local Machine     │◄──────────────────────────►│       Beast         │
│                     │      100.122.215.70        │                     │
│  - Isaac Sim        │                            │  - Docker container │
│  - NeRF Extension   │◄───── RPyC (10001) ───────►│  - nerfstudio 1.1.5 │
│  - RTX 3070 Ti      │                            │  - RTX 3090         │
└─────────────────────┘                            │  - /data/NeRF/      │
                                                   └─────────────────────┘
```

## Quick Start

### 1. Start the Renderer on Beast

SSH to Beast and start the container:
```sh
ssh usman@AIGEN-BEAST
cd /home/usman/Desktop/omni-nerf-extension
docker compose up -d nerfstudio-renderer
```

Verify it's running:
```sh
docker logs nerfstudio-renderer | tail -5
# Should show: INFO SLAVE/10001[MainThread]: server started on [0.0.0.0]:10001
```

### 2. Update Model Path (to try a different NeRF)

Edit the extension on your local machine:
```sh
nano /home/usman/Desktop/code/omni-nerf-extension/extension/exts/omni.nerf.viewport/omni/nerf/viewport/extension.py
```

Find the `init_rpyc` method and update the paths:
```python
def init_rpyc(self):
    host = '100.122.215.70'  # Beast via Tailscale
    port = 10001
    # UPDATE THESE PATHS:
    model_config_path = '/workspace/outputs/YOUR-MODEL/nerfacto/DATE_TIME/config.yml'
    model_checkpoint_path = '/workspace/outputs/YOUR-MODEL/nerfacto/DATE_TIME/nerfstudio_models/step-XXXXXX.ckpt'
    device = 'cuda'
```

### 3. Launch Isaac Sim

```sh
/isaac-sim/isaac-sim.sh \
  --ext-folder /home/usman/Desktop/code/omni-nerf-extension/extension/exts \
  --enable omni.nerf.viewport
```

### 4. Set Up the Scene

1. Create a reference mesh: `Create > Mesh > Plane` (for outdoor scenes)
2. Select the plane in Stage
3. Click **Set** in the NeRF Viewport window
4. Navigate the camera to see your NeRF

## Available NeRFs on Beast

NeRF outputs are stored at `/data/NeRF/outputs/` on Beast, mounted as `/workspace/outputs/` in the container.

To list available models:
```sh
ssh usman@AIGEN-BEAST "ls /data/NeRF/outputs/"
```

To find config and checkpoint paths for a model:
```sh
ssh usman@AIGEN-BEAST "find /data/NeRF/outputs/MODEL_NAME -name 'config.yml' -o -name '*.ckpt'"
```

### Example Model Paths

| Model | Config Path | Checkpoint Path |
|-------|-------------|-----------------|
| hiero-110-crop-cams | `/workspace/outputs/hiero-110-crop-cams/nerfacto/2026-01-19_171518/config.yml` | `.../nerfstudio_models/step-000029999.ckpt` |

## Switching NeRFs Without Restarting Isaac Sim

Currently, you need to restart Isaac Sim to load a different NeRF (the model is loaded on extension startup).

**Workaround**: Disable and re-enable the extension:
1. `Window > Extensions`
2. Search for "nerf"
3. Toggle off `omni.nerf.viewport`
4. Edit the model paths in `extension.py`
5. Toggle on `omni.nerf.viewport`

## Troubleshooting

### Connection Refused

Check the container is running:
```sh
ssh usman@AIGEN-BEAST "docker ps | grep nerfstudio"
```

Check port is open:
```sh
nc -zv 100.122.215.70 10001
```

### Model Loading Errors

Check container logs for errors:
```sh
ssh usman@AIGEN-BEAST "docker logs nerfstudio-renderer --tail 50"
```

Common issues:
- Wrong path (check the path exists in `/workspace/outputs/`)
- Model trained with different nerfstudio version (should work with 1.1.5 models)

### Slow Rendering

- First frame takes longer (model warmup)
- Check Beast GPU isn't being used by other processes:
  ```sh
  ssh usman@AIGEN-BEAST "nvidia-smi"
  ```

### Container Lost Fixes After Restart

If you recreate the container (`docker compose down/up`), you need to reinstall the fixed renderer:
```sh
ssh usman@AIGEN-BEAST "docker exec nerfstudio-renderer bash -c 'cp -r /src /tmp/nerfstudio_renderer && pip install --force-reinstall /tmp/nerfstudio_renderer'"
```

Or rebuild the image with the fixes baked in (see NERFSTUDIO_UPGRADE.md).

## Restarting the Renderer

If you need to restart just the renderer:
```sh
ssh usman@AIGEN-BEAST "docker restart nerfstudio-renderer"
```

To fully recreate (loses pip-installed fixes):
```sh
ssh usman@AIGEN-BEAST "cd /home/usman/Desktop/omni-nerf-extension && docker compose down nerfstudio-renderer && docker compose up -d nerfstudio-renderer"
```
