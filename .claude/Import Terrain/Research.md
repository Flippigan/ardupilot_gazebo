# Custom Model Import Research

## Problem Statement
Attempted to spawn a custom 3D model (`/home/finn/Downloads/model.sdf`) into the iris_runway simulation. Initial approach using the launch file with `ros_gz_sim create` node did not work - the model was not visible in the simulation.

## Solution Approach

**Final Working Solution:**
The custom model was **inlined directly into the Gazebo world file** using **primitive box geometry** as a placeholder for missing mesh files.

**Journey:**
1. ❌ **Attempt 1**: Spawn via launch file → Model not visible
2. ❌ **Attempt 2**: Inline with mesh file URIs → World failed to load (black screen)
3. ✅ **Attempt 3**: Inline with box geometry → **SUCCESS**

**Key Insight:** Gazebo's SDF parser requires all `file://` URI references to exist. Missing mesh files cause world loading failure, not just invisible models.

---

## Technical Details

### World File Modified
**File:** `/home/finn/Documents/ardu_ws/src/ardupilot_gz/ardupilot_gz_gazebo/worlds/iris_runway.sdf`

**Location in launch file:** Referenced at line 78 of `/home/finn/Documents/ardu_ws/src/ardupilot_gz/ardupilot_gz_bringup/launch/iris_runway.launch.py`
```python
pkg_project_gazebo = get_package_share_directory("ardupilot_gz_gazebo")
# ...
"gz_args": "-v4 -s -r " f'{Path(pkg_project_gazebo) / "worlds" / "iris_runway.sdf"}'
```

### Model Source
**Original:** `/home/finn/Downloads/model.sdf`

### Changes Made to Model Definition

#### 1. Fixed Static Tag Placement
**Issue:** In the original file, `<static>true</static>` was placed OUTSIDE the `<model>` tag, which is invalid SDF syntax.

**Before:**
```xml
<model name="test">
  <!-- model contents -->
</model>
<static>true</static>
```

**After:**
```xml
<model name="custom_test_model">
  <static>true</static>
  <!-- model contents -->
</model>
```

#### 2. Updated Mesh URI Paths
**Issue:** Mesh file references were relative paths that wouldn't resolve in the Gazebo environment.

**Changes:**
- `model.dae` � `file:///home/finn/Downloads/model.dae`
- `meshes/DiffuseMap_0.jpg.001` � `file:///home/finn/Downloads/meshes/DiffuseMap_0.jpg.001`

#### 3. Added Pose Tag
Added `<pose>5 0 0 0 0 0</pose>` to position the model 5 meters along the X-axis from the origin (away from the drone).

Format: `<pose>x y z roll pitch yaw</pose>`

---

## Implementation

### World File Insertion Point
The model was inserted at line 81 of `iris_runway.sdf`, after the iris drone include and before the closing `</world>` tag.

### Complete Inserted Code Block (WORKING VERSION)
Located at **lines 81-106** of `iris_runway.sdf`:

```xml
<!-- Custom model - placeholder using box geometry (original mesh file not available) -->
<model name="custom_test_model">
  <static>true</static>
  <pose>5 0 0.5 0 0 0</pose>
  <link name="testlink">
    <visual name="mainVisual">
      <geometry>
        <box>
          <size>1 1 1</size>
        </box>
      </geometry>
      <material>
        <ambient>0.8 0.8 0.8 1.0</ambient>
        <diffuse>1.0 1.0 1.0 1.0</diffuse>
        <specular>0.5 0.5 0.5 1.0</specular>
      </material>
    </visual>
    <collision name="collision">
      <geometry>
        <box>
          <size>1 1 1</size>
        </box>
      </geometry>
    </collision>
  </link>
</model>
```

**Key Features:**
- Uses simple box geometry (1x1x1 meters)
- No external file dependencies
- White material with proper lighting properties
- Static (non-physics) object
- Positioned at (5, 0, 0.5) - 5m along X-axis, 0.5m above ground

---

## Critical Issue Encountered: World Load Failure

### Problem
When using mesh references with `file://` URIs pointing to missing files, Gazebo **failed to load the entire world**:

```
[Err] Error Code 14: Parser configurations requested resolved uris,
      but uri [file:///home/finn/Downloads/model.dae] could not be resolved.
[Err] Error Code 9: Failed to load a world.
```

