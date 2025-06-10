# quip.f95 (Main Program)

## Overview

The `quip.f95` file contains the source code for the `quip` program, which serves as the primary command-line executable and driver for the libAtoms/QUIP (QUantum mechanics and Interatomic Potentials) software package. This program is responsible for parsing user commands and parameters from the command line or input files. Based on these inputs, it orchestrates various computational tasks such_as single-point energy and force evaluations, geometry optimization (relaxation), molecular dynamics (though MD-specific setup seems more prominent in `md.f95`), calculation of material properties like elastic constants, phonon spectra analysis, and interatomic potential/descriptor evaluations. It leverages the rich ecosystem of modules within the libAtoms library to perform these diverse functionalities.

## Key Components

- **`PROGRAM quip`**: The main program block.
- **Command-Line Argument Parsing**:
    - Utilizes the `paramreader_module` to define and parse a comprehensive set of command-line arguments.
    - Key arguments include:
        - `atoms_filename`: Specifies the input file for atomic structures (e.g., XYZ, NetCDF).
        - `param_filename`: Specifies the XML file containing parameters for the interatomic potential.
        - `init_args`: String to initialize the chosen potential (e.g., "IP SW", "IP GAP").
        - `calc_args`: String to control what the potential calculates (e.g., "energy force virial").
        - `action` flags: Numerous flags to specify the task, such as:
            - `-E` or `-energy`: Calculate energy.
            - `-F` or `-forces`: Calculate forces.
            - `-V` or `-virial`: Calculate virial/stress.
            - `-L` or `-local`: Calculate local atomic contributions (e.g., local energy).
            - `-relax`: Perform geometry optimization.
            - `-cij` / `-c0ij`: Calculate relaxed/unrelaxed elastic constants.
            - `-phonons` / `-fine_phonons`: Perform phonon calculations.
            - `-test` / `-n_test`: Test potential gradients.
        - Task-specific parameters like `relax_tol`, `cij_dx`, `phonons_dx`.
        - I/O control: `output_file`, `verbosity`.
- **Initialization Phase**:
    - `system_initialise()`: Sets up the execution environment, including MPI (if enabled) and logging.
    - `Potential_Filename_Initialise(...)` or `Potential_Initialise(...)`: Initializes the `TYPE(Potential)` object (`pot`) based on `init_args` and parameters from `param_filename`.
    - Reads the initial atomic structure(s) from `atoms_filename` into a `TYPE(Atoms)` object (`at`) using `CInOutput_module`.
- **Main Processing Loop**:
    - The program can process multiple atomic configurations if the input file contains multiple frames.
    - Inside the loop, for each configuration:
        - Sets up the `Atoms` object (`at`), including calculating connectivity via `calc_connect` if a potential cutoff is defined.
        - Optionally creates residue labels via `create_residue_labels_arb_pos` (for biomolecules/CHARMM).
        - Optionally fills in atomic masses.
- **Task Dispatching and Execution**:
    - A series of `if` blocks check the boolean flags set by command-line arguments to determine which task(s) to perform.
    - **Energy/Force/Virial/Local Properties**: If flags like `do_E`, `do_F`, `do_V`, `do_local` are set, calls `Calc` on the `pot` object with appropriate `calc_args`. Results are typically printed to the output.
    - **Geometry Optimization (`do_relax`)**: Calls `minim` (or `precon_minim` if specified) on the `pot` object to relax the atomic structure. Supports various minimization algorithms (CG, FIRE, LBFGS) and options for relaxing positions and/or lattice.
    - **Elastic Constants (`do_c0ij`, `do_cij`)**: Calls `calc_elastic_constants` from `elasticity_module`.
    - **Phonon Calculations (`do_phonons`, `do_fine_phonons`)**: Calls `phonons_all` or `Phonon_fine_calc_print` from `phonons_module`. Can output phonon frequencies, eigenvectors, and force constant matrices.
    - **Gradient Testing (`do_test`, `do_n_test`)**: Calls `test_gradient` or `n_test_gradient` on the `pot` object to verify analytical gradients against finite differences.
    - **Descriptor Evaluation (for GAP potentials)**: If `has_descriptor_str` is true, it initializes a `descriptor` object and calculates/prints the descriptors for the current atomic configuration.
