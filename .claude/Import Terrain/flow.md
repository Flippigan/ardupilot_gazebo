# Custom Mesh Import Dataflow Analysis

## Executive Summary

**Problem:** Custom terrain mesh (71MB COLLADA file with 8K texture) loads without errors but **does not appear** in Gazebo simulation.

**Root Cause:** Photogrammetry reconstruction mesh has vertex coordinates offset 300+ meters from local origin. Even with 0.01 scale factor, geometry is positioned 3+ meters away from the model's pose location.

**Status:** World file configured correctly, mesh file accessible, but geometry not visible due to coordinate offset.

---

## Complete Dataflow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        MESH FILES                                │
│  /home/finn/Documents/ardu_ws/meshes/                           │
│    ├── model.dae (71MB COLLADA, vertices at 300+ meters)       │
│    └── DiffuseMap_0.jpg.001.jpg (8192x8192 PBR texture)        │
└─────────────────────────────────────────────────────────────────┘
                              ↓ Referenced via file:// URI
┌─────────────────────────────────────────────────────────────────┐
│                   WORLD FILE (SOURCE)                            │
│  /home/finn/Documents/ardu_ws/src/ardupilot_gz/                │
│    ardupilot_gz_gazebo/worlds/iris_runway.sdf                  │
│                                                                  │
│  <model name="custom_terrain_model">                            │
│    <pose>5 0 0 0 0 0</pose>  ← Model origin at 5m X-axis      │
│    <mesh>                                                        │
│      <uri>file:///.../meshes/model.dae</uri>                   │
│      <scale>0.01 0.01 0.01</scale>  ← 1% scale                 │
│    </mesh>                                                       │
│  </model>                                                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓ colcon build
┌─────────────────────────────────────────────────────────────────┐
│                  WORLD FILE (INSTALLED)                          │
│  /home/finn/Documents/ardu_ws/install/                          │
│    ardupilot_gz_gazebo/share/ardupilot_gz_gazebo/              │
│    worlds/iris_runway.sdf                                       │
│                                                                  │
│  [Copied during build - NOT symlinked!]                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓ Referenced by launch file
┌─────────────────────────────────────────────────────────────────┐
│                      LAUNCH FILE                                 │
│  /home/finn/Documents/ardu_ws/src/ardupilot_gz/                │
│    ardupilot_gz_bringup/launch/iris_runway.launch.py           │
│                                                                  │
│  pkg_project_gazebo = get_package_share_directory(             │
│      "ardupilot_gz_gazebo"                                      │
│  )  # Resolves to install/ardupilot_gz_gazebo/share/...        │
│                                                                  │
│  gz_args: "-v4 -s -r {pkg_project_gazebo}/worlds/iris_runway.sdf" │
└─────────────────────────────────────────────────────────────────┘
                              ↓ ros2 launch
┌─────────────────────────────────────────────────────────────────┐
│                   GAZEBO SIMULATION                              │
│                                                                  │
│  1. gz sim server loads world file                             │
│  2. Parses SDF, finds custom_terrain_model                     │
│  3. Resolves file:// URI to mesh file                          │
│  4. Loads COLLADA file with ogre2 renderer                     │
│  5. Applies 0.01 scale to all vertices                         │
│  6. Places model at pose (5, 0, 0)                             │
│                                                                  │
│  ISSUE: Actual geometry at (5 + 3, 0, 0) or further!          │
│         Camera at (0, 0, 2) cannot see offset geometry         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Component Breakdown

### 1. Mesh Files

**Location:** `/home/finn/Documents/ardu_ws/meshes/`

**model.dae (COLLADA 1.4.1)**
- **Size:** 71,765,953 bytes (71MB)
- **Format:** COLLADA (Digital Asset Exchange)
- **Created:** Blender 4.5.3 LTS
- **Type:** Photogrammetry reconstruction
- **Coordinate System:** Z_UP, meters
- **Geometry:**
  - Name: `mainMesh_001-mesh`
  - Vertices: 248,771 (748,314 float values)
  - **Coordinate Range:** X: 260-470, Y: -400 to 40, Z: -270 to -180
  - **Critical Issue:** Vertices NOT centered at origin!

