# Execution Plan: Fix Missing Terrain Model in Gazebo Simulation

## Problem Summary

**Current State:**
- Custom terrain model (`custom_terrain_model`) loads into Gazebo without errors
- Model appears in Gazebo's entity tree (scene hierarchy)
- **ISSUE**: Geometry is NOT VISIBLE in the simulation viewport

**Root Cause:**
The photogrammetry-reconstructed mesh has vertices positioned 300+ meters from the local origin in Blender space. After applying the 0.01 scale factor, the geometry is still offset 3+ meters from the model's pose location, making it invisible to the default camera view.

**Evidence:**
- Original vertex coordinates: X: 260-470, Y: -400 to 40, Z: -270 to -180 (meters in Blender)
- After 0.01 scale: Geometry center at approximately (3.5, -1.8, -2.25) meters from local origin
- Model pose in world: (5, 0, 0)
- Actual geometry location: (8.5, -1.8, -2.25) - beyond default camera frustum

---

## Execution Strategy Overview

**Three-Phase Approach:**

1. **Phase 1: Rapid Verification** (5 minutes)
   - Confirm the issue via camera navigation
   - Verify current installation state

2. **Phase 2: Mesh Correction** (30 minutes)
   - Re-center mesh geometry in Blender
   - Clean up unnecessary data
   - Re-export optimized mesh

3. **Phase 3: Integration & Validation** (15 minutes)
   - Update world configuration
   - Rebuild and test
   - Optional: Convert to proper Gazebo model package

---

## Phase 1: Rapid Verification (CRITICAL)

### Objective
Confirm that the geometry exists but is offset before investing time in mesh correction.

### Step 1.1: Verify Current Installation

```bash
# Check installed world file has terrain model
cat /home/finn/Documents/ardu_ws/install/ardupilot_gz_gazebo/share/ardupilot_gz_gazebo/worlds/iris_runway.sdf | grep -A 10 "custom_terrain_model"

# Verify mesh files exist and are readable
ls -lh /home/finn/Documents/ardu_ws/meshes/model.dae
ls -lh /home/finn/Documents/ardu_ws/meshes/DiffuseMap_0.jpg.001.jpg

# Check file permissions
file /home/finn/Documents/ardu_ws/meshes/model.dae
```

**Expected Output:**
- ✅ World file contains terrain model definition at lines 81-113
- ✅ model.dae exists (71MB COLLADA file)
- ✅ DiffuseMap texture exists (7.9MB JPEG, 8192x8192)

### Step 1.2: Launch Simulation with Verbose Logging

```bash
cd /home/finn/Documents/ardu_ws
source install/setup.bash
ros2 launch ardupilot_gz_bringup iris_runway.launch.py 2>&1 | tee /tmp/gazebo_launch.log
```

**What to Look For:**
- ✅ No "Error Code 14" (URI resolution failure)
- ✅ No "Error Code 9" (world load failure)
- ✅ Gazebo GUI opens without black screen
- ⚠️ Check for texture loading warnings (8K texture may cause issues)

### Step 1.3: Camera Navigation Test

**In Gazebo GUI:**

1. **Verify entity exists:**
   - Open left panel (Scene Graph)
   - Expand "iris_runway" world
   - Look for "custom_terrain_model" entity
   - **Expected**: Entity should be listed with "terrain_link" child

2. **Navigate to model pose:**
   - Right-click on "custom_terrain_model" in entity tree
   - Select "Move To" (moves camera to model origin)
   - **Expected**: Camera moves to (5, 0, 0) but sees EMPTY SPACE

3. **Search for offset geometry:**
   - Use camera fly mode (Right-click + WASD)
   - Navigate to estimated geometry location: (8, -2, -2)
   - Pan around area from (7, -4, -3) to (10, 0, 0)
   - **Expected**: Geometry should be visible somewhere in this region

4. **Alternative verification:**
   - Open Gazebo View → Transparent mode
   - Check if transparent collision geometry is visible
   - Open Gazebo View → Wireframe mode

### Step 1.4: Decision Point

**If geometry is found offset:**
→ **Proceed to Phase 2** (mesh correction)

**If geometry is NOT found anywhere:**
→ **STOP**: Issue is different than expected. Possible causes:
- Mesh file corrupted
- Material/texture issue preventing rendering
- Ogre2 renderer compatibility problem
- Need to check Gazebo logs: `~/.gazebo/server-11345/default.log`