**Symptoms:**
- Gazebo GUI opened but displayed a black, unresponsive screen
- World loading process aborted
- All simulation processes failed

### Root Cause
Gazebo's SDF parser with URI resolution enabled **requires all referenced files to exist**. Missing mesh files caused a fatal parsing error rather than a graceful fallback.

---

## FINAL SOLUTION: Placeholder Geometry

### Approach
Replaced missing mesh geometry with **simple primitive shapes** that don't require external files.

### Implementation
Changed from mesh-based geometry to box-based geometry:

**Before (FAILED):**
```xml
<visual name="mainMeshNode">
  <geometry>
    <mesh>
      <uri>file:///home/finn/Downloads/model.dae</uri>
    </mesh>
  </geometry>
</visual>
```

**After (SUCCESS):**
```xml
<visual name="mainVisual">
  <geometry>
    <box>
      <size>1 1 1</size>
    </box>
  </geometry>
  <material>
    <ambient>0.8 0.8 0.8 1.0</ambient>
    <diffuse>1.0 1.0 1.0 1.0</diffuse>
    <specular>0.5 0.5 0.5 1.0</specular>
  </material>
</visual>
```

**Result:**
- ✅ World loads successfully
- ✅ Gazebo GUI renders correctly
- ✅ Model visible as a 1x1x1 meter white box
- ✅ Collision geometry functional
- ✅ Positioned at (5, 0, 0.5)

### Future Improvements
If mesh files become available:
1. Place all files in `/home/finn/Downloads/` matching the URI structure
2. Alternatively, convert to a proper Gazebo model:
   - Create directory: `/home/finn/Documents/ardu_ws/src/ardupilot_gazebo/models/custom_test_model/`
   - Add `model.config` file
   - Organize meshes in `meshes/` subdirectory
   - Update URIs to use `model://` protocol

---

## Testing

### Build Command
```bash
cd /home/finn/Documents/ardu_ws
colcon build --symlink-install
source install/setup.bash
```

### Launch Command
```bash
ros2 launch ardupilot_gz_bringup iris_runway.launch.py
```

### Expected Result (VERIFIED WORKING)
- ✅ Gazebo world loads successfully with runway and iris drone
- ✅ Custom model loads at position (5, 0, 0.5) as a white box
- ✅ Model visible in Gazebo scene tree as "custom_test_model"
- ✅ Gazebo GUI renders correctly (not black screen)
- ✅ Full collision geometry functional
- ✅ No error messages in terminal

### Verification Steps
1. ✅ Gazebo GUI opens and displays properly
2. ✅ Scene tree (left panel) shows "custom_test_model"
3. ✅ Navigate camera to position (5, 0, 0.5) to see white box
4. ✅ Terminal shows: `[Msg] World [map] initialized with [1ms] physics profile`
5. ✅ No URI resolution errors

---

## SDF Structure Reference

### Model Include vs Inline Definition

**Include method (used for runway and iris):**
```xml
<include>
  <uri>model://model_name</uri>
  <pose>x y z r p y</pose>
</include>
```

**Inline method (used for custom model):**
```xml
<model name="model_name">
  <static>true</static>
  <pose>x y z r p y</pose>
  <link name="link_name">
    <!-- visual and collision elements -->
  </link>
</model>
```

---

## File References

| Component | Path |
|-----------|------|
| **Launch file** | `/home/finn/Documents/ardu_ws/src/ardupilot_gz/ardupilot_gz_bringup/launch/iris_runway.launch.py` |
| **World file (source)** | `/home/finn/Documents/ardu_ws/src/ardupilot_gz/ardupilot_gz_gazebo/worlds/iris_runway.sdf` |
| **World file (installed)** | `/home/finn/Documents/ardu_ws/install/ardupilot_gz_gazebo/share/ardupilot_gz_gazebo/worlds/iris_runway.sdf` |
| **Mesh file** | `/home/finn/Documents/ardu_ws/meshes/model.dae` (71MB COLLADA file) |
| **Texture file** | `/home/finn/Documents/ardu_ws/meshes/DiffuseMap_0.jpg.001.jpg` (8192x8192 JPEG, 7.9MB) |
| **Original model** | `/home/finn/Downloads/model.sdf` (no longer exists) |
| **Package** | `ardupilot_gz_gazebo` |

### Build and Install Process

