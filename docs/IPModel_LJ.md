# IPModel_LJ.f95

## Overview

The `IPModel_LJ.f95` file defines the `IPModel_LJ_module`, which implements the Lennard-Jones (LJ) 12-6 interatomic potential. This is a widely used, simple pair potential model often employed to describe interactions between neutral atoms or molecules, particularly noble gases or as a component in more complex force fields. The module handles parameter initialization from XML input, and the calculation of energy, forces, and virial contributions based on the LJ formula.

The functional form used in the code for a pair of atoms is:
$V(r) = \epsilon_{12} (\sigma/r)^{12} - \epsilon_6 (\sigma/r)^6$
This is equivalent to the standard $4 \epsilon [(\sigma/r)^{12} - (\sigma/r)^6]$ if $\epsilon_{12} = 4\epsilon\sigma^{12}$ and $\epsilon_6 = 4\epsilon\sigma^6$. The parameters $\epsilon_{12}$ and $\epsilon_6$ are typically provided directly in the input XML.

## Key Components

- **`MODULE IPModel_LJ_module`**: The main Fortran module for the Lennard-Jones potential.
- **`TYPE IPModel_LJ`**: A derived data type that stores all parameters for the LJ potential. Key members include:
    - `n_types`: Integer, number of distinct atom types for LJ interactions.
    - `atomic_num(:)`: Allocatable integer array mapping internal LJ types to atomic numbers (Z).
    - `type_of_atomic_num(:)`: Allocatable integer array, an inverse map from atomic number to internal LJ type.
    - `cutoff`: Real(dp), the global cutoff distance (maximum of all pair-specific cutoffs).
    - `sigma(:,:)`: Real(dp), 2D array storing the $\sigma$ parameter (finite distance where interparticle potential is zero) for each pair of LJ types.
    - `eps6(:,:)`: Real(dp), 2D array storing the $\epsilon_6$ coefficient for the attractive $r^{-6}$ term for each pair of LJ types.
    - `eps12(:,:)`: Real(dp), 2D array storing the $\epsilon_{12}$ coefficient for the repulsive $r^{-12}$ term for each pair of LJ types.
    - `cutoff_a(:,:)`: Real(dp), 2D array for pair-specific cutoff distances.
    - `energy_shift(:,:)`: Real(dp), 2D array for energy shift values at the cutoff to ensure $V(r_c)=0$.
    - `linear_force_shift(:,:)`: Real(dp), 2D array for values used to shift the force to zero at the cutoff.
    - `smooth_cutoff_width(:,:)`: Real(dp), 2D array defining the width of a polynomial switching function region for smoothly bringing energy and force to zero at the cutoff.
    - `do_tail_corrections`: Logical, flag to enable analytical long-range dispersion tail corrections to energy and virial.
    - `tail_corr_const`, `tail_c6_coeffs(:,:)`: Parameters for calculating tail corrections.
    - `only_inter_resid`: Logical, if true, interactions are only computed between atoms belonging to different residues.
    - `label`: Character string, an optional label for this potential instance, read from `args_str`.

- **`Initialise(this, args_str, param_str)` (Interface to `IPModel_LJ_Initialise_str`)**: Subroutine to initialize the LJ potential. It parses `args_str` (primarily for a `label`) and an XML-formatted `param_str` which contains detailed parameters like `n_types`, `per_type_data` (mapping types to atomic numbers), and `per_pair_data` (defining `sigma`, `eps6`, `eps12`, `cutoff`, and shifting options for each pair of types).
- **`Finalise(this)` (Interface to `IPModel_LJ_Finalise`)**: Subroutine to deallocate all allocatable arrays within the `IPModel_LJ` object.
- **`Print(this, file)` (Interface to `IPModel_LJ_Print`)**: Subroutine to print the initialized parameters of the LJ potential.
- **`Calc(this, at, e, local_e, f, virial, local_virial, args_str, mpi, error)` (Interface to `IPModel_LJ_Calc`)**: The core calculation routine. Given an `Atoms` object (`at`), it computes the total energy (`e`), per-atom local energy (`local_e`), per-atom forces (`f`), total virial tensor (`virial`), and per-atom local virial (`local_virial`) based on the LJ interactions. It iterates over atom pairs using neighbor lists provided by the `Atoms` object.
- **`IPModel_LJ_pairenergy(this, ti, tj, r)`**: Function that calculates the raw LJ potential energy for a pair of atoms of types `ti` and `tj` separated by distance `r`. Includes shifting and smoothing if enabled.
- **`IPModel_LJ_pairenergy_deriv(this, ti, tj, r)`**: Function that calculates the derivative of the LJ pair energy with respect to `r` (related to the force magnitude). Includes shifting and smoothing if enabled.
- **XML Parsing Handlers (`IPModel_startElement_handler`, `IPModel_endElement_handler`)**: Internal routines used by FoX XML parser to read parameters from the `param_str` during initialization.