- **Output**:
    - Results (energies, forces, optimized structures, etc.) are printed to standard output by default, or to a file specified by `output_file`.
    - Trajectories during relaxation can be saved to a file specified by `relax_print_filename`.

## Important Variables/Constants

- **`pot :: TYPE(Potential)`**: Holds the initialized interatomic potential.
- **`at :: TYPE(Atoms)`**: Holds the current atomic configuration being processed.
- **`cli_params :: TYPE(Dictionary)`**: Used by `paramreader_module` to store parsed command-line arguments.
- **Numerous logical flags**: `do_E`, `do_F`, `do_V`, `do_relax`, `do_cij`, `do_phonons`, etc., control the execution flow.
- **Filename variables**: `atoms_filename`, `param_filename`, `output_file`, `relax_print_filename`.
- **Control parameters**: `init_args`, `calc_args`, `relax_tol`, `relax_iter`, `minim_method`, etc.

## Usage Examples

The `quip` program is the main command-line interface to the QUIP functionalities. Examples of usage:

- Calculate energy and forces for `input.xyz` using `MyPotential.xml`:
  `quip atoms_file=input.xyz param_filename=MyPotential.xml init_args="IP MyPot" -E -F`

- Relax an atomic structure:
  `quip atoms_file=input.xyz param_filename=MyPotential.xml init_args="IP MyPot" -relax -F -V relax_tol=1e-5`
  (Relax forces and cell, assuming `MyPot` supports virial calculation)

- Perform a calculation based on a command file:
  `quip command_file=run.ini`
  (Where `run.ini` would contain `atoms_filename=... param_filename=...` etc.)

Refer to user manuals or tutorials for detailed command-line options.

## Dependencies and Interactions

The `quip` program acts as a high-level orchestrator and depends on a wide range of modules from the libAtoms library:

- **`libAtoms_module`**: A top-level module that likely USEs many other necessary modules.
- **`System_module`**: For program initialization (`system_initialise`), finalization (`system_finalise`), managing output streams (`mainlog`, `errorlog`), command-line argument parsing (via `paramreader_module`), and timing.
- **`ParamReader_module`**: Crucial for defining and parsing the extensive set of command-line arguments.
- **`Atoms_module` (and `Atoms_types_module`)**: For creating, manipulating, and reading/writing atomic configurations.
- **`Potential_module`**: For initializing the chosen interatomic potential (`pot`) and calling its `Calc`, `Minim`, `Cutoff`, `test_gradient` methods.
- **Specific Potential Implementation Modules (e.g., `IPModel_LJ_module`, `IPModel_GAP_module`, etc.)**: These are used indirectly via the `Potential_module`.
- **`CInOutput_module`**: For reading atomic configurations from `atoms_filename` and writing trajectory files.
- **`DynamicalSystem_module`**: While not explicitly shown for MD runs in this specific program (MD might be handled by `md.f95`), the `Potential` interface it uses is central.
- **`Elasticity_module`**: For calculating elastic constants (`calc_elastic_constants`).
- **`Phonons_module`**: For phonon calculations (`phonons_all`, `Phonon_fine_calc_print`).
- **`LibAtoms_misc_utils_module`**: For miscellaneous utility functions.
- **`Descriptors_module` and `GP_Predict_module`**: If compiled with GAP support, for descriptor calculation and GP model prediction.
- **`Potential_Precon_Minim_module`**: If compiled with preconditioned minimizer support.

The `quip` program demonstrates how the various library components of QUIP are brought together to perform complex atomistic simulation tasks driven by user input.
