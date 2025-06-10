# IPModel_EAM_Ercolessi_Adams.f95

## Overview

The `IPModel_EAM_Ercolessi_Adams.f95` file defines the `IPModel_EAM_ErcolAd_module`. This module implements the Embedded Atom Model (EAM) interatomic potential as developed by Liu, Ercolessi, and Adams (Modelling Simul. Mater. Sci. Eng. 12, 665-670, 2004). EAM potentials are widely used for simulating metals and alloys. Unlike potentials with fixed analytical forms, this variant defines the core EAM functions—the pair repulsion, the electron density function, and the embedding energy function—using tabulated data points. The module then employs cubic splines to interpolate these functions smoothly, allowing for the calculation of energies and forces.

The total energy in an EAM model is typically expressed as:
$E_{tot} = \sum_i F_i(\bar{\rho}_i) + \frac{1}{2} \sum_{i \neq j} V_{ij}(r_{ij})$
where $F_i$ is the embedding energy of atom $i$ as a function of the host electron density $\bar{\rho}_i$ at its site, and $V_{ij}$ is a pair potential term. The host electron density $\bar{\rho}_i$ is a sum of contributions from neighboring atoms: $\bar{\rho}_i = \sum_{j \neq i} \rho_j(r_{ij})$.

## Key Components

- **`MODULE IPModel_EAM_ErcolAd_module`**: The main Fortran module for this EAM potential.
- **`TYPE IPModel_EAM_ErcolAd`**: A derived data type storing all parameters and spline objects for the potential. Key members include:
    - `n_types`: Integer, number of different atom types.
    - `atomic_num(:)`: Allocatable integer array mapping internal EAM types to atomic numbers (Z).
    - `type_of_atomic_num(:)`: Allocatable integer array, an inverse map from atomic number to internal EAM type.
    - `cutoff`: Real(dp), the global maximum cutoff distance derived from per-pair cutoffs.
    - `r_min(:,:)`, `r_cut(:,:)`: Real(dp), 2D arrays for pair-specific minimum interaction distances and cutoff radii.
    - `spline_V(:,:,:,:)`: Real(dp), raw tabulated data points for the pair potential function $V(r)$ for each pair of types.
    - `spline_rho(:,:,:)`: Real(dp), raw tabulated data points for the atomic electron density function $\rho(r)$ for each type.
    - `spline_F(:,:,:)`: Real(dp), raw tabulated data points for the embedding energy function $F(\bar{\rho})$ for each type.
    - `V(:,:)`: 2D array of `TYPE(Spline)`, storing the spline objects for the pair potential $V(r)$ for each pair of types.
    - `rho(:)`: 1D array of `TYPE(Spline)`, storing the spline objects for the electron density function $\rho(r)$ for each type.
    - `F(:)`: 1D array of `TYPE(Spline)`, storing the spline objects for the embedding function $F(\bar{\rho})$ for each type.
    - `V_F_shift(:)`: Real(dp), a per-type shift parameter applied during energy calculation.
    - `label`: Character string, an optional label for this potential instance.

- **`Initialise(this, args_str, param_str)` (Interface to `IPModel_EAM_ErcolAd_Initialise_str`)**: Subroutine to set up the EAM potential. It parses `args_str` (e.g., for a `label`) and an XML-formatted `param_str`. The XML input must define `n_types`, `per_type_data` (linking types to atomic numbers and specifying `V_F_shift`), `per_pair_data` (defining `r_min` and `r_cut`), and then data for `spline_V`, `spline_rho`, and `spline_F` using `<point r="..." y="..." b="..." c="..." d="..."/>` elements. These points are then used to construct the actual `Spline` objects.
- **`Finalise(this)` (Interface to `IPModel_EAM_ErcolAd_Finalise`)**: Subroutine to deallocate all splines and other allocatable arrays within the `IPModel_EAM_ErcolAd` object.
- **`Print(this, file)` (Interface to `IPModel_EAM_ErcolAd_Print`)**: Subroutine to print the initialized parameters and details of the splines.
- **`Calc(this, at, ...)` (Interface to `IPModel_EAM_ErcolAd_Calc`)**: The core routine for calculating energies and forces. For each atom `i`:
    1.  It sums electron density contributions $\rho_j(r_{ij})$ from neighboring atoms `j` to get the total host density $\bar{\rho}_i$.
    2.  It calculates the embedding energy $F_i(\bar{\rho}_i)$.
    3.  It sums the pair potential terms $V_{ij}(r_{ij})$.
    The values and derivatives of $V$, $\rho$, and $F$ are obtained by evaluating the corresponding spline objects.