**Important:** The launch file references the **installed** world file, not the source file:
```python
pkg_project_gazebo = get_package_share_directory("ardupilot_gz_gazebo")
# This resolves to: /home/finn/Documents/ardu_ws/install/ardupilot_gz_gazebo/share/ardupilot_gz_gazebo/
```

**After modifying the source world file, you must rebuild:**
```bash
cd /home/finn/Documents/ardu_ws
colcon build --packages-select ardupilot_gz_gazebo --symlink-install
source install/setup.bash
```

The `--symlink-install` flag creates symlinks for Python files and some resources, but **world files (.sdf) are copied to the install directory**, not symlinked. This means you must rebuild after any changes to the world file.

---

## Conclusion

**Status: ✅ MESH FILES IMPLEMENTED**

The terrain model with actual mesh geometry has been successfully integrated into the iris_runway simulation world.

### Current Implementation (Updated)

**Model Name:** `custom_terrain_model`
**Location in World:** Lines 81-113 of `iris_runway.sdf`
**Position:** (5, 0, 0) - 5 meters along X-axis from origin

**Model Configuration:**
```xml
<model name="custom_terrain_model">
  <static>true</static>
  <pose>5 0 0 0 0 0</pose>
  <link name="terrain_link">
    <visual name="mainMeshNode">
      <geometry>
        <mesh>
          <uri>file:///home/finn/Documents/ardu_ws/meshes/model.dae</uri>
          <scale>0.01 0.01 0.01</scale>
        </mesh>
      </geometry>
      <material>
        <diffuse>1.0 1.0 1.0 1.0</diffuse>
        <specular>0.1 0.1 0.1 1.0</specular>
        <pbr>
          <metal>
            <albedo_map>file:///home/finn/Documents/ardu_ws/meshes/DiffuseMap_0.jpg.001.jpg</albedo_map>
          </metal>
        </pbr>
      </material>
    </visual>
    <collision name="collision">
      <geometry>
        <mesh>
          <uri>file:///home/finn/Documents/ardu_ws/meshes/model.dae</uri>
          <scale>0.01 0.01 0.01</scale>
        </mesh>
      </geometry>
    </collision>
  </link>
</model>
```

**Key Features:**
- ✅ Uses actual COLLADA mesh geometry (71MB .dae file)
- ✅ PBR texture with 8192x8192 albedo map
- ✅ Scaled down by 0.01x to compensate for large coordinate values in Blender export
- ✅ Static (non-physics) object for performance
- ✅ Full collision detection using mesh geometry
- ✅ No submesh specification (removed to prevent rendering issues)

**Key Learnings:**
1. **Critical**: Gazebo's SDF parser fails hard when `file://` URIs reference missing files
2. **Solution**: Ensure mesh and texture files exist before referencing them
3. **Best Practice**: Use `<scale>` tag to adjust mesh size when imported from Blender with non-standard units
4. **Important**: World files must be rebuilt after changes - they are copied to install directory, not symlinked

**Evolution of Implementation:**
1. ❌ **Initial**: Box placeholder (mesh files missing)
2. ✅ **Current**: Full mesh with texture (files located at `/home/finn/Documents/ardu_ws/meshes/`)

**Files Modified:**
- `/home/finn/Documents/ardu_ws/src/ardupilot_gz/ardupilot_gz_gazebo/worlds/iris_runway.sdf` (lines 81-113)
- **Must rebuild after changes:** `colcon build --packages-select ardupilot_gz_gazebo --symlink-install`

---

## Why is the Model a Cube?

### Question
Is the cube a function of what was downloaded, or did the implementer change the original model to a box?

### Answer: **It's a Placeholder Substitution**

The cube is **NOT** what was in the original model file. The implementer intentionally replaced the original mesh geometry with a simple box placeholder.

### Evidence from Original Model File

**File:** `/home/finn/Downloads/model.sdf`

The original model definition references **3D mesh files**, not box geometry:

```xml
<visual name="mainMeshNode">
  <geometry>
    <mesh>
      <uri>model.dae</uri>                    <!-- COLLADA mesh file -->
      <submesh>
        <name>mainMeshNode</name>
      </submesh>
    </mesh>
  </geometry>
  <material>
    <diffuse>1.0 1.0 1.0 1.0</diffuse>
    <specular>0.0 0.0 0.0 1.0</specular>
    <pbr>
      <metal>
        <albedo_map>meshes/DiffuseMap_0.jpg.001</albedo_map>  <!-- Texture file -->
      </metal>
    </pbr>
  </material>
</visual>
```