**If black screen or world load failure:**
→ **STOP**: URI resolution issue. Verify mesh files exist and are readable.

---

## Phase 2: Mesh Correction (PRIMARY FIX)

### Objective
Re-center the mesh geometry to align with the local origin, eliminating the 300+ meter offset.

### Prerequisites
- ✅ Blender installed (tested with 4.5.3 LTS, any recent version works)
- ✅ Backup created of original mesh

### Step 2.1: Backup Original Files

```bash
cd /home/finn/Documents/ardu_ws/meshes

# Create backup directory
mkdir -p backup_$(date +%Y%m%d_%H%M%S)

# Backup mesh and texture
cp model.dae backup_*/model_original.dae
cp DiffuseMap_0.jpg.001.jpg backup_*/texture_original.jpg

# Verify backups
ls -lh backup_*/
```

**Safety Check:** Confirm backups exist before proceeding.

### Step 2.2: Analyze Mesh in Blender

**Open mesh:**
```bash
blender /home/finn/Documents/ardu_ws/meshes/model.dae
```

**Inspection steps:**

1. **Check object location:**
   - Select mesh object in Outliner (top-right panel)
   - Look at Properties panel → Object Properties → Transform
   - Note the Location values (likely NOT 0, 0, 0)

2. **Check vertex coordinates:**
   - Switch to Edit Mode (Tab key)
   - Select all vertices (A key)
   - Look at 3D Viewport header → "N" panel → Item tab
   - Check "Median" coordinates (shows average vertex position)
   - **Expected**: Large positive/negative values (300+ range)

3. **Check mesh dimensions:**
   - Object Properties → Transform → Dimensions
   - Note the size (helps determine if scale is needed)

4. **Count cameras (photogrammetry artifacts):**
   - Expand scene in Outliner
   - Count camera objects (should be 200+)
   - These will be deleted

5. **Document findings:**
   - Object location: `________`
   - Median vertex position: `________`
   - Mesh dimensions: `________`
   - Number of cameras: `________`

### Step 2.3: Center Mesh Geometry

**CRITICAL OPERATION** - Follow exactly:

1. **Switch to Object Mode:**
   ```
   Press Tab (if in Edit Mode)
   ```

2. **Select mesh object:**
   ```
   Click mesh in Outliner or viewport
   Ensure it's highlighted (orange outline)
   ```

3. **Set origin to geometry center:**
   ```
   Menu: Object → Set Origin → Geometry to Origin
   ```
   - This moves the object's origin point to the geometric center
   - Object location still shows old values

4. **Move object to world origin:**
   ```
   Press Alt+G
   ```
   - Object location now shows (0, 0, 0) in Transform properties
   - Mesh is now centered at world origin

5. **Apply all transforms:**
   ```
   Menu: Object → Apply → All Transforms
   ```
   - Bakes the location/rotation/scale into vertex coordinates
   - Transform properties should now show:
     - Location: 0, 0, 0 (already set)
     - Rotation: 0, 0, 0
     - Scale: 1, 1, 1

6. **Verify centering:**
   - Switch to Edit Mode (Tab)
   - Select all (A)
   - Check "N" panel → Item → Median
   - **Expected**: Values should be close to (0, 0, 0), within ±5 meters

### Step 2.4: Clean Up Photogrammetry Artifacts

**Delete camera objects:**

1. **Switch to Object Mode** (Tab if needed)

2. **Select all cameras:**
   ```
   Click first camera in Outliner
   Hold Shift, click last camera
   ```
   OR
   ```
   Click in viewport
   Press A (select all)
   Hold Shift, click mesh to deselect it
   (Only cameras remain selected)
   ```

3. **Delete cameras:**
   ```
   Press X
   Select "Delete" from menu
   ```

4. **Verify deletion:**
   - Outliner should now show only:
     - Scene
     - Mesh object (your terrain)
     - Possibly lights/default camera
   - No "ER_02_21260_..." camera objects

**Expected result:** File size may decrease slightly, clean scene hierarchy.

### Step 2.5: Scale Assessment

**Determine if scaling is needed:**

1. **Check mesh dimensions:**
   - Object Properties → Dimensions
   - Note X, Y, Z values

2. **Decision criteria:**
   - If dimensions > 50 meters: Consider scaling down
   - If dimensions 1-50 meters: Probably good as-is
   - If dimensions < 1 meter: May be too small

