# md.f95 (Molecular Dynamics Program)

## Overview

The `md.f95` file contains the source code for the `md` program, a command-line executable designed for performing molecular dynamics (MD) simulations using the libAtoms/QUIP framework. This program handles the setup and execution of an MD run, including reading initial atomic configurations, defining the interatomic potential, specifying simulation parameters (ensemble, temperature, pressure, timestep, duration), running the simulation loop, and outputting trajectory and thermodynamic data.

## Key Components

- **`PROGRAM md`**: The main program block that drives the simulation.
- **`MODULE md_module`**: A module contained within `md.f95` that defines:
    - **`TYPE md_params`**: A derived data type to encapsulate the numerous parameters controlling an MD simulation. These parameters are typically read from a command-line or an input file. Key parameters include:
        - Input/Output: `atoms_filename`, `param_filename` (for potential), `trajectory_filename`.
        - Simulation Length: `N_steps`, `max_time`.
        - Integrator: `dt` (timestep).
        - Thermostat: `T_initial` (target temperature), `const_T` (flag for NVT), `langevin_tau`, `nose_hoover_tau`, `all_purpose_thermostat` and related flags/parameters.
        - Barostat: `p_ext` (external pressure), `const_P` (flag for NPT), `barostat_tau`.
        - Potential: `pot_init_args` (for initializing the potential), `pot_calc_args` (for potential calculations).
        - Output Control: `summary_interval`, `trajectory_print_interval`.
        - Other: `rng_seed`, `cutoff_skin`, `zero_momentum`, `continuation` (for restarting runs).
    - **`get_params(params, mpi_glob)`**: Subroutine to populate the `md_params` type by reading parameters. It first checks for an `md_params` file and, if not found, parses command-line arguments using `paramreader_module`.
    - **`print_params(params)`**: Subroutine to print the current values of the MD parameters.
    - **`initialise_md_thermostat(ds, params)`**: Subroutine to set up the thermostat and barostat within the `DynamicalSystem` object based on the user-specified parameters.
    - **`update_md_thermostat(ds, params)`**: Subroutine to update thermostat parameters, e.g., for variable temperature simulations.
    - **`do_prints(...)`**: Subroutine to handle periodic output of simulation data (summary, trajectory, etc.).

- **Main Program Flow**:
    1.  **Initialization**:
        - Calls `system_initialise()` for basic setup.
        - Initializes MPI context (`mpi_glob`).
        - Calls `get_params` to read and process all MD parameters.
        - Initializes the interatomic potential (`pot :: TYPE(Potential)`) using `param_filename` and `pot_init_args`.
        - Reads the initial atomic configuration (`at_in :: TYPE(Atoms)`) from `atoms_filename`.
        - Initializes the `DynamicalSystem` object (`ds`) using `at_in`.
        - Sets up simulation parameters in `ds` (e.g., `ds%avg_time`).
        - Initializes restraints and constraints via `init_restraints_constraints`.
        - Opens output files (trajectory via `CInOutput`, flux file).
        - Calls `initialise_md_thermostat` to configure thermostat/barostat in `ds`.
        - Optionally zeroes total momentum and/or angular momentum of the system.
        - Performs an initial force calculation using `Calc(pot, ds%atoms, ...)` to get initial forces and energies, and initializes accelerations in `ds%atoms%acc`.
    2.  **Molecular Dynamics Loop**:
        - Iterates from `initial_i_step` up to `N_steps` or until `max_time` is reached.
        - `update_md_thermostat`: If running a variable temperature simulation, updates the target temperature in the thermostat.
        - `advance_md` (internal helper, calls `advance_md_one`):
            - `advance_verlet1(ds, params%dt, ...)`: Performs the first part of the velocity Verlet integration step (updates positions $r(t) \rightarrow r(t+dt)$ and velocities $v(t) \rightarrow v(t+dt/2)$).
            - Updates neighbor lists (`calc_connect` or `calc_dists` in `ds%atoms`) based on atomic displacements and `cutoff_skin`.
            - `Calc(pot, ds%atoms, ...)`: Calculates new forces and energy based on the updated positions $r(t+dt)$.
            - `advance_verlet2(ds, params%dt, ...)`: Performs the second part of the Verlet step (updates velocities $v(t+dt/2) \rightarrow v(t+dt)$ using new forces).
        - `do_prints`: Periodically prints summary statistics, trajectory frames, and other requested data.
    3.  **Finalization**:
        - Performs final prints.
        - Calls `system_finalise()`.

