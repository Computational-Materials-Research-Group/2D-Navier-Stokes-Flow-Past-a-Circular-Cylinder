# 2D Navier-Stokes Flow Past a Circular Cylinder

<p align="center">
  <img src="https://img.shields.io/badge/FreeFEM++-Simulation-blue?style=for-the-badge&logo=gnu&logoColor=white"/>
  <img src="https://img.shields.io/badge/Re%3D500-Unsteady%20Flow-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/2D-Circular%20Cylinder-red?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Characteristics-Method%20(convect)-purple?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/ParaView-VTK%20Export-green?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/License-MIT-lightgrey?style=for-the-badge"/>
</p>

<p align="center">
  A 2D finite element simulation of <b>unsteady incompressible Navier-Stokes flow around
  a circular cylinder</b> at Re=500 using FreeFEM++.
  The simulation uses the <b>method of characteristics (convect)</b> for semi-Lagrangian
  advection and <b>adaptive mesh refinement (AMR)</b> to automatically resolve the
  wake, boundary layer, and shed vortices.
  All fields are exported as <i>VTU/PVD files</i> for unified ParaView animation
  across 200 frames spanning 40 seconds of physical time.
</p>

<img width="1008" height="772" alt="vorticity" src="https://github.com/user-attachments/assets/822f200b-37df-4d0a-8d85-b12c2db7af9a" />

---

## Physics

The simulation solves the unsteady 2D incompressible Navier-Stokes equations,
capturing the full development of the Kármán vortex street. Key features include:

- Unsteady incompressible Navier-Stokes solved with a splitting/characteristics approach
- Semi-Lagrangian convection via FreeFEM++ `convect()` — unconditionally stable advection
- Taylor-Hood P2/P1 element pair — inf-sup stable, no spurious pressure modes
- Adaptive mesh refinement (AMR) every 20 steps driven by vorticity and velocity fields
- Fine cells appear automatically in the wake and shear layers; coarse cells fill the far field
- UMFPACK direct sparse solver — robust for 2D, no convergence failures
- VTU/PVD export for ParaView animation with 8 output fields per frame

---

## Geometry

```
      label=3 (top wall, Ux=Uinf)
  ┌─────────────────────────────────────────────┐
  │                                             │
  │              ○  label=4                     │
label=1         (cylinder, no-slip)           label=2
(inlet                                        (outlet,
Ux=Uinf)                                      free)
  │                                             │
  └─────────────────────────────────────────────┘
      label=3 (bottom wall, Ux=Uinf)

  Cylinder at origin (0,0)
  Domain: x ∈ [−5, 20] m    (5 diameters upstream, 20 downstream)
          y ∈ [−8,  8] m    (half-height Ly = 8 m)
  Cylinder radius Rcyl = 0.5 m  (diameter D = 1 m)
```

- Boundary labels:
  - Label 1: Inlet (left wall) — Dirichlet Ux = 1.0, Uy = 0
  - Label 2: Outlet (right wall) — Neumann (natural), free outflow
  - Label 3: Top + Bottom walls — Dirichlet Ux = 1.0, Uy = 0
  - Label 4: Cylinder surface — no-slip Ux = 0, Uy = 0

---

## Fluid Properties

| Property | Symbol | Value | Unit |
|---|---|---|---|
| Density | Rho | 1.0 | kg/m³ |
| Dynamic viscosity | Mu | 0.002 | Pa·s |
| Free-stream velocity | Uinf | 1.0 | m/s |
| Dynamic pressure | dynPres | 0.5 | Pa |
| Reynolds number | Re | 500 | — |

---

## Process Parameters

| Parameter | Symbol | Value | Description |
|---|---|---|---|
| Time step | dt | 0.02 s | Fixed timestep |
| Total steps | Nsteps | 2000 | Physical time = 40 s |
| Save frequency | saveFreq | 10 steps | VTU written every 10 steps |
| AMR frequency | adaptFreq | 20 steps | Mesh adapted every 20 steps |
| AMR error target | errAMR | 0.02 | Interpolation error threshold |
| Min element size | hminAMR | 0.01 m | Smallest allowed cell |
| Max element size | hmaxAMR | 2.0 m | Largest allowed cell |
| Max vertices | nbvxMax | 80,000 | Vertex count cap |

