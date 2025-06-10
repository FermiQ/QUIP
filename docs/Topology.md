# Topology.f95

## Overview

The `Topology.f95` file defines the `topology_module`, which is dedicated to determining and managing the structural topology of atomic systems. This involves identifying chemical bonds, angles, dihedral angles, and improper torsions based on atomic connectivity. A key function of this module is the identification of residues (e.g., amino acids in proteins) and other molecular motifs using a predefined library. Once the topology is established, the module can generate output files in formats suitable for various molecular simulation packages, such as PDB for coordinates and PSF (Protein Structure File) for CHARMM/NAMD topologies. It also includes specialized functionalities for systems like silica and titania, including position-dependent charge calculations.

## Key Components

- **`MODULE topology_module`**: The main Fortran module.
- **`create_residue_labels_arb_pos(...)` / `create_residue_labels_internal(...)`**: Core subroutines that analyze an `Atoms` object to assign residue and atom type information. They use connectivity (neighbor lists) and a residue library (read by `next_motif`) to identify residues and assign appropriate CHARMM/AMBER atom types, residue names, and potentially charges.
- **`write_pdb_file(...)` (with variants `write_brookhaven_pdb_file`, `write_cp2k_pdb_file`)**: Subroutines to generate Protein Data Bank (PDB) files, which store atomic coordinates. Different variants cater to specific formatting requirements.
- **`write_psf_file(...)` / `write_psf_file_arb_pos(...)`**: Subroutines that create Protein Structure Files (PSF). These files define the molecular topology: lists of atoms with their types, charges, masses, and the explicit bonds, angles, dihedrals, and impropers connecting them. This is crucial for force fields like CHARMM.
- **List Creation Subroutines**:
    - `create_bond_list(...)`: Generates a table of all bonds based on neighbor connectivity.
    - `create_angle_list(...)`: Generates a table of all bond angles from the bond list.
    - `create_dihedral_list(...)`: Generates a table of all dihedral angles from the angle list.
    - `create_improper_list(...)`: Generates a table of improper dihedrals, often from a combination of recognized angles and pre-defined intra-residue impropers.
- **Motif and Molecule Identification**:
    - `next_motif(...)`: Reads a residue or molecular motif definition from a library file.
    - `find_molecule_ids(...)`: Identifies and labels separate molecules within the system based on connectivity.
    - `find_water_monomer(...)`, `find_A2_monomer(...)`, `find_AB_monomer(...)`, `find_general_monomer(...)`: Functions to locate specific small molecular units (e.g., H2O, diatomic molecules).
    - `find_monomer_pairs(...)`, `find_monomer_triplets(...)` (and MPI versions): Functions to find interacting pairs or triplets of monomers within a specified cutoff distance.
- **Specialized System Routines**:
    - `create_pos_dep_charges(at, SiOH_list, charge)`: Calculates position-dependent atomic charges for silica systems, based on the method by Cole et al.
    - `delete_metal_connects(at)`: Removes bonds involving metal ions, which are often treated non-bonded in classical force fields.

## Important Variables/Constants

- **Cutoff Radii for Specific Materials**:
    - `SILICON_2BODY_CUTOFF`, `SILICA_2BODY_CUTOFF`, `TITANIA_2BODY_CUTOFF`
    - `SILICON_3BODY_CUTOFF`, `SILICA_3BODY_CUTOFF`, `TITANIA_3BODY_CUTOFF`
  These define interaction ranges for bond and angle calculations in these materials.
- **Residue Library Limits**:
    - `MAX_KNOWN_RESIDUES`: Maximum number of different residue types that can be loaded from the library.
    - `MAX_ATOMS_PER_RES`: Maximum atoms a single residue definition can contain.
    - `MAX_IMPROPERS_PER_RES`: Maximum predefined improper dihedrals per residue.
- **Run Type Parameters**:
    - `NONE_RUN`, `QS_RUN`, `MM_RUN`, `QMMM_RUN_CORE`, `QMMM_RUN_EXTENDED`: Integer parameters likely used to flag different types of simulations or regions (e.g., in QM/MM setups), which might influence topology generation.

## Usage Examples

This module is typically used in pre-processing steps or by simulation setup tools to prepare the system's topology for force field calculations or for detailed structural analysis.

```fortran
! Conceptual example of using topology_module
use topology_module
use atoms_module
use system_module

type(Atoms) :: my_atoms
type(Table) :: bonds, angles, dihedrals, impropers
character(len=256) :: residue_lib_file, pdb_out_file, psf_out_file

! ... (Initialize my_atoms with positions, Z, etc.) ...
! ... (Set residue_lib_file, pdb_out_file, psf_out_file paths) ...

! Ensure connectivity is calculated
call calc_connect(my_atoms)

! Load residue library into atoms object's parameters (simplified)
call set_value(my_atoms%params, 'Library', residue_lib_file)

! Identify residues and assign atom/residue types and charges
call create_residue_labels_arb_pos(my_atoms, do_CHARMM=.true.)

! Generate PDB file
call write_brookhaven_pdb_file(my_atoms, pdb_out_file)

! Generate PSF file (topology)
call write_psf_file_arb_pos(my_atoms, psf_out_file)

! ... (Optionally, directly access bond/angle lists for analysis) ...
! call create_bond_list(my_atoms, bonds, .false.) ! .false. for not add_silica_23body
! call print('Number of bonds found: '//bonds%N)

call finalise(my_atoms)
! ...
```
This module is used by potential routines that require connectivity information (e.g., bond-order potentials, or potentials with explicit bond/angle terms) and by analysis tools.

## Dependencies and Interactions

- **`atoms_module` / `Atoms_types_module`**: This is the most critical dependency. `topology_module` reads atomic positions, atomic numbers (`Z`), and pre-calculated neighbor lists (via the `Connection` object) from an `Atoms` object. It then adds new derived properties to the `Atoms` object, such as `atom_type`, `atom_res_name`, `atom_mol_name`, `atom_res_number`, and `atom_charge`.
- **`connection_module`**: Used extensively for accessing neighbor information (`n_neighbours`, `neighbour`) and for graph traversal algorithms like Breadth-First Search (`bfs_step`) to identify molecules and connected components.
- **`system_module`**: For basic utilities like file I/O (`InOutput` type for reading residue libraries and writing PDB/PSF files), error handling (`system_abort`, `RAISE_ERROR`), and string manipulation.
- **`dictionary_module` and `table_module`**: Used for managing data structures, such as the residue library (likely stored or processed using dictionaries) and for creating and manipulating lists of bonds, angles, etc. (often stored in `Table` objects).
- **`periodictable_module`**: To get element-specific information (e.g., `ElementName`, `ElementCovRad`).
- **`clusters_module` and `structures_module`**: May be used for more advanced structural identification or analysis, although their direct use is not as prominent as `Connection`.
- **Potential evaluation routines**: Classical potentials that include terms for bonds, angles, dihedrals, and impropers will consume the topological information generated by this module (typically via a PSF file or by directly accessing the generated lists).
- **Simulation setup tools**: Programs that prepare simulation inputs often use this module to define the system's topology from a simpler input like a PDB file and a residue library.