**Example vertex coordinates:**
```
312.792, -326.5077, -268.6721
433.3629, 31.41583, -187.222
266.4038, -395.5083, -240.9587
```

**Visual Scene Structure:**
- 1 main mesh node: `mainMeshNode`
- 200+ camera nodes (photogrammetry metadata)
- Transform matrix: Identity (no translation/rotation)

**DiffuseMap_0.jpg.001.jpg**
- **Size:** 7,925,035 bytes (7.9MB)
- **Format:** JPEG
- **Resolution:** 8192 × 8192 pixels
- **Usage:** PBR albedo map for metal workflow

### 2. World Files

#### Source File
**Path:** `/home/finn/Documents/ardu_ws/src/ardupilot_gz/ardupilot_gz_gazebo/worlds/iris_runway.sdf`

**Purpose:** Developer-editable world definition

**Model Definition (lines 81-113):**
```xml
<model name="custom_terrain_model">
  <static>true</static>
  <pose>5 0 0 0 0 0</pose>  <!-- X=5m, Y=0, Z=0, no rotation -->
  <link name="terrain_link">
    <visual name="mainMeshNode">
      <geometry>
        <mesh>
          <uri>file:///home/finn/Documents/ardu_ws/meshes/model.dae</uri>
          <scale>0.01 0.01 0.01</scale>  <!-- 1% of original size -->
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

#### Installed File
**Path:** `/home/finn/Documents/ardu_ws/install/ardupilot_gz_gazebo/share/ardupilot_gz_gazebo/worlds/iris_runway.sdf`

**Purpose:** Runtime world file used by Gazebo

**Install Mechanism:** Copied by CMake during `colcon build`
```cmake
install(
  DIRECTORY worlds/
  DESTINATION share/${PROJECT_NAME}/worlds
)
```

**Critical Note:** World files are **COPIED, not symlinked!** Changes to source require rebuild.

### 3. Build System

**CMakeLists.txt:** `/home/finn/Documents/ardu_ws/src/ardupilot_gazebo/CMakeLists.txt`

**Relevant Install Directives:**
```cmake
install(DIRECTORY models/ DESTINATION share/${PROJECT_NAME}/models)
install(DIRECTORY worlds/ DESTINATION share/${PROJECT_NAME}/worlds)
install(DIRECTORY config/ DESTINATION share/${PROJECT_NAME}/config)
```

**Build Command:**
```bash
cd /home/finn/Documents/ardu_ws
colcon build --packages-select ardupilot_gz_gazebo --symlink-install
```

**Build Output:**
- Source worlds/ → Installed to `install/ardupilot_gz_gazebo/share/ardupilot_gz_gazebo/worlds/`
- `--symlink-install` applies to Python files, **NOT SDF files**
- World files always copied

### 4. Launch System

**Launch File:** `/home/finn/Documents/ardu_ws/src/ardupilot_gz/ardupilot_gz_bringup/launch/iris_runway.launch.py`

**World File Resolution:**
```python
from ament_index_python.packages import get_package_share_directory

pkg_project_gazebo = get_package_share_directory("ardupilot_gz_gazebo")
# Returns: /home/finn/Documents/ardu_ws/install/ardupilot_gz_gazebo/share/ardupilot_gz_gazebo

gz_args = "-v4 -s -r " + f'{Path(pkg_project_gazebo) / "worlds" / "iris_runway.sdf"}'
# Expands to: -v4 -s -r /home/finn/.../install/.../worlds/iris_runway.sdf
```

**Gazebo Sim Server Launch:**
```python
gz_sim_server = IncludeLaunchDescription(
    PythonLaunchDescriptionSource(
        f'{Path(pkg_ros_gz_sim) / "launch" / "gz_sim.launch.py"}'
    ),
    launch_arguments={"gz_args": gz_args}.items(),
)
```

**Parameters:**
- `-v4`: Verbosity level 4 (debug)
- `-s`: Run server
- `-r`: Run immediately (no pause)

### 5. Gazebo Rendering Pipeline

**Rendering Engine:** ogre2 (configured in world file)
```xml
<plugin filename="gz-sim-sensors-system" name="gz::sim::systems::Sensors">
  <render_engine>ogre2</render_engine>
