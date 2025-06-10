# DynamicalSystem.f95

## Overview

The `DynamicalSystem.f95` file defines the `dynamicalsystem_module`. This module is responsible for managing the time evolution of a system of atoms. It orchestrates molecular dynamics (MD) simulations and structural optimizations by applying integration algorithms, thermostatting and barostatting methods, and handling constraints or rigid bodies. It acts as a high-level driver for simulations, utilizing an `Atoms` object to store the state of the particles.

## Key Components

- **`MODULE dynamicalsystem_module`**: The main Fortran module.
- **`TYPE DynamicalSystem`**: The core derived data type that encapsulates all information needed to run a simulation. Key members include:
    - `atoms`: Pointer to an `Atoms` object, holding particle data (positions, velocities, forces, mass, etc.).
    - `t`: Real(dp), current simulation time.
    - `dt`: Real(dp), last integration time step used.
    - `nSteps`: Integer, number of integration steps performed.
    - `Epot`, `Ekin`, `Wkin`: Real(dp), potential energy, kinetic energy, and kinetic virial tensor.
    - `cur_temp`, `avg_temp`: Real(dp), current and time-averaged temperature.
    - `thermostat`: Allocatable array of `thermostat` type, for temperature control.
    - `barostat`: `barostat` type, for pressure control.
    - `constraint`, `restraint`: Allocatable arrays of `Constraint` type, for geometric constraints.
    - `rigidbody`: Allocatable array of `RigidBody` type.
    - `group`, `group_lookup`: For managing groups of atoms with different integration types.
- **`ds_initialise(this, atoms_in, ...)`**: Subroutine to initialize a `DynamicalSystem` using an existing `Atoms` object. It also sets up default properties like `velo`, `acc`, `mass` if not already present in the `Atoms` object.
- **`ds_finalise(this, ...)`**: Subroutine to deallocate memory associated with a `DynamicalSystem`.
- **`advance_verlet1(this, dt, ...)`**: The first part of the velocity Verlet integration step. It advances velocities by `dt/2` and then positions by `dt`.
- **`advance_verlet2(this, dt, f, ...)`**: The second part of the velocity Verlet integration step. It takes the newly calculated forces `f`, updates accelerations, and then advances velocities by another `dt/2`.
- **`advance_verlet(ds, dt, f, ...)`**: A convenience routine that calls `advance_verlet2` followed by `advance_verlet1`.
- **`ds_add_thermostat(this, type, T, ...)`**: Subroutine to add a thermostat (e.g., Langevin, Nose-Hoover) to the system, specifying target temperature `T` and coupling parameters.
- **`ds_set_barostat(this, type, p_ext, ...)`**: Subroutine to configure a barostat for constant pressure simulations, specifying the external pressure `p_ext` and coupling parameters.
- **`ds_add_constraint(this, atoms, func, data, ...)`**: Subroutine to add a geometric constraint (e.g., fixed bond length, angle) to a set of `atoms`. Helper routines like `constrain_bondlength` utilize this.
- **`temperature(this, ...)`**: Function to calculate the instantaneous or average temperature of the system or a sub-region.
- **`rescale_velo(this, temp, ...)`**: Subroutine to rescale atomic velocities to match a target temperature `temp`.
- **`zero_momentum(this, ...)`**: Subroutine to adjust atomic velocities to ensure the total linear momentum of the system (or a subset of atoms) is zero.
- **`zero_angular_momentum(this)`**: Subroutine to adjust atomic velocities to ensure the total angular momentum of the system (about its center of mass) is zero.
- **`TYPE_ATOM`, `TYPE_CONSTRAINED`, `TYPE_RIGID` (Integer Parameters)**: Define how different groups of atoms are treated during integration (normal, constrained, or part of a rigid body).

## Important Variables/Constants