## Important Variables/Constants

- **`params :: TYPE(md_params)`**: An instance of the `md_params` derived type, holding all configurable parameters for the MD run.
- **`ds :: TYPE(DynamicalSystem)`**: The object that encapsulates the atomic system's state (positions, velocities, etc., via its internal `Atoms` object) and manages the MD integration, including thermostat and barostat application.
- **`pot :: TYPE(Potential)`**: The interatomic potential used to calculate forces and energies.
- **Key parameters from `md_params`**:
    - `dt`: Timestep for the Verlet integrator.
    - `N_steps` / `max_time`: Duration of the simulation.
    - `T_initial`, `const_T`: Define temperature control (NVT ensemble).
    - `p_ext`, `const_P`: Define pressure control (NPT ensemble).
    - `langevin_tau`, `nose_hoover_tau`, `barostat_tau`: Time constants for thermostats/barostats.

## Usage Examples

The `md` program is used to run molecular dynamics simulations. Input parameters are typically specified either in a file named `md_params` in the run directory or via command-line arguments.

- **Example command-line usage**:
  ```bash
  md atoms_filename=start.xyz param_filename=potential.xml init_args="IP MyPot" \
     N_steps=10000 dt=1.0 T_initial=300.0 langevin_tau=100.0 \
     trajectory_filename=traj.xyz summary_interval=100 trajectory_print_interval=1000
  ```
  This command would run an NVT simulation at 300K for 10000 steps with a 1 fs timestep, using "MyPot" potential defined in `potential.xml`, starting from `start.xyz`, and saving trajectory frames to `traj.xyz`.

- **Example `md_params` file content**:
  ```
  atoms_filename = my_config.cif
  param_filename = eam_alloy.xml
  init_args = "IP EAM_ErcolAd"
  N_steps = 50000
  dt = 2.0
  T_initial = 500.0
  const_P = T
  p_ext = 1.0  # GPa
  barostat_tau = 200.0
  trajectory_filename = npt_run.nc
  summary_interval = 50
  trajectory_print_interval = 200
  ```

## Dependencies and Interactions

- **`DynamicalSystem_module`**: This is the core engine for the MD simulation. `md.f95` sets up and drives a `DynamicalSystem` object (`ds`), relying on its `advance_verlet1` and `advance_verlet2` methods for time integration, and its thermostat/barostat capabilities.
- **`Potential_module`**: Essential for calculating energies and forces. The `md` program initializes a `Potential` object (`pot`) and passes it to the `DynamicalSystem` (implicitly) or uses it directly for initial force calculations.
- **`Atoms_module` (`Atoms_types_module`)**: The `DynamicalSystem` object contains an `Atoms` object, which stores the coordinates, velocities, forces, and other per-atom data. The `md` program reads the initial configuration into an `Atoms` object.
- **`ParamReader_module`**: Used by the `get_params` subroutine within `md_module` to parse command-line arguments or the `md_params` input file.
- **`System_module`**: For overall program initialization (`system_initialise`), finalization (`system_finalise`), logging (`print`, `mainlog`), error handling, and MPI context management.
- **`CInOutput_module`**: Used for reading the initial atomic structure (`atoms_filename`) and writing trajectory data (`trajectory_filename`).
- **`Restraints_Constraints_XML_module`**: For reading and applying restraint or constraint definitions from an XML file.
- Specific potential implementation modules (e.g., `IPModel_LJ_module`, `IPModel_EAM_ErcolAd_module`, `IPModel_GAP_module`, etc.) are used indirectly via the `Potential_module`.

The `md` program brings together these modules to provide a complete molecular dynamics simulation capability.
