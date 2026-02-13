# STAR-CCM+ Automated CFD Workflow

Complete automation macros for Formula Student aerodynamics simulations using STAR-CCM+ 2510.

## Table of Contents
- [Overview](#overview)
- [Required Files](#required-files)
- [Workflow Macros](#workflow-macros)
- [Naming Conventions](#naming-conventions)
- [Usage](#usage)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)

---

## Overview

This automated workflow handles the complete CFD simulation pipeline from geometry import to post-processing, supporting three simulation types:
- **Type 1 (Symmetry)**: Half-car with symmetry plane, front wheels only
- **Type 2 (Yaw)**: Full car with yaw angle, all four wheels
- **Type 3 (Corner)**: Full car with rotating reference frame, all four wheels

### Workflow Steps
```
1. geometryimport.java  → Import CAD files & load parameters
2. transform.java       → Apply rotations & create coordinate systems
3. boundarysetup_N.java → Set boundary conditions (N = 1, 2, or 3)
4. mesh.java            → Generate mesh & save as XXX[s|y|c]@meshed.sim
5. solve.java           → Initialize & run simulation
6. postprocess.java     → Export animations & reports
```

---

## Required Files

### Input Files (in simulation directory)
```
case_parameter.dat      # Simulation parameters
body.x_t               # Car body geometry
unsprung.x_t           # Unsprung mass geometry
TH11_NNN.x_t           # Wings & diffuser (NNN = version number, uses highest)
```

### Macro Files
```
geometryimport.java    # Geometry import
transform.java         # Coordinate transformations
boundarysetup_1.java   # Boundary conditions (Type 1)
boundarysetup_2.java   # Boundary conditions (Type 2)
boundarysetup_3.java   # Boundary conditions (Type 3)
mesh.java              # Meshing
solve.java             # Solver
postprocess.java       # Post-processing
gui_run.java           # Complete workflow (GUI mode)
batch_run.java         # Complete workflow (batch mode with core control)
```

### Template File Requirements
Your template `.sim` file must contain:
- **3D-CAD Model 1** with groups: `car`, `unsprung`, `refinement_box`
- **Domains**: `domain_1`, `domain_2`, `domain_3` bodies
- **Scenes**: `type_1`, `type_2`, `type_3`
- **Mesh operation**: `Automated Mesh`
- **Physics continuum**: `Physics 1`
- **Reference frame**: `Rotating` (for Type 3)
- **Plots**: `Residuals`, `DF`
- **Sections**: `x plane`, `z plane`

---

## Workflow Macros

### 1. geometryimport.java
**Purpose**: Import CAD files and load simulation parameters

**Actions**:
1. Reads `case_parameter.dat` and applies all parameters to simulation
2. Imports `body.x_t` → moves to `car` group
3. Imports `unsprung.x_t` 
4. Finds and imports `TH11_NNN.x_t` (highest N value)
5. Moves bodies containing `FW`, `SW`, `RW`, `UT`, `INLET` to `car` group

**API Objects** (name-based):
```java
// CAD Model
CadModel cadModel = (CadModel) sim.get(SolidModelManager.class).getObject("3D-CAD Model 1");

// Body Groups
BodyGroup carGroup = cadModel.getBodyGroupManager().getBodyGroup("car");
BodyGroup bodyGroup = cadModel.getBodyGroupManager().getBodyGroup("body");

// Parameters
ScalarGlobalParameter param = (ScalarGlobalParameter) sim.get(GlobalParameterManager.class).getObject("parameter_name");
```

**Required Naming**:
- CAD Model: `"3D-CAD Model 1"`
- Body groups: `"car"`, `"unsprung"`, `"refinement_box"`
- Parameter file: `case_parameter.dat` in session directory

---

### 2. transform.java
**Purpose**: Apply pitch/roll/yaw rotations and create wheel coordinate systems

**Actions** (12 steps):
1. Pitch rotation on `car` group
2. Roll rotation on `car` group + `car_center_ref` body
3. Move `unsprung` group to `car` group
4. Yaw rotation on `car` group + `car_center_ref` + `refinement_box`
5. Update 3D-CAD
6. Create 4 wheel coordinate systems (steering_coordinate_1-4)
7. Create steering rotations for front wheels (1 & 3)
8. Create post-steering coordinates (wheel_coordinate_1 & 3)
9. Final update 3D-CAD
10. Create and export `car_coordinate`
11. Export 4 wheel coordinate systems
12. Domain boolean subtract based on `simulation_type`

**API Objects**:
```java
// Bodies
star.cadmodeler.Body carCenterRef = (star.cadmodeler.Body) cadModel.getBody("car_center_ref");
star.cadmodeler.Body wheelRef1 = (star.cadmodeler.Body) cadModel.getBody("wheel_ref_1");

// Vertices (for coordinate system creation)
Vertex vertexA = wheelRef1.getVertex("wheel_ref_1_a");
Vertex vertexB = wheelRef1.getVertex("wheel_ref_1_b");
Vertex vertexC = wheelRef1.getVertex("wheel_ref_1_c");

// Coordinate Systems
LabCoordinateSystem labCS = sim.getCoordinateSystemManager().getLabCoordinateSystem();
```

**Required Naming**:
- Bodies: `"car_center_ref"`, `"wheel_ref_1"` to `"wheel_ref_4"`
- Vertices: `"wheel_ref_N_a"`, `"wheel_ref_N_b"`, `"wheel_ref_N_c"` (N = 1-4)
- Face: `"car_center_ref_front"`
- Domains: `"domain_1"`, `"domain_2"`, `"domain_3"`
- Groups: `"car"`, `"unsprung"`, `"refinement_box"`
- Bodies to rotate (wheel 1): `"contact_patches 1"`, `"motor 1"`, `"tyre 1"`, `"tire_refinement_1"`, `"wheel_ref_1"`
- Bodies to rotate (wheel 3): `"contact_patches 3"`, `"motor 3"`, `"tyre 3"`, `"tire_refinement_3"`, `"wheel_ref_3"`

**Parameters Used**:
- `pitch_angle`, `pitch_center_y`, `pitch_center_z`
- `roll_angle`, `roll_center_y`
- `yaw_angle`, `yaw_center_z`
- `steering_angle`
- `simulation_type`

---

### 3. boundarysetup_N.java (N = 1, 2, or 3)
**Purpose**: Set boundary conditions based on simulation type

#### boundarysetup_1.java (Type 1: Symmetry)
**Boundaries**:
- `domain.inlet` → Velocity Inlet (`${free_stream_velocity}`)
- `domain.outlet` → Pressure Outlet
- `domain.symmetry` → Symmetry
- `domain.wall` → Slip Wall
- `domain.ground` → Moving Wall `[0, 0, -${free_stream_velocity}]`
- `tyre 1`, `tyre 2` → Rotating walls with `wheel_coordinate_1/2`

#### boundarysetup_2.java (Type 2: Yaw)
**Boundaries**:
- `domain.inlet` → Velocity Inlet
- `domain.outlet` → Pressure Outlet
- `domain.wall` → Slip Wall
- `domain.ground` → Moving Wall
- `tyre 1`, `tyre 2` → Rotating (RPM = `${wheel_rpm}`)
- `tyre 3`, `tyre 4` → Rotating (RPM = `-${wheel_rpm}`) ← **negative!**

#### boundarysetup_3.java (Type 3: Corner/Rotating)
**Special Features**:
- Sets region to `Rotating` reference frame
- `domain.inlet` → **Stagnation Inlet**
- `domain.ground` → Rotating wall (axis `[0,1,0]`, origin `[-${corner_radius}, 0, 0]`, RPM = `${domain_rpm}`)
- Creates `corner_tail_refinement` part
- Disables wake refinement on `car_surface`

**API Objects**:
```java
// Region
Region domainRegion = sim.getRegionManager().getRegion("domain");

// Boundaries
Boundary inletBoundary = domainRegion.getBoundaryManager().getBoundary("domain.inlet");
Boundary tyreBoundary = domainRegion.getBoundaryManager().getBoundary("tyre 1");

// Coordinate Systems
CartesianCoordinateSystem wheelCS = (CartesianCoordinateSystem) labCS
  .getLocalCoordinateSystemManager().getObject("wheel_coordinate_1");

// Physics
PhysicsContinuum physics = (PhysicsContinuum) sim.getContinuumManager().getContinuum("Physics 1");

// Reference Frame (Type 3 only)
UserRotatingReferenceFrame rotatingFrame = (UserRotatingReferenceFrame) sim
  .get(ReferenceFrameManager.class).getObject("Rotating");

// Mesh Control (Type 3 only)
AutoMeshOperation autoMesh = (AutoMeshOperation) sim.get(MeshOperationManager.class)
  .getObject("Automated Mesh");
SurfaceCustomMeshControl carSurfaceControl = (SurfaceCustomMeshControl) autoMesh
  .getCustomMeshControls().getObject("car_surface");
```

**Required Naming**:
- Region: `"domain"`
- Boundaries: `"domain.inlet"`, `"domain.outlet"`, `"domain.symmetry"`, `"domain.wall"`, `"domain.ground"`, `"tyre 1"` to `"tyre 4"`
- Coordinate systems: `"wheel_coordinate_1"` to `"wheel_coordinate_4"`
- Physics: `"Physics 1"`
- Reference frame: `"Rotating"` (Type 3)
- Mesh operation: `"Automated Mesh"` (Type 3)
- Mesh control: `"car_surface"` (Type 3)
- Body: `"corner_tail_refinement"` (Type 3)

**Parameters Used**:
- `free_stream_velocity`
- `wheel_rpm`
- `corner_radius` (Type 3)
- `domain_rpm` (Type 3)

---

### 4. mesh.java
**Purpose**: Execute meshing and save with type-specific suffix

**Actions**:
1. Saves current state
2. Executes `Automated Mesh` operation
3. Saves as `XXX[s|y|c]@meshed.sim` based on `simulation_type`
   - Type 1 → `XXXs@meshed.sim`
   - Type 2 → `XXXy@meshed.sim`
   - Type 3 → `XXXc@meshed.sim`

**API Objects**:
```java
AutoMeshOperation autoMesh = (AutoMeshOperation) sim.get(MeshOperationManager.class)
  .getObject("Automated Mesh");
```

**Required Naming**:
- Mesh operation: `"Automated Mesh"`

---

### 5. solve.java
**Purpose**: Initialize solution, run simulation, export plots, and save

**Actions**:
1. Initializes solution
2. Enables auto-save for `Residuals` and `DF` plots (overwrites same file)
3. Runs simulation
4. Exports final `Residuals_final.csv` and `DF_final.csv`
5. **Saves simulation state** (important for non-standard stop iterations)

**API Objects**:
```java
// Solution
Solution solution = sim.getSolution();

// Plots
ResidualPlot residualPlot = (ResidualPlot) sim.getPlotManager().getPlot("Residuals");
Cartesian2DPlot dfPlot = (Cartesian2DPlot) sim.getPlotManager().getPlot("DF");

// Iterator
sim.getSimulationIterator().run();
```

**Required Naming**:
- Plots: `"Residuals"`, `"DF"`

---

### 6. postprocess.java
**Purpose**: Export animations and report data

**Actions**:
1. Sets coordinate system for `x plane` and `z plane` to `car_coordinate`
2. Selects scene based on `simulation_type` (`type_1`, `type_2`, or `type_3`)
3. Sets X-direction view and exports animations:
   - `x direction static p`
   - `x direction velocity`
   - Duration: 1.6s (Type 1) or 3.2s (Type 2/3)
4. Sets Z-direction view and exports animations:
   - `z direction static p`
   - `z direction total p`
   - `z direction velocity`
   - `z direction vorticity`
   - Duration: 6.6s (all types)
5. Exports reports to `Result.dat`

**Animation Settings**:
- Resolution: 1600×900 (compromise between quality and speed)
- Frame rate: 25 fps
- Format: PNG sequence

**API Objects**:
```java
// Sections
PlaneSection xPlane = (PlaneSection) sim.getPartManager().getObject("x plane");
PlaneSection zPlane = (PlaneSection) sim.getPartManager().getObject("z plane");

// Scenes
Scene scene = sim.getSceneManager().getScene("type_1"); // or "type_2", "type_3"

// Displayers
Object displayerObj = scene.getDisplayerManager().getObject("x direction static p");
Displayer displayer = (Displayer) displayerObj;

// Animation
AnimationDirector animDirector = scene.getAnimationDirector();

// Reports (using reflection)
Object reportObj = sim.getReportManager().getReport("mean_Cl/Cd");
java.lang.reflect.Method method = reportObj.getClass().getMethod("getReportMonitorValue");
double value = (Double) method.invoke(reportObj);
```

**Required Naming**:
- Sections: `"x plane"`, `"z plane"`
- Scenes: `"type_1"`, `"type_2"`, `"type_3"`
- Displayers: 
  - `"x direction static p"`
  - `"x direction velocity"`
  - `"z direction static p"`
  - `"z direction total p"`
  - `"z direction velocity"`
  - `"z direction vorticity"`
- Reports:
  - `"mean_Cl/Cd"`
  - `"mean_Cla"`
  - `"mean_FW_DF"`
  - `"mean_RW_DF"`
  - `"mean_SW_DF"`
  - `"mean_UT_DF"`
  - `"mean_W_DF"`
  - `"Aero_balance_x"`
  - `"Aero_balance_z"`
- Coordinate system: `"car_coordinate"`

**Parameters Used** (for camera views):
- `x_focal_x`, `x_focal_y`, `x_focal_z`
- `x_position_x`, `x_position_y`, `x_position_z`
- `z_focal_x`, `z_focal_y`, `z_focal_z`
- `z_position_x`, `z_position_y`, `z_position_z`

---

### 7. gui_run.java
**Purpose**: Execute complete workflow in GUI mode (single core count)

**Execution Order**:
1. geometryimport.java
2. transform.java
3. boundarysetup_N.java (N determined by `simulation_type`)
4. mesh.java
5. solve.java
6. postprocess.java

**Usage**:
```bash
# In STAR-CCM+ GUI: Tools → Macros → Execute...
# Select: gui_run.java
```

---

### 8. batch_run.java
**Purpose**: Execute workflow with different core counts for mesh and solve phases

**Configuration** (edit at top of file):
```java
private static final int MESH_CORE = 10;      // Cores for meshing, but the real meshing core is determined in batch mode command
private static final int SOLVE_CORE = 20;     // Cores for solving
private static final int GPGPU = 0;           // Number of GPUs (0 = no GPU) if one want to use gpgpu, SOLVE_CORE is recommand to be the same number of GPUs 
private static final int POSTPROCESS = 1;     // 1 = run postprocess, 0 = skip
```

**Phase 1** (Mesh Phase):
- Runs with `MESH_CORE` cores
- Executes: geometryimport → transform → boundarysetup_N → mesh
- Saves and generates Phase 2 scripts

**Phase 2** (Solve Phase):
- Runs with `SOLVE_CORE` cores (and GPUs if specified)
- Executes: solve (and postprocess if enabled)
- Use generated `run_phase2.sh` (Linux) or `run_phase2.bat` (Windows)

**Usage**:
```bash
# Phase 1: Start with MESH_CORE
starccm+ -np 10 -batch batch_run.java simulation.sim # if MESH_CORE=10, 

# Phase 2: After Phase 1 completes, run generated script
./run_phase2.sh  # Linux
run_phase2.bat   # Windows
```

---

## Naming Conventions

### Critical Object Names (DO NOT CHANGE without updating macros)

#### 3D-CAD Model & Groups
- CAD Model: `"3D-CAD Model 1"`
- Groups: `"car"`, `"unsprung"`, `"refinement_box"`

#### Bodies
- Reference bodies: `"car_center_ref"`, `"wheel_ref_1"` to `"wheel_ref_4"`
- Vertices: `"wheel_ref_N_a"`, `"wheel_ref_N_b"`, `"wheel_ref_N_c"` (N=1-4)
- Face: `"car_center_ref_front"`
- Domains: `"domain_1"`, `"domain_2"`, `"domain_3"`
- Refinement (Type 3): `"corner_tail_refinement"`

#### Wheel Bodies (for steering rotation)
- Wheel 1: `"contact_patches 1"`, `"motor 1"`, `"tyre 1"`, `"tire_refinement_1"`, `"wheel_ref_1"`
- Wheel 3: `"contact_patches 3"`, `"motor 3"`, `"tyre 3"`, `"tire_refinement_3"`, `"wheel_ref_3"`

#### Regions & Boundaries
- Region: `"domain"`
- Boundaries: `"domain.inlet"`, `"domain.outlet"`, `"domain.symmetry"`, `"domain.wall"`, `"domain.ground"`
- Tyres: `"tyre 1"`, `"tyre 2"`, `"tyre 3"`, `"tyre 4"`

#### Coordinate Systems
- Exported: `"car_coordinate"`, `"wheel_coordinate_1"` to `"wheel_coordinate_4"`
- Steering (internal): `"steering_coordinate_1"`, `"steering_coordinate_3"`

#### Scenes & Displayers
- Scenes: `"type_1"`, `"type_2"`, `"type_3"`
- Sections: `"x plane"`, `"z plane"`
- Displayers (6 total):
  - `"x direction static p"`, `"x direction velocity"`
  - `"z direction static p"`, `"z direction total p"`, `"z direction velocity"`, `"z direction vorticity"`

#### Physics & Solver
- Physics: `"Physics 1"`
- Reference frame: `"Rotating"`
- Mesh operation: `"Automated Mesh"`
- Mesh control: `"car_surface"`

#### Plots & Reports
- Plots: `"Residuals"`, `"DF"`
- Reports (9 total):
  - `"mean_Cl/Cd"`, `"mean_Cla"`
  - `"mean_FW_DF"`, `"mean_RW_DF"`, `"mean_SW_DF"`, `"mean_UT_DF"`, `"mean_W_DF"`
  - `"Aero_balance_x"`, `"Aero_balance_z"`

---

## Configuration

### case_parameter.dat Format

 - simulation_type = 1 
 - corner_radius = 10
 - free_stream_velocity = 12.5
 - pitch_angle = 0
 - roll_angle = 1.46
 - yaw_angle = 10.64
 - ride_height = 0.035
 - steering_angle = 5
 - stopping_iteration = 3000

other configulation parameter like suspension geometry, scene position, refinement box can be find in template file

### Required Parameters
**All Types**:
- `simulation_type` (1, 2, or 3)
- `pitch_angle`, `pitch_center_y`, `pitch_center_z`
- `roll_angle`, `roll_center_y`
- `yaw_angle`, `yaw_center_z`
- `steering_angle`
- `free_stream_velocity`
- `wheel_rpm`
- Camera positions (x and z focal/position coordinates)

**Type 3 Only**:
- `corner_radius`
- `domain_rpm`

---

## Usage

### Method 1: GUI Mode (Simple)
```bash
# 1. Open STAR-CCM+ with your template file
starccm+ template.sim

# 2. Place all required files in the same directory:
#    - case_parameter.dat
#    - body.x_t
#    - unsprung.x_t
#    - TH11_NNN.x_t
#    - All .java macro files

# 3. Save the simulation file

# 4. Run gui_run.java from GUI:
#    Tools → Macros → Execute... → gui_run.java
```

### Method 2: Batch Mode (Advanced)
```bash
# 1. Configure batch_run.java parameters:
#    MESH_CORE, SOLVE_CORE, GPGPU, POSTPROCESS

# 2. Run Phase 1 (Mesh):
starccm+ -np 10 -batch batch_run.java template.sim

# 3. Phase 1 will generate:
#    - XXX[s|y|c]@meshed.sim
#    - run_phase2.sh (Linux)
#    - run_phase2.bat (Windows)

# 4. Run Phase 2 (Solve + Postprocess):
./run_phase2.sh          # Linux
run_phase2.bat           # Windows
```

### Method 3: Individual Macros (Debug/Development)
```bash
# Run macros one by one
starccm+ -batch geometryimport.java simulation.sim
starccm+ -batch transform.java simulation.sim
starccm+ -batch boundarysetup_2.java simulation.sim
starccm+ -batch mesh.java simulation.sim
starccm+ -np 20 -gpu 1 -batch solve.java simulation_meshed.sim
starccm+ -batch postprocess.java simulation_meshed.sim
```

---

## Troubleshooting

### Common Issues

**1. "Parameter 'XXX' not found"**
- Check that all required parameters exist in your template file
- Verify parameter names match exactly (case-sensitive)

**2. "Body/Boundary 'XXX' not found"**
- Check naming conventions section
- Use `view` tool in 3D-CAD to verify exact names
- Names are case-sensitive!

**3. "Macro file not found"**
- Ensure all `.java` files are in the session directory
- Check file permissions

**4. Wheel rotation direction wrong**
- Wheel 1, 2: Should use `${wheel_rpm}`
- Wheel 3, 4: Should use `-${wheel_rpm}` (negative!)
- Check wheel coordinate system orientation

**5. Animation export too slow**
- Reduce resolution in postprocess.java (line with `animDirector.record`)
- Reduce duration
- Disable Line Integral Convolution smooth shading

**6. Phase 2 script not executing**
- Linux: `chmod +x run_phase2.sh`
- Check that meshed file path is correct

---

## Output Files

### After Complete Workflow
```
simulation_directory/
├── case_parameter.dat
├── XXX[s|y|c]@meshed.sim        # Meshed simulation
├── Residuals.csv                 # Auto-updated during solve
├── Residuals_final.csv           # Final residuals
├── DF.csv                        # Auto-updated during solve
├── DF_final.csv                  # Final downforce data
├── Result.dat                    # Report summary
├── x direction static p/         # Animation frames
│   ├── frame_0001.png
│   ├── frame_0002.png
│   └── ...
├── x direction velocity/
├── z direction static p/
├── z direction total p/
├── z direction velocity/
├── z direction vorticity/
└── run_phase2.sh                 # Generated Phase 2 script (batch mode)
```

---

## Quick Reference: API Pattern

All macros use **name-based lookup** instead of index-based:

```java
// ✅ CORRECT: Name-based (robust)
Body wheelRef = cadModel.getBody("wheel_ref_1");
Boundary inlet = region.getBoundaryManager().getBoundary("domain.inlet");
Scene scene = sim.getSceneManager().getScene("type_1");

// ❌ WRONG: Index-based (fragile)
Body wheelRef = cadModel.getBodies().get(5);
Boundary inlet = region.getBoundaryManager().getBoundaries().get(0);
Scene scene = sim.getSceneManager().getScenes().get(2);
```

**Why?** Name-based lookup ensures macros work even if object order changes in the template file.

---

## For Next Year's Team

### If You Need to Modify Object Names:

1. **Search all .java files** for the old name
2. **Update ALL occurrences** consistently
3. **Test with a simple case** before running full simulations

### Example: Renaming "car_center_ref" to "car_ref_center"
```bash
# Search for all occurrences
grep -r "car_center_ref" *.java

# Update in all relevant macros:
# - transform.java (multiple locations)
# - boundarysetup_*.java (if used)
```

### Adding New Simulation Type (Type 4):

1. Create `boundarysetup_4.java` based on Type 2 or 3
2. Add case to `getMeshedFileName()` in mesh.java and batch_run.java
3. Add case to `getSimulationType()` validation (change max from 3 to 4)
4. Create `type_4` scene in template
5. Update postprocess.java scene selection

---

## Support & Documentation

- **STAR-CCM+ User Guide**: Help → User Guide (within STAR-CCM+)
- **Java API Documentation**: Help → Java API (within STAR-CCM+)
- **Macro Recording**: Tools → Macros → Start Recording (to learn API usage)

---

## Version History

- **v1.0** (2026-02): Initial release
  - Complete automated workflow
  - Support for 3 simulation types
  - Batch mode with core/GPU control
  - Post-processing with animations

---

*Last Updated: February 2026*
*Compatible with: STAR-CCM+ 2510*