**Original Model Requirements:**
- `model.dae` - COLLADA 3D mesh file (the actual geometry)
- `meshes/DiffuseMap_0.jpg.001` - Albedo/texture map for material

### What's Missing

**Status of Required Files:**
- ✅ `model.sdf` - Model definition file (exists in `/home/finn/Downloads/`)
- ❌ `model.dae` - 3D mesh geometry file (MISSING)
- ❌ `meshes/DiffuseMap_0.jpg.001` - Texture file (MISSING)

### Why the Substitution Was Necessary

When the mesh files are missing and the model references them with `file://` URIs:
1. Gazebo's SDF parser attempts to resolve all file references
2. Missing files cause **fatal parsing errors** (Error Code 14)
3. The entire world fails to load (black screen, Error Code 9)
4. The simulation is unusable

**The implementer made a pragmatic decision:**
- Replace `<mesh>` geometry with `<box>` geometry
- Remove all external file dependencies
- Allow the world to load successfully
- Use the box as a visual placeholder

### What the Original Model Was Supposed to Look Like

Based on the SDF definition:
- A 3D mesh model with custom geometry defined in `model.dae`
- Textured with a PBR (Physically Based Rendering) material
- Uses metal workflow with an albedo map
- Full collision detection using the same mesh geometry

**The original model could have been:**
- Terrain features (rocks, hills, buildings)
- Obstacles for testing
- Any custom 3D asset exported to COLLADA format

### How to Restore the Original Model

If you obtain the complete model package:

1. **Verify you have all files:**
   ```bash
   ls -la /home/finn/Downloads/
   # Should show: model.sdf, model.dae, and meshes/DiffuseMap_0.jpg.001
   ```

2. **Update the world file** (`iris_runway.sdf` lines 81-106) to use mesh geometry:
   ```xml
   <visual name="mainMeshNode">
     <geometry>
       <mesh>
         <uri>file:///home/finn/Downloads/model.dae</uri>
         <submesh>
           <name>mainMeshNode</name>
         </submesh>
       </mesh>
     </geometry>
     <material>
       <diffuse>1.0 1.0 1.0 1.0</diffuse>
       <pbr>
         <metal>
           <albedo_map>file:///home/finn/Downloads/meshes/DiffuseMap_0.jpg.001</albedo_map>
         </metal>
       </pbr>
     </material>
   </visual>
   ```

3. **Rebuild and test:**
   ```bash
   colcon build --packages-select ardupilot_gz_gazebo --symlink-install
   source install/setup.bash
   ros2 launch ardupilot_gz_bringup iris_runway.launch.py
   ```

### Summary

| Aspect | Status |
|--------|--------|
| **Downloaded model type** | 3D mesh (COLLADA .dae format) |
| **Current implementation** | Box placeholder (1x1x1m cube) |
| **Reason for change** | Missing mesh files would crash Gazebo |
| **Can it be restored?** | Yes, if complete model files are obtained |

**Conclusion:** The cube is a **temporary placeholder** chosen by the implementer to keep the simulation functional while the actual mesh files are unavailable.

---

## Troubleshooting

### Issue: Gazebo GUI Black Screen / Not Responding

**Symptoms:**
- Gazebo window opens but shows only black screen
- GUI does not respond to input
- Terminal shows: `Error Code 14: Parser configurations requested resolved uris, but uri [...] could not be resolved`
- Terminal shows: `Error Code 9: Failed to load a world`

**Root Cause:**
The world SDF file contains `file://` URI references to mesh files that don't exist on the filesystem.

**Solution:**
Replace mesh geometry with primitive shapes (box, sphere, cylinder) that don't require external files.

**Example Fix:**
```xml
<!-- BEFORE (causes black screen) -->
<geometry>
  <mesh>
    <uri>file:///path/to/missing/file.dae</uri>
  </mesh>
</geometry>

<!-- AFTER (works) -->
<geometry>
  <box>
    <size>1 1 1</size>
  </box>
</geometry>
```

**Rebuild After Fix:**
```bash
cd /home/finn/Documents/ardu_ws
colcon build --packages-select ardupilot_gz_gazebo --symlink-install
source install/setup.bash
ros2 launch ardupilot_gz_bringup iris_runway.launch.py rviz:=false
```