</plugin>
```

**Mesh Loading Process:**
1. **URI Resolution:** `file:///home/finn/Documents/ardu_ws/meshes/model.dae`
2. **COLLADA Parser:** Reads DAE XML structure
3. **Geometry Extraction:** Loads `mainMesh_001-mesh` vertices
4. **Transform Application:**
   - Original vertex: (312.792, -326.5077, -268.6721)
   - After 0.01 scale: (3.12792, -3.265077, -2.686721)
   - After model pose: (5 + 3.12792, -3.265077, -2.686721) = **(8.12792, -3.265077, -2.686721)**
5. **Material Application:** PBR metal workflow with albedo map
6. **Scene Placement:** Entity added to world scene graph

**Camera Default Position:** Typically (0, 0, 2) looking at origin

**Visibility Issue:**
- Model pose: (5, 0, 0)
- Actual geometry: (8+, -3+, -2+)
- Camera cannot see offset geometry without manual navigation

---

## URI Resolution Patterns

### Current Approach: file:// URIs

**Used in World File:**
```xml
<uri>file:///home/finn/Documents/ardu_ws/meshes/model.dae</uri>
```

**Characteristics:**
- ✅ Simple absolute path
- ✅ Works for external files
- ❌ Not portable across systems
- ❌ Requires absolute paths
- ❌ No package integration

### Standard Approach: package:// URIs

**Used by Existing Models:**
```xml
<uri>package://ardupilot_gazebo/models/iris_with_standoffs/meshes/iris.dae</uri>
```

**Resolution Mechanism:**
1. `package://ardupilot_gazebo` → Resolved via ROS package system
2. Maps to: `/home/finn/Documents/ardu_ws/install/ardupilot_gazebo/share/ardupilot_gazebo/`
3. Appends: `models/iris_with_standoffs/meshes/iris.dae`
4. Final path: `install/ardupilot_gazebo/share/ardupilot_gazebo/models/iris_with_standoffs/meshes/iris.dae`

**Characteristics:**
- ✅ Portable across systems
- ✅ Integrated with ROS build system
- ✅ Package dependencies handled
- ✅ Standard convention
- ❌ Requires proper package structure

### Proper Model Package Structure

**Directory Layout:**
```
src/ardupilot_gazebo/models/custom_terrain/
├── model.config          # Gazebo model metadata
├── model.sdf            # Model definition (can reference in world files)
└── meshes/
    ├── model.dae        # Mesh geometry
    └── DiffuseMap_0.jpg # Texture
```

**model.config Example:**
```xml
<?xml version="1.0"?>
<model>
  <name>custom_terrain</name>
  <version>1.0</version>
  <sdf version="1.9">model.sdf</sdf>
  <author>
    <name>Your Name</name>
    <email>email@example.com</email>
  </author>
  <description>Photogrammetry terrain model</description>
</model>
```

**Usage in World File (with package structure):**
```xml
<include>
  <uri>package://ardupilot_gazebo/models/custom_terrain</uri>
  <pose>5 0 0 0 0 0</pose>
</include>
```

---

## Identified Issues

### Issue 1: Geometry Offset (PRIMARY ISSUE)

**Problem:** Mesh vertices are positioned far from local origin

**Evidence:**
- Vertex coordinates: 260-470 (X), -400 to 40 (Y), -270 to -180 (Z)
- Center of geometry: Approximately (350, -180, -225) in Blender units
- After 0.01 scale: (3.5, -1.8, -2.25) meters from local origin
- After model pose (5, 0, 0): Geometry at (8.5, -1.8, -2.25)

**Impact:**
- Model loads successfully
- Entity appears in Gazebo scene tree
- Geometry is rendered but far from expected location
- Default camera view cannot see the geometry

