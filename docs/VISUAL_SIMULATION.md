# NeRF Visual Simulation Guide

This guide covers how to use the NeRF viewport extension for visual simulation in Isaac Sim, including camera animation and robot integration.

## Basic Setup

1. Launch Isaac Sim with the NeRF extension enabled:
   ```sh
   /isaac-sim/isaac-sim.sh --ext-folder /path/to/omni-nerf-extension/extension/exts --enable omni.nerf.viewport
   ```

2. Create a **Plane** mesh as your ground reference: `Create > Mesh > Plane`

3. Select the plane in the Stage and click **Set** in the NeRF Viewport window to bind it as the reference object

## How It Works

The NeRF viewport tracks the **active viewport camera** position relative to your reference mesh. As you move the camera (manually or via simulation), the NeRF renders the corresponding view.

The reference mesh defines the coordinate system origin for the NeRF - position it to match where the NeRF's origin should be in your scene.

## Simulating a Roving Camera

### Option 1: Animated Camera Path

1. **Create a Camera**: `Create > Camera`
2. **Set it as active**: Right-click camera in Stage > "Set as Active Camera"
3. **Create animation keyframes**:
   - Move timeline to frame 0, position camera, right-click transform > "Set Key"
   - Move to frame 100, reposition camera, set another key
4. **Play simulation**: Press Play - NeRF viewport updates as camera moves

### Option 2: Camera on a Physics-Driven Object

1. Create a simple collision mesh (cube/box) for your "drone" or vehicle
2. Add **Rigid Body** physics: `Right-click > Add > Physics > Rigid Body`
3. Parent a **Camera** to this object
4. Apply forces or set velocities via script/Action Graph
5. The camera follows the physics object, NeRF renders the view

## Adding a Robot with Physics

Since NeRF provides visual-only rendering, you need to create manual collision geometry for physics simulation.

### 1. Create Collision Geometry

Create simplified meshes that approximate the ground and obstacles in your NeRF scene:
```
Create > Mesh > Cube  (or import simplified collision geometry)
```
Scale and position to match the approximate layout of your NeRF environment.

### 2. Set Up Ground Plane Physics

- Select your plane mesh
- `Right-click > Add > Physics > Collision` (makes it a static collider)

### 3. Import/Create Robot

- Import a robot URDF/USD, or create a simple wheeled box
- Add **Articulation Root** for jointed robots
- Add **Rigid Body** + **Collision** to robot parts

### 4. Attach Camera to Robot

- Create a Camera, parent it to the robot base or a sensor mount
- Set as active viewport camera
- The NeRF viewport will show the robot's "eye view"

### 5. Control the Robot

- Use **Action Graph** for keyboard/gamepad control
- Or write a Python script using `omni.isaac.core` APIs
- Or connect via ROS2 for external control

## Quick Test: Keyboard-Controlled Camera Rig

1. `Create > Mesh > Cube` (scale to ~0.5m, this is your "vehicle")
2. Add Rigid Body + Collision to the cube
3. `Create > Camera`, parent under cube
4. Set camera as active viewport camera
5. Use Action Graph with "On Keyboard Input" nodes to apply forces to cube
6. Press Play - drive around, NeRF follows!

## Camera Controls (Manual Navigation)

When manually navigating the viewport:

- **Right-click + drag**: Look around
- **Right-click + WASD**: Move (W forward, S backward, A left, D right)
- **Right-click + Q/E**: Move down/up
- **Scroll wheel**: Zoom in/out
- **Alt + left-click drag**: Orbit around a point

Adjust camera speed in the viewport settings (gear icon) if movement is too fast/slow.

## Tips

- For flat outdoor scenes, use a **Plane** as reference mesh
- For indoor/enclosed scenes, consider a **Cube** positioned at room center
- The NeRF renders progressively - lower resolution first, then refines
- Performance depends on the remote renderer GPU and network latency