3. **Optional scaling:**
   ```
   Press S (scale)
   Type scale factor (e.g., 0.1 for 10x smaller)
   Press Enter
   Menu: Object → Apply → Scale
   ```

**Recommendation:** Start without scaling. Can always adjust later in SDF if needed.

### Step 2.6: Export Corrected Mesh

**Export settings:**

1. **Menu: File → Export → Collada (Default) (.dae)**

2. **File browser:**
   - Navigate to: `/home/finn/Documents/ardu_ws/meshes/`
   - Filename: `model_centered.dae` (different name for safety)

3. **Export settings (left panel):**
   ```
   Main:
   - Selection Only: OFF (export all)
   - Apply Modifiers: ON
   - Use Object Instances: OFF

   Geometry:
   - Include UV Textures: ON
   - Include Material Textures: ON
   - Triangulate: OFF (keep as-is)

   Armature:
   - (Not relevant, ignore)

   Collada:
   - Use Blender Profile: ON
   - Keep Bind Info: OFF
   ```

4. **Click "Export COLLADA"**

5. **Verify export:**
   ```bash
   ls -lh /home/finn/Documents/ardu_ws/meshes/model_centered.dae
   # Should be similar size to original (60-75MB)
   ```

### Step 2.7: Validate Centered Mesh

**Re-import to verify:**

1. **Close Blender** (File → Quit)

2. **Reopen with new file:**
   ```bash
   blender /home/finn/Documents/ardu_ws/meshes/model_centered.dae
   ```

3. **Verify centering:**
   - Select mesh object
   - Object Properties → Transform → Location
   - **Expected**: (0, 0, 0) or very close
   - Switch to Edit Mode → Select All → "N" panel → Median
   - **Expected**: Near (0, 0, 0), within ±1 meter

4. **If centered correctly:**
   - Replace original mesh:
     ```bash
     cd /home/finn/Documents/ardu_ws/meshes
     mv model.dae model_original_uncorrected.dae
     mv model_centered.dae model.dae
     ```

5. **If NOT centered:**
   - Review Step 2.3, especially Alt+G and Apply Transforms
   - Try again or seek assistance

---

## Phase 3: Integration & Validation

### Objective
Update world configuration for centered mesh and verify visibility in Gazebo.

### Step 3.1: Update World File Configuration

**File to edit:** `/home/finn/Documents/ardu_ws/src/ardupilot_gz/ardupilot_gz_gazebo/worlds/iris_runway.sdf`

**Changes to make (lines 81-113):**

#### Option A: Remove Scale (Recommended if mesh is proper size)

```xml
<model name="custom_terrain_model">
  <static>true</static>
  <pose>5 0 0 0 0 0</pose>
  <link name="terrain_link">
    <visual name="mainMeshNode">
      <geometry>
        <mesh>
          <uri>file:///home/finn/Documents/ardu_ws/meshes/model.dae</uri>
          <!-- Scale removed - mesh now centered and properly sized -->
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
        </mesh>
      </geometry>
    </collision>
  </link>
</model>
```

**Changes made:**
- ❌ Removed: `<scale>0.01 0.01 0.01</scale>` from both visual and collision
- ✅ Mesh is now at 1:1 scale (vertices already at correct positions)

#### Option B: Simplified Material (If PBR texture causes issues)

**If terrain doesn't appear or looks wrong, try basic material:**

```xml
      <material>
        <ambient>0.8 0.8 0.8 1.0</ambient>
        <diffuse>0.8 0.8 0.8 1.0</diffuse>
        <specular>0.2 0.2 0.2 1.0</specular>
      </material>
```

**This removes:**
- PBR metal workflow (may have compatibility issues)
- 8K texture (may cause memory/performance issues)

**Can add texture back later once basic geometry is working.**

### Step 3.2: Rebuild Package

**CRITICAL:** World files are copied during build, not symlinked!

```bash
cd /home/finn/Documents/ardu_ws

# Rebuild world file package
colcon build --packages-select ardupilot_gz_gazebo --symlink-install

# Source updated environment
source install/setup.bash

# Verify installed file was updated
cat install/ardupilot_gz_gazebo/share/ardupilot_gz_gazebo/worlds/iris_runway.sdf | grep -A 10 "custom_terrain_model"
# Should NOT show <scale> tag anymore
```