**Verification:**
1. Launch simulation: `ros2 launch ardupilot_gz_bringup iris_runway.launch.py`
2. Check entity tree (left panel): "custom_terrain_model" should be listed
3. Navigate camera to (8, -2, -2) - geometry might be visible
4. Or navigate to (5, 0, 0) - will only see empty space at model origin

### Issue 2: Scale Factor Insufficient

**Problem:** 0.01 scale still leaves large offsets

**Calculation:**
- Original offset: 350 meters
- With 0.01 scale: 3.5 meters
- With 0.001 scale: 0.35 meters (better, but mesh will be 1/10 size)
- With 0.0001 scale: 0.035 meters (3.5cm offset, very small mesh)

**Trade-off:** Smaller scale = smaller offset BUT also smaller visible mesh

### Issue 3: Material Compatibility

**Problem:** PBR metal workflow may not render correctly in ogre2

**Current Material:**
```xml
<material>
  <diffuse>1.0 1.0 1.0 1.0</diffuse>
  <specular>0.1 0.1 0.1 1.0</specular>
  <pbr>
    <metal>
      <albedo_map>file:///.../DiffuseMap_0.jpg.001.jpg</albedo_map>
    </metal>
  </pbr>
</material>
```

**Potential Issues:**
- Ogre2 PBR support may be limited
- Metal workflow might not display textures correctly
- 8K texture might cause performance/memory issues

**Test:** Simplify to basic diffuse material:
```xml
<material>
  <ambient>0.8 0.8 0.8 1.0</ambient>
  <diffuse>0.8 0.8 0.8 1.0</diffuse>
  <specular>0.2 0.2 0.2 1.0</specular>
</material>
```

### Issue 4: Photogrammetry Artifacts

**Problem:** COLLADA file contains 200+ camera nodes

**Evidence:**
```xml
<node id="ER_02_21260_0129_0347_JPG" name="ER_02_21260_0129_0347_JPG" type="NODE">
<node id="ER_02_21260_0129_0346_JPG" name="ER_02_21260_0129_0346_JPG" type="NODE">
... (200+ more camera nodes)
```

**Impact:**
- Increases file size
- Potentially confuses Gazebo parser
- Not needed for rendering (only mesh geometry used)

**Solution:** Clean up COLLADA file in Blender:
1. Open model.dae in Blender
2. Delete all camera objects
3. Select only mesh
4. Export as new COLLADA file

---

## Debugging Checklist

### Phase 1: Verify Installation

- [x] Source world file contains mesh reference
- [x] Installed world file matches source (rebuild completed)
- [x] Mesh files exist at specified paths
- [x] Texture file exists and is readable
- [ ] Launch simulation without errors
- [ ] Check Gazebo entity tree shows "custom_terrain_model"

**Commands:**
```bash
# Verify installed world has changes
cat /home/finn/Documents/ardu_ws/install/ardupilot_gz_gazebo/share/ardupilot_gz_gazebo/worlds/iris_runway.sdf | grep -A 5 "custom_terrain"

# Verify mesh exists
ls -lh /home/finn/Documents/ardu_ws/meshes/model.dae

# Launch and check logs
ros2 launch ardupilot_gz_bringup iris_runway.launch.py 2>&1 | grep -i "mesh\|terrain\|error"
```

### Phase 2: Camera Navigation Test

- [ ] Open Gazebo GUI
- [ ] Locate "custom_terrain_model" in entity tree (left panel)
- [ ] Right-click model → "Move To"
- [ ] Manual camera navigation:
  - Press and hold right mouse button
  - Use WASD keys to fly camera
  - Navigate to (5, 0, 0) first
  - Then search nearby area (8, -2, -2)
- [ ] Check if geometry is visible anywhere

**Alternative:** Use Gazebo camera controls
```
Rotate: Left-click + drag
Pan: Shift + Left-click + drag
Zoom: Scroll wheel
Fly: Right-click + drag + WASD
```

### Phase 3: Material Simplification

- [ ] Remove PBR material
- [ ] Use basic diffuse/ambient only
- [ ] Rebuild and test
- [ ] Check if rendering improves