---

## Mesh

| Parameter | Value | Description |
|---|---|---|
| Bottom/Top | 60 points | Horizontal boundary resolution |
| Left/Right | 20 points | Vertical boundary resolution |
| Cylinder | 60 points (negative) | Clockwise orientation = hole in domain |
| Initial vertices | ~3,000 | Before first AMR pass |
| Max vertices | 80,000 | Hard cap (nbvxMax) |
| Method | buildmesh | Unstructured triangular |

### AMR Settings

| Setting | Value | Effect |
|---|---|---|
| errAMR | 0.02 | Target interpolation error |
| hminAMR | 0.01 m | Minimum element size |
| hmaxAMR | 2.0 m | Maximum element size |
| nbvxMax | 80,000 | Maximum vertex count |
| Refine on | omega, ux, uy | Vorticity + velocity driven |

Fine cells are automatically placed in the wake and boundary layer.
Coarse cells fill the far field, keeping computational cost low.

---

## Finite Element Spaces

| Space | Element | Fields |
|---|---|---|
| Uh | P2 × P2 | Velocity (ux, uy) and test functions (vx, vy) |
| Ph | P1 | Pressure p, test function q |
| Sh | P1 | All derived scalar fields |

P2/P1 Taylor-Hood element pair — inf-sup stable, no spurious pressure modes.
No pressure stabilisation needed.

---

## Governing Equations

### Navier-Stokes — Characteristics Splitting

```
Advection step (semi-Lagrangian):
  u* = convect([ux, uy], -dt, ux)     (foot-of-characteristic)
  v* = convect([ux, uy], -dt, uy)

Diffusion + pressure correction step (Stokes solve):
  Rho/dt * (u - u*) - Mu * Lap(u) + grad(p) = 0    in Omega
  div(u) = 0                                         in Omega
```

### Boundary Conditions

```
Inlet   (label 1): ux = Uinf = 1.0,  uy = 0    (Dirichlet)
Outlet  (label 2): natural Neumann   (no BC)    (free outflow)
Walls   (label 3): ux = Uinf = 1.0,  uy = 0    (Dirichlet)
Cylinder(label 4): ux = 0,           uy = 0     (no-slip Dirichlet)
```

### Derived Fields

```
Vorticity:    omega = dUy/dx - dUx/dy           [1/s]
Vel. magnitude: VelMag = sqrt(Ux^2 + Uy^2)     [m/s]
Pressure coeff: Cp = p / dynPres                [—]
```

---

## Numerical Method

| Aspect | Choice |
|--------|--------|
| Spatial discretisation | Finite Element Method (FEM) |
| Velocity element | P2 (quadratic, nodal) |
| Pressure element | P1 (linear, nodal) |
| Advection | Semi-Lagrangian characteristics — convect() |
| Time integration | Fixed-step splitting scheme |
| Linear solver | UMFPACK (direct sparse) |
| Mesh adaptation | adaptmesh() — error-driven isotropic refinement |
| Stability | Unconditional (characteristics); no CFL constraint |

---

## Output Fields (ParaView)

Each `.vtu` frame contains the following fields:

| Field | Type | Description | Units |
|---|---|---|---|
| Ux | P1 nodal | X-velocity component | m/s |
| Uy | P1 nodal | Y-velocity component | m/s |
| VelMag | P1 nodal | Velocity magnitude √(Ux²+Uy²) | m/s |
| Pressure | P1 nodal | Static pressure | Pa |
| Cp | P1 nodal | Pressure coefficient p / (0.5 ρ U²) | — |
| Vorticity | P1 nodal | Signed vorticity ∂Uy/∂x − ∂Ux/∂y | 1/s |
| VorticityMag | P1 nodal | Absolute vorticity | 1/s |
| CellSize | P0 element | Local mesh element size (AMR diagnostic) | m |

---

## Repository Structure

```
cylinder2d_flow.edp                    # Main FreeFEM++ simulation script
README.md                              # This file

C:\Users\pedit\Downloads\cylinder2d\
├── cylinder2d.pvd                     # Master animation (open this in ParaView)
├── frame0000.vtu                      # t = 0.2 s   (step 10)
├── frame0001.vtu                      # t = 0.4 s   (step 20)
├── ...
└── frame0199.vtu                      # t = 40.0 s  (step 2000 — 200 frames total)
```