- **Spline Evaluation Functions**:
    - `eam_spline_V(this, ti, tj, r)`, `eam_spline_rho(this, ti, r)`, `eam_spline_F(this, ti, rho)`: Evaluate the V, rho, and F splines respectively.
    - `eam_spline_V_d(...)`, `eam_spline_rho_d(...)`, `eam_spline_F_d(...)`: Evaluate the derivatives of the V, rho, and F splines, used for force calculations.
- **XML Parsing Handlers (`IPModel_startElement_handler`, `IPModel_endElement_handler`)**: Internal routines used by the FoX XML library to parse the `param_str` during initialization.

## Important Variables/Constants

- **Tabulated EAM Functions**: The core of the potential is defined by the sets of (r, y, b, c, d) points for $V(r)$, $\rho(r)$, and $F(\bar{\rho})$ provided in the XML input. These points are used to construct `Spline` objects.
- **`r_cut(:,:)`**: Pair-specific cutoff radii beyond which interactions are zero.
- **XML Tags**: `<EAM_ErcolAd_params>`, `<per_type_data>`, `<per_pair_data>`, `<spline_V>`, `<spline_rho>`, `<spline_F>`, and `<point r="..." y="..." ... />` are essential for the input parameter file structure.

## Usage Examples

This EAM potential is specified in the input, with parameters typically read from standard EAM potential files (often in `funcfl` or setfl formats, though here an XML wrapper is used to provide the spline points). It's used by `DynamicalSystem.f95` through the `Potential` interface.

An example snippet of the XML structure:
```xml
<Potential>
  <EAM_ErcolAd_params n_types="1" label="MyAlloy" n_spline_V="100" n_spline_rho="100" n_spline_F="100">
    <per_type_data type="1" atomic_num="13" V_F_shift="0.0" />
    <per_pair_data atomic_num_i="13" atomic_num_j="13" r_min="2.0" r_cut="5.5" />
    <spline_V>
      <point r="2.0" y="0.5" b="..." c="..." d="..."/>
      <!-- ... more points ... -->
    </spline_V>
    <spline_rho>
      <point r="2.0" y="1.2" b="..." c="..." d="..."/>
      <!-- ... more points ... -->
    </spline_rho>
    <spline_F>
      <point r="0.1" y="-0.5" b="..." c="..." d="..."/> <!-- Note: 'r' here is density rho -->
      <!-- ... more points ... -->
    </spline_F>
  </EAM_ErcolAd_params>
</Potential>
```

## Dependencies and Interactions

- **`Potential_module` (via `IPModel_interface.h`)**: This module implements a specific interatomic potential model, conforming to the general interface provided by the `Potential_module` (likely through `Potential_simple_module`).
- **`atoms_module` (`Atoms_types_module`)**: The `Calc` routine operates on an `Atoms` object, using atomic positions, types (via `Z`), and neighbor lists (`n_neighbours`, `neighbour`).
- **`spline_module`**: This is a critical dependency. The tabulated EAM functions ($V, \rho, F$) are interpolated using `Spline` objects. `spline_value` and `spline_deriv` from this module are used extensively in `Calc`.
- **`System_module`**: Provides fundamental types like `dp` (double precision), I/O utilities (`inoutput`, `print`), and error handling (`system_abort`).
- **`dictionary_module`, `paramreader_module`**: Used for parsing the `args_str` during initialization.
- **`QUIP_Common_module`**: Provides XML parsing utilities (FoX wrappers like `QUIP_FoX_get_value`).
- **`mpi_context_module`**: For MPI support, enabling parallel calculations within the `Calc` routine.

This module is a concrete implementation of the EAM formalism, designed to be used within the broader QUIP simulation framework. It allows for flexible EAM potentials where the functional forms are determined by tabulated data rather than fixed analytical expressions.