**Edit world file to:**
```xml
<material>
  <ambient>0.8 0.8 0.8 1.0</ambient>
  <diffuse>0.8 0.8 0.8 1.0</diffuse>
  <specular>0.2 0.2 0.2 1.0</specular>
</material>
```

### Phase 4: Geometry Analysis

- [ ] Open model.dae in Blender
- [ ] Check mesh position relative to origin
- [ ] Measure bounding box center
- [ ] Verify vertex coordinate ranges
- [ ] Check for multiple meshes/objects

**Blender Commands:**
1. File → Import → COLLADA (.dae)
2. Select mesh object
3. Object Properties → Transform → Location (should be near 0,0,0)
4. Properties → Object → Dimensions

### Phase 5: Mesh Preprocessing

- [ ] Center mesh in Blender
- [ ] Scale mesh appropriately
- [ ] Clean up camera nodes
- [ ] Re-export as COLLADA
- [ ] Test with centered mesh

**Blender Centering Process:**
1. Select mesh in Object Mode
2. Object → Set Origin → Geometry to Origin
3. Alt+G (clear location - moves object to world origin)
4. Object → Apply → All Transforms
5. File → Export → COLLADA (.dae)

---

## Alternative Approaches

### Option A: Re-export Mesh from Blender (RECOMMENDED)

**Advantages:**
- ✅ Fixes geometry offset permanently
- ✅ Can clean up unnecessary camera nodes
- ✅ Can optimize mesh size
- ✅ No complex SDF workarounds needed

**Process:**
1. **Import mesh:**
   ```
   Blender → File → Import → COLLADA → model.dae
   ```

2. **Center geometry:**
   ```
   Select mesh → Object → Set Origin → Geometry to Origin
   Press Alt+G (clears location)
   Object → Apply → All Transforms
   ```

3. **Delete camera nodes:**
   ```
   Select all cameras in Outliner
   Press X → Delete
   ```

4. **Scale if needed:**
   ```
   Press S, then type scale factor (e.g., 0.01), Enter
   Object → Apply → Scale
   ```

5. **Export:**
   ```
   File → Export → COLLADA (.dae)
   Settings:
   - Apply Modifiers: On
   - Selection Only: Off (or select mesh first)
   - Include UV Textures: On
   - Include Material Textures: On
   ```

6. **Replace file:**
   ```bash
   mv /home/finn/Documents/ardu_ws/meshes/model.dae /home/finn/Documents/ardu_ws/meshes/model_backup.dae
   cp ~/exported_model.dae /home/finn/Documents/ardu_ws/meshes/model.dae
   ```

7. **Update SDF to remove scale or use 1:1:**
   ```xml
   <scale>1 1 1</scale>  <!-- or remove scale tag entirely -->
   ```

### Option B: Add Large Pose Translation

**Advantages:**
- ✅ No mesh modification needed
- ✅ Quick test to verify geometry location

**Disadvantages:**
- ❌ Hardcoded offset - not portable
- ❌ Doesn't fix root cause
- ❌ Collision geometry also offset

**Implementation:**
```xml
<model name="custom_terrain_model">
  <static>true</static>
  <!-- Compensate for mesh offset: pose includes offset correction -->
  <pose>2 1.8 2.25 0 0 0</pose>  <!-- 5 - 3.5 + offset adjustments -->
  ...
</model>
```

**Calculation:**
1. Determine mesh center from Blender: (~350, -180, -225)
2. Apply scale: (3.5, -1.8, -2.25)
3. Desired final position: (5, 0, 0)
4. Required pose: (5 - 3.5, 0 - (-1.8), 0 - (-2.25)) = **(1.5, 1.8, 2.25)**

### Option C: Create Proper Gazebo Model Package

**Advantages:**
- ✅ Follows Gazebo conventions
- ✅ Portable and reusable
- ✅ Can be included in other worlds
- ✅ Proper package:// URI resolution

**Disadvantages:**
- ❌ More setup required
- ❌ Still need to fix mesh offset

