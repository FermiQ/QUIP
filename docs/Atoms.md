# Atoms.f95

## Overview

The `Atoms.f95` file defines the `atoms_module`, which provides the core data structure and routines for representing a collection of atoms within a simulation. This includes their atomic numbers, dynamical variables (positions, velocities, etc.), properties (like mass or charge), and connectivity information (neighbour lists). It is a fundamental component of the libAtoms simulation package, serving as the primary container for atomic system information.

## Key Components

- **`MODULE atoms_module`**: The main Fortran module encapsulating all atom-related data types and procedures.
- **`TYPE Atoms`**: The derived data type that holds all information for a collection of atoms. Key members include:
    - `N`: Integer, number of atoms.
    - `Nbuffer`: Integer, allocated size for atom arrays.
    - `lattice`: Real(dp), dimension(3,3), stores the simulation cell lattice vectors.
    - `properties`: Type(Dictionary), stores per-atom properties (e.g., `pos`, `Z`, `mass`, `velo`).
    - `params`: Type(Dictionary), stores per-configuration parameters.
    - `connect`: Type(Connection), stores neighbour list information.
    - `hysteretic_connect`: Type(Connection), for hysteretic bond tracking.
- **`initialise(this, N, lattice, ...)` / `atoms_initialise(...)`**: Subroutine to initialize an `Atoms` object, allocating space for `N` atoms and setting the initial `lattice`.
- **`finalise(this)` / `atoms_finalise(...)`**: Subroutine to deallocate memory associated with an `Atoms` object.
- **`add_atoms(...)` (via interfaces `add_atom_single`, `add_atom_multiple`, `atoms_join`)**: Subroutines to add one or more atoms to an existing `Atoms` object.
- **`remove_atoms(...)` (via interfaces `remove_atom_single`, `remove_atom_multiple`, `remove_atom_multiple_mask`)**: Subroutines to remove one or more atoms.
- **`calc_connect(this, ...)` / `atoms_calc_connect(...)`**: Subroutine to calculate neighbour lists based on cutoff distances.
- **`calc_dists(this, ...)` / `atoms_calc_dists(...)`**: Subroutine to update stored distances in the connectivity object, typically after atom moves but before a full `calc_connect`.
- **`n_neighbours(this, i, ...)` / `atoms_n_neighbours(...)`**: Function to get the number of neighbours for a given atom `i`.
- **`neighbour(this, i, n, ...)` / `atoms_neighbour(...)`**: Function to get information (index, distance, shift vector) about the n-th neighbour of atom `i`.
- **`set_lattice(this, new_lattice, ...)` / `atoms_set_lattice(...)`**: Subroutine to change the simulation cell lattice vectors.
- **`map_into_cell(this)` / `atoms_map_into_cell(...)`**: Subroutine to map atomic positions back into the primary unit cell.
- **`centre_of_mass(at, ...)`**: Function to calculate the centre of mass of the system or a subset of atoms.
- **`get_param_value(...)` / `set_param_value(...)`**: Interfaces to get/set values in the `params` dictionary.
- **`has_property(this, name)` / `atoms_has_property(...)`**: Function to check if a specific property exists.
- **`add_property(this, name, ...)`**: (Though not directly public, it's used internally by `atoms_initialise` and others) Subroutine to add new per-atom properties.

## Important Variables/Constants

- **`NOT_NEIGHBOUR` (Integer Parameter)**: Returned by `Find_Neighbour` (internal to `Connection_module` but relevant here) if a neighbour is not found. Value is 0.
- **`this%cutoff` (Real(dp))**: Main cutoff radius for neighbour finding.
- **`this%cutoff_skin` (Real(dp))**: Skin distance for neighbour list builds. Connectivity is only fully rebuilt if an atom moves more than half this skin distance.
- **`this%Z` (Integer Pointer, dimension(:))**: Points to the array of atomic numbers for each atom.
- **`this%pos` (Real(dp) Pointer, dimension(3,:))**: Points to the array of atomic positions (Cartesian coordinates).
- **`this%species` (Character Pointer, dimension(:,:))**: Points to the array of atom species names.
- **`this%mass` (Real(dp) Pointer, dimension(:))**: Points to the array of atomic masses.
- **`this%fixed_size` (Logical)**: If true, the number of atoms cannot be changed by adding/removing atoms.

## Usage Examples

Detailed Fortran usage examples can be found in the `src/Programs/` directory or in tutorial materials provided with the QUIP/libAtoms package. A typical initialization sequence:

```fortran
use atoms_module
use system_module ! For matrix types, etc.

type(Atoms) :: my_atoms
real(dp) :: lattice_vectors(3,3)
integer :: n_atoms

! Define lattice_vectors and n_atoms
lattice_vectors = ...
n_atoms = ...

call initialise(my_atoms, n_atoms, lattice_vectors)

! ... add properties, set positions, etc. ...

! Calculate connectivity
call calc_connect(my_atoms)

! ... perform simulation steps ...

call finalise(my_atoms)
```

## Dependencies and Interactions

The `atoms_module` has dependencies on several other modules within libAtoms/QUIP:

- **`error_module`**: For error handling.
- **`system_module`**: Provides system-wide definitions and types (e.g., `dp` for double precision).
- **`units_module`**: For physical unit conversions.
- **`mpi_context_module`**: For MPI parallelization context.
- **`linearalgebra_module`**: For matrix and vector operations (e.g., `matrix3x3_inverse`).
- **`extendable_str_module`**: For flexible string handling.
- **`dictionary_module`**: Used for `properties` and `params` members.
- **`table_module`**: Used by `Connection` objects and for some list operations.
- **`paramreader_module`**: For reading parameters.
- **`periodictable_module`**: For element-specific data (e.g., covalent radii, names).
- **`Atoms_types_module`**: Contains the definition of the `Atoms` derived type itself and related types.
- **`Connection_module`**: Crucial for neighbour list calculations and storage (`this%connect`).
- **`DomainDecomposition_module`**: For handling domain decomposition in parallel simulations.
- **`minimization_module`**: Likely for energy minimization routines that would operate on an `Atoms` object.
- **`Quaternions_module`**: For rotational operations.

The `Atoms` object is central to most operations in a molecular simulation. It is passed to and manipulated by:
- Potential energy and force calculation routines (e.g., in a `Potential` module/object).
- Molecular dynamics or minimization algorithms (e.g., in a `DynamicalSystem` module/object).
- Analysis tools that require atomic coordinates, species, or connectivity.
- I/O routines for reading/writing simulation configurations.