## Important Variables/Constants

- **Input XML Parameters**:
    - `<LJ_params n_types="..." label="...">`: Main XML tag.
    - `<per_type_data type="..." atomic_num="...">`: Maps an internal type index to an atomic number.
    - `<per_pair_data type1="..." type2="..." sigma="..." eps6="..." eps12="..." cutoff="..." energy_shift="T/F" linear_force_shift="T/F" smooth_cutoff_width="...">`: Defines the LJ parameters for each pair of atom types.
- **Physical Parameters within `TYPE IPModel_LJ`**: `sigma`, `eps6`, `eps12`, `cutoff_a`.
- **Shifting/Smoothing Parameters**: `energy_shift`, `linear_force_shift`, `smooth_cutoff_width`.

## Usage Examples

This module is selected and configured via input files when a Lennard-Jones potential is desired. The `ParamReader.f95` module would parse the high-level arguments specifying "IP LJ" and the XML parameter file/string. The `Potential.f95` module would then use `IPModel_LJ_module` to initialize and perform calculations, and `DynamicalSystem.f95` would use it via the generic `Potential` interface.

An example of the XML parameter structure:
```xml
<Potential>
  <LJ_params n_types="1" label="Argon_LJ">
    <per_type_data type="1" atomic_num="18" />
    <per_pair_data type1="1" type2="1" sigma="3.405" eps6="0.01032" eps12="1.698e-5" cutoff="8.5" energy_shift="T" linear_force_shift="F" />
    <!-- For eps6=4*eps*sigma^6, eps12=4*eps*sigma^12. Here eps6 is likely 4*epsilon*(sigma^6) and eps12 is 4*epsilon*(sigma^12) from direct input of epsilon and sigma -->
  </LJ_params>
</Potential>
```

## Dependencies and Interactions

- **`Potential_module` (specifically `Potential_simple_module` which includes `IPModel_interface.h`)**: `IPModel_LJ_module` implements the interface defined for simple interatomic potentials, allowing it to be used polymorphically by the main `Potential` type.
- **`Atoms_module` (`Atoms_types_module`)**: The `Calc` routine requires an `Atoms` object to access atomic positions, atomic numbers (to map to LJ types), and neighbor lists (obtained via `n_neighbours` and `neighbour` which are part of the `Atoms` type methods).
- **`System_module`**: Provides fundamental types like `dp` (double precision), I/O utilities (`inoutput`, `print`), error handling (`system_abort`), and string operations.
- **`dictionary_module`, `paramreader_module`**: Used for parsing the `args_str` during initialization.
- **`QUIP_Common_module`**: Provides XML parsing utilities (via FoX library wrappers like `QUIP_FoX_get_value`) and mathematical helper functions like `poly_switch` and `dpoly_switch` for implementing smoothed cutoffs.
- **`units_module`, `linearalgebra_module`**: For physical constants or basic math if needed, though direct use is minimal in this specific module.
- **`mpi_context_module`**: For MPI support, allowing calculations to be parallelized. The `Calc` routine has an optional `mpi` argument and can sum contributions if running in parallel.

This module provides a concrete implementation of a widely used pair potential, fitting into the larger `Potential` framework of QUIP.