**Structure:**
```
src/ardupilot_gazebo/models/custom_terrain/
├── model.config
├── model.sdf
└── meshes/
    ├── model.dae
    └── DiffuseMap_0.jpg.001.jpg
```

**model.config:**
```xml
<?xml version="1.0"?>
<model>
  <name>custom_terrain</name>
  <version>1.0</version>
  <sdf version="1.9">model.sdf</sdf>
  <author>
    <name>Finn</name>
  </author>
  <description>
    Photogrammetry-reconstructed terrain model with 8K PBR texture
  </description>
</model>
```

**model.sdf:**
```xml
<?xml version="1.0"?>
<sdf version="1.9">
  <model name="custom_terrain">
    <static>true</static>
    <link name="terrain_link">
      <visual name="visual">
        <geometry>
          <mesh>
            <uri>package://ardupilot_gazebo/models/custom_terrain/meshes/model.dae</uri>
            <scale>0.01 0.01 0.01</scale>
          </mesh>
        </geometry>
        <material>
          <diffuse>1.0 1.0 1.0 1.0</diffuse>
          <specular>0.1 0.1 0.1 1.0</specular>
          <pbr>
            <metal>
              <albedo_map>package://ardupilot_gazebo/models/custom_terrain/meshes/DiffuseMap_0.jpg.001.jpg</albedo_map>
            </metal>
          </pbr>
        </material>
      </visual>
      <collision name="collision">
        <geometry>
          <mesh>
            <uri>package://ardupilot_gazebo/models/custom_terrain/meshes/model.dae</uri>
            <scale>0.01 0.01 0.01</scale>
          </mesh>
        </geometry>
      </collision>
    </link>
  </model>
</sdf>
```

**World file inclusion:**
```xml
<world>
  ...
  <include>
    <uri>package://ardupilot_gazebo/models/custom_terrain</uri>
    <name>terrain</name>
    <pose>5 0 0 0 0 0</pose>
  </include>
  ...
</world>
```

**Move files:**
```bash
# Create model directory
mkdir -p /home/finn/Documents/ardu_ws/src/ardupilot_gazebo/models/custom_terrain/meshes

# Move mesh files
mv /home/finn/Documents/ardu_ws/meshes/model.dae \
   /home/finn/Documents/ardu_ws/src/ardupilot_gazebo/models/custom_terrain/meshes/

mv /home/finn/Documents/ardu_ws/meshes/DiffuseMap_0.jpg.001.jpg \
   /home/finn/Documents/ardu_ws/src/ardupilot_gazebo/models/custom_terrain/meshes/

# Rebuild
cd /home/finn/Documents/ardu_ws
colcon build --packages-select ardupilot_gazebo --symlink-install
```

### Option D: Use Mesh Preprocessing Tool

**Advantages:**
- ✅ Automated centering
- ✅ Can simplify mesh
- ✅ Can fix other issues

**Tools:**
- MeshLab: Open-source mesh processing
- Blender: Full 3D suite with scripting
- gz-tools: Gazebo mesh utilities

**MeshLab Process:**
```bash
# Install
sudo apt install meshlab

# Open mesh
meshlab model.dae

# Center
Filters → Normals, Curvatures and Orientation → Transform: Center
Filters → Mesh Layer → Flatten Visible Layers

# Export
File → Export Mesh As → model_centered.dae
```

---

## Working Examples Reference

### Example 1: iris_with_standoffs Model

**Location:** `/home/finn/Documents/ardu_ws/src/ardupilot_gazebo/models/iris_with_standoffs/`

**Structure:**
```
iris_with_standoffs/
├── model.config         # Model metadata
├── model.sdf           # Model definition
└── meshes/
    ├── iris.dae        # Main body mesh
    ├── iris_prop_ccw.dae    # Propeller mesh
    └── iris_prop_cw.dae     # Propeller mesh
```

**model.sdf mesh reference:**
```xml
<visual name='base_visual'>
  <geometry>
    <mesh>
      <uri>package://ardupilot_gazebo/models/iris_with_standoffs/meshes/iris.dae</uri>
    </mesh>
  </geometry>
  <material>
    <ambient>0.05 0.05 0.05</ambient>
    <diffuse>0.05 0.05 0.05</diffuse>
    <specular>1 1 1 1</specular>
    <pbr>
      <metal>
        <metalness>0.8</metalness>
        <roughness>0.2</roughness>
      </metal>
    </pbr>
  </material>
</visual>
```

