# Potential.f95

## Overview

The `Potential.f95` file defines the `Potential_module`, which provides a generic framework and interface for interatomic potentials within the QUIP (QUantum mechanics and Interatomic Potentials) software package. It establishes a `TYPE Potential` that acts as a versatile wrapper or base class, allowing various types of force fields—ranging from simple pair potentials and many-body classical potentials to tight-binding models and interfaces to external codes—to be used in a consistent manner. This abstraction is crucial for modules like `DynamicalSystem` that require energy and force calculations without needing to know the specifics of the underlying potential model.

## Key Components

- **`MODULE Potential_module`**: The main Fortran module that encapsulates the potential framework.
- **`TYPE Potential`**: The central derived data type. It is not a strict abstract base type but functions as one, capable of representing different potential forms through internal pointers and flags. Key aspects include:
    - Pointers to other `Potential` objects (`l_mpot1`, `l_mpot2`) for constructing composite potentials (e.g., `Sum`, `ForceMixing`).
    - `is_simple`: A logical flag indicating if it's a basic potential type (handled by `Potential_simple_module` or specific IP/TB modules).
    - `simple`: An object of `TYPE(Potential_simple)`, which likely acts as a container or interface for individual potential models like Lennard-Jones, Stillinger-Weber, EAM, etc.
    - Flags and pointers for composite potential types: `is_sum` (for `Potential_Sum`), `is_forcemixing` (for `Potential_ForceMixing`), `is_evb`, `is_local_e_mix`, `is_oniom`, `is_cluster`.
    - Configuration strings: `init_args_pot1`, `init_args_pot2` (for composite potentials), `xml_label` (to select specific parameter sets from an XML file), `xml_init_args` (arguments read from XML), and `calc_args` (default arguments for calculations).
    - Global scaling factors: `r_scale` for distances and `E_scale` for energy.

- **Core Operations (Interfaces)**:
    - **`Initialise(this, args_str, ...)`**: Subroutine to initialize a `Potential` object. It typically takes an `args_str` (a string defining the potential type, e.g., "IP SW", "Sum", "TB DFTB") and parameters, which can be provided as an XML string (`param_str`) or from a file (`param_filename` or `io_obj`). This is where the specific potential model is chosen and its parameters are loaded.
    - **`Finalise(this)`**: Subroutine to deallocate resources associated with a `Potential` object.
    - **`Calc(this, at, ...)`**: The primary routine for performing calculations. It computes quantities like energy, forces, virial, local energy, and local virial for a given `Atoms` object (`at`). What is calculated and where results are stored (e.g., in `at%params`, `at%properties`, or directly into output arrays) is controlled by an `args_str` specific to this call.
    - **`Cutoff(this)`**: Function that returns the interaction cutoff radius of the potential. This is used to set up neighbor lists in the `Atoms` object.
    - **`Minim(this, at, method, ...)`**: Subroutine to perform energy minimization (geometry optimization) of an `Atoms` object using the defined potential.
    - **`test_gradient(pot, at, ...)`**: Subroutine for numerically checking the consistency between the potential energy and the calculated forces (analytical gradients).
    - **`set_callback(...)`**: Allows setting a Python callback function for potentials of type `CallbackPot`.
    - **`run(ds, pot, ...)`**: Interface to `DynamicalSystem_run`, allowing the potential to drive a molecular dynamics simulation.

- **Supported Potential Types (via `args_str` in `Initialise`)**:
    - Basic: `IP` (Interatomic Potential), `TB` (Tight Binding).
    - Wrappers/Interfaces: `FilePot` (external code via files), `CallbackPot` (Python callback), `KIM` (OpenKIM models).
    - Composite: `Sum` (sum of two potentials), `ForceMixing`, `EVB` (Empirical Valence Bond), `Local_E_Mix`, `ONIOM`, `Cluster`.
    - Numerous specific IP models (e.g., `LJ`, `SW`, `EAM_ErcolAd`, `Brenner`, `Tersoff`, `GAP`) and TB models (e.g., `DFTB`, `GSP`) are listed in the comments.

## Important Variables/Constants

- **`args_str` (Character string)**: Crucial input to `Initialise` and `Calc`. For `Initialise`, it dictates the type of potential and its composition. For `Calc`, it specifies which quantities to compute (e.g., "energy force virial").
- **`param_str` / `param_filename` (Character string)**: Provides the parameters for the potential, often in XML format.
- **`xml_label` (Character string in `TYPE Potential`)**: Used to select a specific `<Potential label="...">` block from a larger XML parameter file.
- **Internal pointers in `TYPE Potential`**: `simple`, `sum`, `forcemixing`, etc., point to the actual data/methods for the specific potential type being used.

## Usage Examples

This module defines the interface for potentials. Specific implementations are found in files like `IPModel_LJ.f95`, `IPModel_EAM_Ercolessi_Adams.f95`, etc. The `DynamicalSystem` module uses this interface to compute forces.

```fortran
use Potential_module
use Atoms_module
use System_module

type(Potential) :: lj_potential, sw_potential, sum_potential
type(Atoms) :: my_atoms
real(dp) :: energy_val
real(dp) :: forces_val(3, my_atoms%N)

! Initialize atoms (example)
! call create_diamond_structure(my_atoms, alat=3.57, ElementName='C')
! call set_cutoff(my_atoms, 5.0_dp) ! Initial cutoff

! Initialize a Lennard-Jones potential from an XML file
call Initialise(lj_potential, args_str="IP LJ", param_filename="lj_params.xml")

! Initialize a Stillinger-Weber potential using an inline XML string
character(len=:), allocatable :: sw_xml_params
sw_xml_params = "<SW_params n_types='1'>...</SW_params>" ! (Content of SW.xml)
call Initialise(sw_potential, args_str="IP SW", param_str=sw_xml_params)

! Create a sum potential
call Initialise(sum_potential, args_str="Sum", pot1=lj_potential, pot2=sw_potential)

! Calculate energy and forces using the sum potential
call Calc(sum_potential, my_atoms, energy=energy_val, force=forces_val)
call print("Total Energy: ", energy_val)

! Clean up
call Finalise(lj_potential)
call Finalise(sw_potential)
call Finalise(sum_potential)
call Finalise(my_atoms)
```

## Dependencies and Interactions

- **`Atoms_module` (`Atoms_types_module`)**: This is a primary interaction. The `Calc` method takes an `Atoms` object to obtain coordinates, species, and cell data. It updates atomic forces and system energy/virial, often stored back into the `Atoms` object's properties or parameters. The `Potential` module also queries and can modify `Atoms%cutoff` and trigger `calc_connect`.
- **`DynamicalSystem_module`**: A major consumer of the `Potential` interface. It calls `Calc` to get forces and energies needed for molecular dynamics simulations or geometry optimizations.
- **`Potential_simple_module` and other specific potential modules (e.g., `TB_module`, `adjustablepotential_module`, `Potential_Sum_module`, etc.)**: The generic `Potential` type delegates actual calculations to these more specialized modules based on its initialized type.
- **`paramreader_module`**: Used during `Initialise` to parse the `args_str` and the XML parameter strings/files that define the potential.
- **`System_module`**: For low-level utilities like I/O (`InOutput` for reading parameter files), error handling, and numerical kinds (`dp`).
- **`linearalgebra_module`, `units_module`, `periodictable_module`, `dictionary_module`, `table_module`, `mpi_context_module`, `minimization_module`, `connection_module`**: These provide various support functions for calculations, data handling, and parallel execution.
- **External libraries/codes**: Some potential types (e.g., `KIM`, `FilePot`) act as interfaces to external potential libraries or simulation codes.