---

## How to Run

### Requirements

- FreeFEM++ v4.10 or later: https://freefem.org
- ParaView v5.x or later: https://www.paraview.org

### Step 1 — Run the simulation

```bash
FreeFem++ cylinder2d_flow.edp
```

The script will:
1. Build the initial unstructured triangular mesh (~3,000 vertices)
2. For each of 2,000 timesteps: advect with characteristics, solve Stokes system
3. Apply AMR every 20 steps, driven by vorticity and velocity error
4. Compute all post-processing fields (Cp, vorticity, VelMag, CellSize)
5. Save one VTU per 10 steps and append to master PVD file
6. Print per-step progress to console

Console output example:
```
Step 150/2000  t=3.0  nv=12453  ux_max=1.23  ux_min=-0.31  |om|_max=4.51
  -> AMR: nv=14821  nt=28904
Step 160/2000  t=3.2  nv=14821  ux_max=1.24  ux_min=-0.33  |om|_max=4.63
...
Step 600/2000  t=12.0 nv=21350  ux_max=1.41  ux_min=-0.52  |om|_max=7.82
  -> AMR: nv=23108  nt=45220
```

### Step 2 — Open in ParaView

1. `File > Open` → navigate to `C:\Users\pedit\Downloads\cylinder2d\`
2. Change Files of type to `All Files (*.*)`
3. Select `cylinder2d.pvd` → OK
4. Choose PVD Reader when prompted → OK
5. Click `Apply`
6. Set colour field to `VelMag`
7. Click `Rescale to Data Range Over All Timesteps`

### Step 3 — Visualise the flow fields

**Option A — Vortex street (recommended first view)**
```
Color by Vorticity
Colormap: Blue-White-Red (diverging), range −5 to +5
Press Play
-> Alternating positive (red) and negative (blue) vortices shed from cylinder
-> Street fully developed and periodic after t ≈ 25 s
```

**Option B — Velocity wake deficit**
```
Color by Ux (Cool-Warm colormap)
-> Low-velocity wake region behind cylinder (blue)
-> Accelerated flow around cylinder flanks (red)
-> Recirculation zone and periodic oscillation clearly visible
```

**Option C — AMR mesh evolution**
```
Representation: Wireframe
-> Watch fine cells track the vortex street as it develops
Color by CellSize (Rainbow)
-> Fine cells (small h) concentrate in wake; coarse cells in far field
```

**Option D — Pressure and drag**
```
Color by Cp (range −2.0 to 1.0)
-> Stagnation point at front of cylinder (Cp ≈ +1)
-> Suction on flanks (Cp < 0)
-> Asymmetric low-pressure wake oscillates with shedding frequency
```

**Option E — Velocity magnitude**
```
Color by VelMag (range 0.0 to 1.3)
-> Speed-up around cylinder: VelMag > Uinf on flanks
-> Wake deficit: VelMag < Uinf downstream
```

Press `Play` to watch all 200 frames of vortex development and periodic shedding.

---

## What to Look for in Results

### Symmetry Breaking and Onset of Shedding
At early times (t < 5 s) the wake is symmetric and attached. A small numerical
perturbation grows via the Kelvin-Helmholtz instability of the separated shear layers.
By t ≈ 10–15 s the symmetry is broken and alternating vortex shedding begins. Watch
the `Vorticity` field transition from a steady symmetric pattern to an oscillating one.

### Kármán Vortex Street
Once established (t > 25 s), the shed vortices form a staggered double row downstream —
the classical Kármán vortex street. Positive (counterclockwise) vortices shed from the
top and negative (clockwise) vortices from the bottom in strict alternation. The street
convects downstream at roughly 80% of the free-stream speed.

### Strouhal Number
The shedding frequency f can be read from the time history of lift on the cylinder.
At Re=500, the expected Strouhal number is St = f·D/U ≈ 0.20. To measure it in
ParaView: use `Filters > Plot Selection Over Time` on a point in the near wake and
read the dominant frequency from the oscillation in Uy.

### AMR Refinement Tracking
The `CellSize` and Wireframe views show the mesh adapting as the flow evolves.
Fine cells (h ≈ hminAMR = 0.01 m) are placed automatically in the shear layers and
wake. The vertex count grows from ~3,000 at t = 0 to ~20,000–30,000 once the
street is fully developed, then stabilises as AMR reaches equilibrium.

### Pressure Coefficient Distribution
The `Cp` field on and around the cylinder shows the classic bluff-body pressure
pattern: stagnation (Cp ≈ +1) at the front, strong suction (Cp ≈ −1.5) at the
flanks, and an unsteady separated base region (Cp ≈ −0.5 to −1.0) at the rear.
The oscillating asymmetry in Cp drives the periodic lift force.

---

## Expected Physics

```
t =  0– 5 s  : Symmetric attached wake — no shedding, steady recirculation bubble
t =  5–15 s  : Instability grows — asymmetry develops in shear layers
t = 15–25 s  : Alternating vortex shedding begins (Kármán street forming)
t = 25–40 s  : Fully developed, statistically periodic shedding