**Validation:**
- ✅ Build completes without errors
- ✅ Installed world file matches source changes
- ✅ No `<scale>0.01 0.01 0.01</scale>` in installed file

### Step 3.3: Launch and Verify Visibility

**Launch simulation:**

```bash
ros2 launch ardupilot_gz_bringup iris_runway.launch.py
```

**Verification checklist:**

1. **✅ Gazebo opens without errors**
   - No black screen
   - GUI responsive

2. **✅ World loads successfully**
   - Terminal shows: `[Msg] World [map] initialized with [1ms] physics profile`
   - No URI resolution errors

3. **✅ Entity tree shows model**
   - Left panel → Expand "iris_runway"
   - "custom_terrain_model" present
   - Expand to see "terrain_link"

4. **✅ Geometry is VISIBLE**
   - Default camera view should show terrain OR
   - Right-click model → "Move To" → Geometry visible at (5, 0, 0)
   - No need to search far from pose location

5. **✅ Material renders correctly**
   - If using PBR: Texture should be applied
   - If using basic material: Gray/white surface
   - No pink "missing texture" indicators

6. **✅ Collision geometry working**
   - View → Wireframe or View → Transparent
   - Collision mesh should match visual mesh

### Step 3.4: Decision Point - Success or Debug

**If ALL verification steps pass:**
→ **SUCCESS!** Proceed to Step 3.5 (Optional cleanup)

**If geometry still not visible:**
→ **Debug Path A**: Camera/rendering issue
```bash
# Try different camera positions
# View → Camera → Reset Camera
# Or manually navigate around (5, 0, 0)
```

**If material looks wrong (pink, black, or missing texture):**
→ **Debug Path B**: Material/texture issue
- Switch to simplified material (Option B from Step 3.1)
- Rebuild and retest
- Check Gazebo logs: `~/.gazebo/server-*/default.log`

**If world fails to load:**
→ **Debug Path C**: SDF syntax error
- Check terminal for SDF parsing errors
- Verify XML syntax in world file
- Ensure mesh file path is correct and accessible

### Step 3.5: Optional Refinements

**A. Adjust terrain position:**

If terrain needs to be moved:
```xml
<pose>X Y Z roll pitch yaw</pose>
```
- X: Forward/backward (5 = 5 meters along runway)
- Y: Left/right (0 = centered)
- Z: Up/down (0 = ground level, may need adjustment)
- Rotations in radians (0 0 0 = no rotation)

**B. Adjust scale if needed:**

If mesh is too large/small:
```xml
<scale>0.5 0.5 0.5</scale>  <!-- 50% size -->
```
OR
```xml
<scale>2.0 2.0 2.0</scale>  <!-- 200% size -->
```

**C. Restore PBR texture (if using simplified material):**

Once basic geometry works, try adding texture back:
```xml
<material>
  <diffuse>1.0 1.0 1.0 1.0</diffuse>
  <specular>0.1 0.1 0.1 1.0</specular>
  <pbr>
    <metal>
      <albedo_map>file:///home/finn/Documents/ardu_ws/meshes/DiffuseMap_0.jpg.001.jpg</albedo_map>
    </metal>
  </pbr>
</material>
```

Rebuild and test. If it causes issues, stick with basic material.

**D. Performance optimization:**

8K texture may be overkill:
```bash
# Resize texture to 4K or 2K
convert /home/finn/Documents/ardu_ws/meshes/DiffuseMap_0.jpg.001.jpg \
        -resize 4096x4096 \
        /home/finn/Documents/ardu_ws/meshes/DiffuseMap_4K.jpg

# Update world file to use new texture
```

---

## Phase 4: Long-Term Improvement (OPTIONAL)

### Objective
Convert to proper Gazebo model package for portability and reusability.

**Benefits:**
- ✅ Follows Gazebo conventions
- ✅ Uses `package://` URIs (portable across systems)
- ✅ Can be included in multiple worlds
- ✅ Easier to share and version control

**Time estimate:** 20 minutes

### Step 4.1: Create Model Package Structure

```bash
cd /home/finn/Documents/ardu_ws/src/ardupilot_gazebo

# Create model directory
mkdir -p models/custom_terrain/meshes

# Move mesh files
cp /home/finn/Documents/ardu_ws/meshes/model.dae \
   models/custom_terrain/meshes/

cp /home/finn/Documents/ardu_ws/meshes/DiffuseMap_0.jpg.001.jpg \
   models/custom_terrain/meshes/texture.jpg
```