**Key Observations:**
- Uses `package://` URI scheme
- Material uses simple PBR without texture maps
- Meshes are properly centered in Blender (vertices near origin)
- Scale not needed (meshes exported at correct size)

### Example 2: runway Model

**Location:** `/home/finn/Documents/ardu_ws/src/ardupilot_gazebo/models/runway/`

**Usage in World:**
```xml
<include>
  <uri>model://runway</uri>
  <pose degrees="true">-29 545 0 0 0 363</pose>
</include>
```

**Key Observations:**
- Uses `model://` URI (Gazebo's model database protocol)
- Simple include with pose override
- No inline geometry definition needed

### Example 3: Current Custom Terrain (Not Working)

**Differences from Working Examples:**
- ❌ Uses `file://` instead of `package://`
- ❌ Inlined in world file instead of separate model package
- ❌ Mesh not centered (300+ meter offset)
- ❌ Large scale factor needed (0.01)
- ❌ Complex PBR material with 8K texture

---

## Recommended Solution Path

### Priority 1: Fix Mesh Geometry (ROOT CAUSE)

**Steps:**
1. Open `/home/finn/Documents/ardu_ws/meshes/model.dae` in Blender
2. Select mesh object
3. Object → Set Origin → Geometry to Origin
4. Alt+G (clear location - moves to world origin)
5. Delete all camera objects
6. Object → Apply → All Transforms
7. File → Export → COLLADA (.dae) → Replace existing file

**Expected Result:** Mesh vertices centered near (0, 0, 0)

### Priority 2: Test with Simplified Configuration

**Remove scale and complex material:**
```xml
<model name="custom_terrain_model">
  <static>true</static>
  <pose>5 0 0 0 0 0</pose>
  <link name="terrain_link">
    <visual name="mainMeshNode">
      <geometry>
        <mesh>
          <uri>file:///home/finn/Documents/ardu_ws/meshes/model.dae</uri>
          <!-- Scale removed - mesh should be at correct size after Blender fix -->
        </mesh>
      </geometry>
      <material>
        <ambient>0.8 0.8 0.8 1.0</ambient>
        <diffuse>0.8 0.8 0.8 1.0</diffuse>
        <specular>0.2 0.2 0.2 1.0</specular>
      </material>
    </visual>
  </link>
</model>
```

**Rebuild and test:**
```bash
cd /home/finn/Documents/ardu_ws
colcon build --packages-select ardupilot_gz_gazebo --symlink-install
source install/setup.bash
ros2 launch ardupilot_gz_bringup iris_runway.launch.py
```

### Priority 3: Convert to Proper Gazebo Model (LONG-TERM)

Once mesh is centered and working:
1. Create model directory structure
2. Add model.config
3. Move mesh to model package
4. Convert to package:// URIs
5. Include in world file instead of inlining

---

## Summary

### Current Status
- ✅ Mesh file exists and is valid
- ✅ World file configured correctly
- ✅ Build system working
- ✅ Launch system working
- ✅ Gazebo loads world without errors
- ❌ Geometry not visible due to offset

### Root Cause
**Photogrammetry mesh has vertices 300+ meters from local origin.** Even with 0.01 scale, geometry is positioned 3+ meters away from model's pose location.

### Immediate Solution
**Re-export mesh from Blender with centered geometry:**
1. Import model.dae
2. Center mesh to origin
3. Delete camera nodes
4. Apply transforms
5. Export as new COLLADA file
6. Test in Gazebo

### Long-Term Solution
**Convert to proper Gazebo model package with package:// URIs** for portability and convention compliance.

### Verification Method
After mesh centering:
1. Launch simulation
2. Check entity tree for "custom_terrain_model"
3. Navigate camera to (5, 0, 0)
4. Geometry should be visible at model pose location