- **`dt` (Real(dp))**: The time step for integration, passed to `advance_verlet1`/`advance_verlet2`.
- **`T` (Real(dp))**: Target temperature for thermostats and velocity rescaling.
- **`p_ext` (Real(dp))**: Target external pressure for barostats.
- **`this%atoms%velo` (Real(dp), Pointer, Dim(:,:))**: Atomic velocities.
- **`this%atoms%acc` (Real(dp), Pointer, Dim(:,:))**: Atomic accelerations (derived from forces).
- **`this%atoms%force` (Real(dp), Pointer, Dim(:,:))**: Atomic forces (typically calculated by a separate potential module and passed to `advance_verlet2`).
- **`this%atoms%mass` (Real(dp), Pointer, Dim(:))**: Atomic masses.
- **`this%atoms%move_mask` (Integer, Pointer, Dim(:))**: Mask indicating if an atom is mobile (1) or fixed (0).
- **`this%atoms%thermostat_region` (Integer, Pointer, Dim(:))**: Specifies which thermostat (if any) applies to each atom.
- **`this%thermostat(:)%gamma` or `this%thermostat(:)%tau` (Real(dp))**: Damping parameter or relaxation time for Langevin or Nose-Hoover thermostats.
- **`this%thermostat(:)%Q` (Real(dp))**: "Mass" parameter for Nose-Hoover thermostats.
- **`this%barostat%tau_epsilon` (Real(dp))**: Relaxation time for the barostat.
- **`CONSTRAINT_WARNING_TOLERANCE` (Real(dp))**: Tolerance for constraint satisfaction before issuing a warning.

## Usage Examples

Usage examples can be found in `src/Programs/md.f95` or specific simulation input files. A conceptual MD loop:

```fortran
use dynamicalsystem_module
use atoms_module
! ... potentially a potential_module ...

type(DynamicalSystem) :: ds
type(Atoms) :: atoms_obj
real(dp) :: timestep, forces_on_atoms(3, ds%atoms%N)
! ... (Initialize atoms_obj, then ds from atoms_obj) ...
! ... (Initialize potential) ...
! ... (Add thermostats, barostats, constraints if needed) ...

call calc_connect(ds%atoms) ! Initial neighbor list

do istep = 1, num_steps
    ! 1. Advance velocities by dt/2, then positions by dt
    call advance_verlet1(ds, timestep)

    ! 2. Calculate forces (and potential energy) based on new positions
    !    (This is typically done by an external potential routine)
    ! call potential_calc(ds%atoms, force=forces_on_atoms, energy=ds%Epot)
    ds%atoms%force = forces_on_atoms ! Assuming force is a property in Atoms

    ! 3. Advance velocities by another dt/2 using new forces
    call advance_verlet2(ds, timestep, ds%atoms%force) ! Pass forces to verlet2

    ! 4. Print status, save trajectories, etc.
    call ds_print_status(ds, epot=ds%Epot)

    ! 5. Periodically update neighbor list if needed
    if (mod(istep, rebuild_connect_interval) == 0) then
        call calc_connect(ds%atoms)
    end if
end do

call finalise(ds)
```

## Dependencies and Interactions

The `dynamicalsystem_module` is a central orchestrator and thus has several key dependencies:

- **`atoms_module`**: Essential. The `DynamicalSystem` contains and operates on an `Atoms` object, which stores all particle data (positions, velocities, masses, forces, etc.).
- **Potential/Force Calculation Module (e.g., `Potential.f95`, `ForceCalculator.f95` - often implicitly used)**: This is a critical interaction. `DynamicalSystem` relies on an external component to calculate the forces acting on atoms at each step. The forces are then used in `advance_verlet2` to update accelerations and velocities.
- **`thermostat_module`**: Provides the implementations for various thermostat algorithms (Langevin, Nose-Hoover, etc.) that `DynamicalSystem` uses for temperature control.
- **`barostat_module`**: Provides implementations for barostat algorithms used for pressure control.
- **`constraints_module`**: Provides the logic for handling geometric constraints (e.g., SHAKE, RATTLE).
- **`rigidbody_module`**: For integrating the motion of rigid bodies.
- **`group_module`**: Used to define and manage groups of atoms that might be treated differently (e.g., different integration schemes, thermostats).
- **`error_module`, `system_module`, `units_module`, `mpi_context_module`, `linearalgebra_module`**: General utility modules for error handling, system parameters, unit conversions, MPI support, and linear algebra operations.

The `DynamicalSystem` provides the main loop and integration logic for simulations. It interacts with the potential to get forces, with thermostat/barostat modules to control temperature/pressure, and with constraint modules to enforce geometric restrictions, all while updating the state of the `Atoms` object.