### Step 4.2: Create model.config

**File:** `models/custom_terrain/model.config`

```xml
<?xml version="1.0"?>
<model>
  <name>custom_terrain</name>
  <version>1.0</version>
  <sdf version="1.9">model.sdf</sdf>
  <author>
    <name>Finn</name>
    <email>your.email@example.com</email>
  </author>
  <description>
    Photogrammetry-reconstructed terrain model with PBR texture.
    Captured with [camera/method], processed with [software].
  </description>
</model>
```

### Step 4.3: Create model.sdf

**File:** `models/custom_terrain/model.sdf`

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
          </mesh>
        </geometry>
        <material>
          <diffuse>1.0 1.0 1.0 1.0</diffuse>
          <specular>0.1 0.1 0.1 1.0</specular>
          <pbr>
            <metal>
              <albedo_map>package://ardupilot_gazebo/models/custom_terrain/meshes/texture.jpg</albedo_map>
            </metal>
          </pbr>
        </material>
      </visual>
      <collision name="collision">
        <geometry>
          <mesh>
            <uri>package://ardupilot_gazebo/models/custom_terrain/meshes/model.dae</uri>
          </mesh>
        </geometry>
      </collision>
    </link>
  </model>
</sdf>
```

### Step 4.4: Update World File to Use Model Include

**File:** `worlds/iris_runway.sdf`

**Replace lines 81-113 with:**

```xml
<!-- Custom terrain model using proper package structure -->
<include>
  <uri>package://ardupilot_gazebo/models/custom_terrain</uri>
  <name>terrain</name>
  <pose>5 0 0 0 0 0</pose>
</include>
```

**Much simpler!** All model details in separate package.

### Step 4.5: Rebuild and Test

```bash
cd /home/finn/Documents/ardu_ws

# Rebuild to install new model
colcon build --packages-select ardupilot_gazebo ardupilot_gz_gazebo --symlink-install

source install/setup.bash

# Verify model installed
ls install/ardupilot_gazebo/share/ardupilot_gazebo/models/custom_terrain/

# Launch
ros2 launch ardupilot_gz_bringup iris_runway.launch.py
```

**Validation:**
- ✅ Model loads from package:// URI
- ✅ Terrain visible at (5, 0, 0)
- ✅ World file cleaner and more maintainable

---

## Troubleshooting Guide

### Issue: Mesh still offset after Blender centering

**Symptoms:**
- Performed Alt+G and Apply Transforms
- Re-exported mesh
- Geometry still not at model pose

**Diagnosis:**
```bash
# Re-open in Blender
blender /home/finn/Documents/ardu_ws/meshes/model.dae

# Check Edit Mode median
# Tab → A (select all) → N panel → Item → Median
```

**If median NOT near (0,0,0):**
- Transform operation didn't work correctly
- Try manual centering:
  1. Edit Mode → A (select all)
  2. Mesh → Transform → Center
  3. Object Mode → Apply All Transforms
  4. Re-export

### Issue: Gazebo black screen after changes

**Symptoms:**
- Gazebo opens but shows black/unresponsive screen
- Terminal shows URI resolution errors

**Cause:** Mesh file path incorrect or file missing

**Fix:**
```bash
# Verify mesh exists at exact path in SDF
ls -l /home/finn/Documents/ardu_ws/meshes/model.dae

# Check file permissions
chmod 644 /home/finn/Documents/ardu_ws/meshes/model.dae

# Verify no typos in world file URI
grep -n "model.dae" /home/finn/Documents/ardu_ws/src/ardupilot_gz/ardupilot_gz_gazebo/worlds/iris_runway.sdf
```

### Issue: Pink material (missing texture)

**Symptoms:**
- Geometry visible but bright pink color
- Texture not loading

**Cause:** Texture file path incorrect or Ogre2 compatibility

**Fix Option 1:** Check texture path
```bash
ls -l /home/finn/Documents/ardu_ws/meshes/DiffuseMap_0.jpg.001.jpg

# Verify URI in world file matches exactly
```

**Fix Option 2:** Use simplified material (no texture)
```xml
<material>
  <ambient>0.6 0.5 0.4 1.0</ambient>  <!-- Brownish terrain color -->
  <diffuse>0.6 0.5 0.4 1.0</diffuse>
  <specular>0.1 0.1 0.1 1.0</specular>