Drag coefficient Cd ≈ 1.0–1.2   (literature at Re=500: ~1.0–1.5)
Lift coefficient Cl ≈ ±0.3–0.5  (once shedding is periodic)
Strouhal number  St ≈ 0.20       (f·D/U, shedding frequency)
```

---

## Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| No vortex shedding observed | dt too large or Re too low | Lower dt to 0.01 or raise Re to 1000 |
| Mesh grows excessively large | nbvxMax set too high | Lower nbvxMax to 40,000 |
| Very slow per-step solve | Too many vertices after AMR | Raise errAMR to 0.04 |
| Checkerboard pressure field | Wrong FE pair used | Do not change P2/P1 to P1/P1 |
| VTU files not written | Incorrect output path | Check backslash escaping in outdir string |
| Simulation diverges | dt too large at high Re | Reduce dt to 0.005 |
| Symmetric wake never breaks | No perturbation seeded | Add small Uy = epsilon IC or reduce dt |
| AMR removes wake resolution | errAMR too coarse | Lower errAMR to 0.01 |

---

## Extending the Model

| Extension | What to change |
|---|---|
| Higher Reynolds number | Increase Re (lower Mu); reduce dt to maintain stability |
| 3D simulation | Port to 3D with buildlayers; add spanwise direction |
| Elliptical cylinder | Replace circular border with parametric ellipse |
| Forced oscillation | Add time-varying Dirichlet BC on cylinder label 4 |
| Finer cylinder resolution | Increase cylinder border points from 60 to 120+ |
| Longer domain | Increase Lx2 beyond 20 m to capture far-wake decay |
| Inflow perturbation | Add sinusoidal Uy component at inlet to seed shedding earlier |
| Drag/lift monitoring | Integrate pressure and viscous stress on label 4 at each step |
| Finer time resolution | Reduce dt to 0.005 s; increase Nsteps to 8000 |
| Turbulence (RANS) | Add k-epsilon or Smagorinsky subgrid model terms to variational form |

---

## Citation

If you use this simulation, please cite:

```bibtex
@software{mishra_2026_cylinder2d,
  author    = {Mishra, A.},
  title     = {2D Navier-Stokes Flow Past a Circular Cylinder},
  year      = {2026},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.20380080},
  url       = {https://doi.org/10.5281/zenodo.20380080}
}
```

Plain text citation:

> Mishra, A. (2026). *2D Navier-Stokes Flow Past a Circular Cylinder*. Zenodo. https://doi.org/10.5281/zenodo.20380080

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.20380080.svg)](https://doi.org/10.5281/zenodo.20380080)

---

## Author

**akshansh11**
GitHub: https://github.com/akshansh11

---

## License

<p>
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
<img alt="Creative Commons Licence" style="border-width:0" src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png"/>
</a>
<br/>
This work is licensed under a
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">Creative Commons Attribution-NonCommercial 4.0 International License</a>.
</p>

You are free to:

- **Share** — copy and redistribute the material in any medium or format
- **Adapt** — remix, transform, and build upon the material

Under the following terms:

- **Attribution** — You must give appropriate credit and provide a link to this repository
- **NonCommercial** — You may not use the material for commercial purposes

Copyright 2026 akshansh11. All rights reserved for commercial use.