</material>
```

### Issue: Mesh too large or too small

**Symptoms:**
- Geometry visible but wrong size
- Too large: Fills entire viewport, can't see drone
- Too small: Hard to see, looks like a dot

**Fix:** Add scale factor back
```xml
<mesh>
  <uri>file:///home/finn/Documents/ardu_ws/meshes/model.dae</uri>
  <scale>X X X</scale>  <!-- Adjust value -->
</mesh>
```

**Scale guidelines:**
- Too large? Try: 0.5, 0.1, 0.01
- Too small? Try: 2.0, 5.0, 10.0

### Issue: Collision not working (drone passes through terrain)

**Symptoms:**
- Terrain visible
- Drone can fly through it without collision

**Cause:** Collision geometry not loaded or too complex

**Fix Option 1:** Verify collision tag present
```xml
<collision name="collision">
  <geometry>
    <mesh>
      <uri>file:///home/finn/Documents/ardu_ws/meshes/model.dae</uri>
    </mesh>
  </geometry>
</collision>
```

**Fix Option 2:** Use simplified collision (box approximation)
```xml
<collision name="collision">
  <geometry>
    <box>
      <size>10 10 2</size>  <!-- Adjust to terrain size -->
    </box>
  </geometry>
</collision>
```

**Note:** Complex mesh collision can impact performance. Box/cylinder approximations are often better.

---

## Expected Outcomes

### Successful Completion

**Visual confirmation:**
- ✅ Gazebo opens smoothly without black screen
- ✅ Terrain model visible in default camera view or at (5, 0, 0)
- ✅ Mesh geometry properly centered at model pose
- ✅ No large offsets requiring camera hunting
- ✅ Material renders correctly (textured or solid color)

**Technical confirmation:**
```bash
# Entity appears in scene tree
# Collision detection works
# No error messages in terminal
# Gazebo logs clean: ~/.gazebo/server-*/default.log
```

**Performance:**
- Simulation runs smoothly (30+ FPS in Gazebo GUI)
- Mesh loading time < 5 seconds
- No memory issues (even with 8K texture)

### Failure Modes & Escalation

**If Phase 1 verification shows geometry IS visible at offset:**
→ Mesh centering will fix it. Proceed confidently to Phase 2.

**If Phase 1 shows NO geometry anywhere:**
→ Different issue (material, mesh corruption, etc.). Debug before centering.

**If Phase 2 centering doesn't help:**
→ Check Blender export settings, try different format (OBJ, STL)

**If Phase 3 integration fails repeatedly:**
→ Consider starting with simple box placeholder, verify SDF configuration is correct, then substitute mesh.

---

## Alternative Approaches

### Approach B: Pose Compensation (Quick Hack)

**If Blender is not available or centering fails:**

Calculate compensating pose based on known offset:

```xml
<pose>1.5 1.8 2.25 0 0 0</pose>
```

**Calculation:**
- Original offset after scale: (3.5, -1.8, -2.25)
- Desired final position: (5, 0, 0)
- Compensating pose: (5 - 3.5, 0 - (-1.8), 0 - (-2.25)) = (1.5, 1.8, 2.25)

**Drawback:** Hardcoded offset, not portable, doesn't fix root cause.

### Approach C: Mesh Conversion to Different Format

**If COLLADA causes issues:**

Convert to OBJ or STL using Blender:

1. Import COLLADA
2. Center geometry (Phase 2 steps)
3. Export as: File → Export → Wavefront (.obj) or STL

Update SDF to use new format:
```xml
<uri>file:///home/finn/Documents/ardu_ws/meshes/model.obj</uri>
```

**Note:** OBJ supports textures, STL does not.

### Approach D: Use MeshLab for Automated Centering

**If Blender workflow is too complex:**

```bash
# Install MeshLab
sudo apt install meshlab

# Open mesh
meshlab /home/finn/Documents/ardu_ws/meshes/model.dae

# Center: Filters → Normals, Curvatures and Orientation → Transform: Center
# Apply: Filters → Mesh Layer → Flatten Visible Layers
# Export: File → Export Mesh As → model_centered.dae
```

Simpler interface, automated centering, good for this specific task.

---

## Risk Assessment

### Low Risk Operations
- ✅ Reading files and verification steps
- ✅ Launching simulation to inspect
- ✅ Creating backups
- ✅ Blender analysis (read-only)

### Medium Risk Operations
- ⚠️ Editing mesh in Blender (backups mitigate)
- ⚠️ Editing world file (source controlled, can revert)
- ⚠️ Rebuilding packages (reversible)

### High Risk Operations
- ⚠️ Overwriting original mesh without backup
- ⚠️ Deleting files without backup
- ⚠️ Modifying installed files directly (will be overwritten on rebuild)

### Risk Mitigation

**Backups before any modifications:**
```bash
# Mesh files
cp -r /home/finn/Documents/ardu_ws/meshes /home/finn/Documents/ardu_ws/meshes_backup_$(date +%Y%m%d)

# World file
cp /home/finn/Documents/ardu_ws/src/ardupilot_gz/ardupilot_gz_gazebo/worlds/iris_runway.sdf \
   /home/finn/Documents/ardu_ws/src/ardupilot_gz/ardupilot_gz_gazebo/worlds/iris_runway.sdf.backup
```

**Git version control:**
```bash
cd /home/finn/Documents/ardu_ws/src/ardupilot_gz
git status  # Check if tracked
git diff    # See changes
git checkout -- worlds/iris_runway.sdf  # Revert if needed
```

---

## Time Estimates

| Phase | Estimated Time | Actual Time |
|-------|----------------|-------------|
| **Phase 1: Verification** | 5 minutes | _____ |
| **Phase 2: Mesh Correction** | 30 minutes | _____ |
| **Phase 3: Integration** | 15 minutes | _____ |
| **Phase 4: Model Package** (optional) | 20 minutes | _____ |
| **Total (Phases 1-3)** | **50 minutes** | _____ |
| **Total (All phases)** | **70 minutes** | _____ |

**Assumptions:**
- Blender already installed and familiar
- No major unexpected issues
- Mesh centering works on first attempt

**Contingency:**
- Add 30 minutes for learning Blender if unfamiliar
- Add 15 minutes for troubleshooting if issues arise

---

## Success Criteria

### Must-Have (Blocking)
1. ✅ Terrain model visible in Gazebo at pose location (5, 0, 0)
2. ✅ No geometry offset beyond 0.5 meters from pose
3. ✅ Gazebo loads without errors
4. ✅ Mesh geometry centered in Blender (median < 1 meter from origin)

### Should-Have (Important)
1. ✅ Texture renders correctly (or acceptable solid color material)
2. ✅ Collision detection functional
3. ✅ Build process clean (no warnings)
4. ✅ Performance acceptable (30+ FPS)

### Nice-to-Have (Optional)
1. ✅ Converted to proper model package structure
2. ✅ Using package:// URIs
3. ✅ Clean scene hierarchy (no camera artifacts)
4. ✅ Optimized texture size for performance

---

## Post-Execution Validation

### After Completion, Verify:

**Functional tests:**
```bash
# 1. Clean build
cd /home/finn/Documents/ardu_ws
rm -rf build/ install/
colcon build
source install/setup.bash

# 2. Launch simulation
ros2 launch ardupilot_gz_bringup iris_runway.launch.py

# 3. Visual inspection
# - Terrain visible at (5, 0, 0)
# - Texture/material correct
# - No errors in terminal

# 4. Collision test (if ArduPilot integrated)
# - Fly drone toward terrain
# - Verify collision prevents pass-through
```

**Documentation:**
```bash
# Update Research.md with outcome
# Add screenshots to documentation
# Document any deviations from plan
```

---

## Conclusion

**Primary Fix:** Phase 2 (Blender mesh centering) addresses the root cause - photogrammetry offset.

**Critical Path:**
1. Verify issue (Phase 1)
2. Center mesh in Blender (Phase 2)
3. Update world config and rebuild (Phase 3)

**Expected Result:** Terrain model visible in Gazebo at intended location without offset.

**Confidence Level:** HIGH - Root cause clearly identified, solution well-documented in flow.md, similar fixes proven effective in CAD/simulation workflows.

**Fallback:** If centering fails, pose compensation (Approach B) provides temporary workaround while investigating alternative mesh processing methods.

---

## Notes for Execution

- Keep terminal log of all commands for documentation
- Take screenshots at key verification points
- Note any deviations from plan and reasons
- Document any additional issues discovered
- Update this plan with "Actual Time" and lessons learned

**Ready to execute when approved.**
