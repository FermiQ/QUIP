# Matrix.f95

## Overview

The `Matrix.f95` file defines the `matrix_module`, which provides data structures and a comprehensive suite of routines for performing linear algebra operations on real (`MatrixD`) and complex (`MatrixZ`) matrices. The module is designed to work with both serial computations and distributed-memory parallel computations via integration with ScaLAPACK. Its functionalities include matrix initialization, finalization, basic manipulations (zeroing, printing, scaling), and more advanced operations such as matrix multiplication, inversion, trace calculations, and eigenvalue problem solutions (diagonalization).

## Key Components

- **`MODULE matrix_module`**: The main Fortran module for matrix operations.
- **`TYPE MatrixD`**: A derived data type for representing real (double precision) matrices. It includes:
    - `N`, `M`: Global number of rows and columns.
    - `l_N`, `l_M`: Local number of rows and columns (relevant for ScaLAPACK).
    - `data(:,:)`: A pointer to the 2D array holding the matrix elements.
    - `ScaLAPACK_Info_obj`: A `Matrix_ScaLAPACK_Info` type object containing metadata for ScaLAPACK distribution and operations.
    - `use_allocate`: A logical flag indicating if the `data` pointer should be allocated/deallocated by the type's constructor/destructor.
- **`TYPE MatrixZ`**: A derived data type for complex (double precision) matrices, structured similarly to `MatrixD`.

- **Core Operations (Interfaces with specific procedures for MatrixD and MatrixZ)**:
    - **`Initialise`**: Subroutines to allocate and set up `MatrixD` or `MatrixZ` objects, including optional ScaLAPACK setup (e.g., `MatrixD_Initialise`, `MatrixZ_Initialise_mat`).
    - **`Finalise`**: Subroutines to deallocate matrix data and finalize any associated ScaLAPACK resources.
    - **`Wipe`**: Clears matrix data, deallocates memory (if `use_allocate` is true), and resets dimensions to zero.
    - **`Zero`**: Sets matrix elements to zero. Can be targeted to diagonal or off-diagonal elements.
    - **`Print`**: Prints the matrix dimensions, ScaLAPACK info (if active), and its elements.
    - **`add_block(...)`**: Adds a smaller source block matrix to a specified location within a larger target matrix.
    - **`diagonalise(...)`**: Solves standard (Ax = λx) or generalized (Ax = λBx) eigenvalue problems for symmetric/Hermitian matrices. Returns eigenvalues and optionally eigenvectors.
    - **`inverse(...)`**: Computes the inverse of a matrix.
    - **`matrix_product_sub(...)`**: Performs matrix multiplication C = A * B, with options for transposing and conjugating A and B. Handles various combinations of real and complex matrices.
    - **`TraceMult(...)`**: Computes the trace of a product of two matrices (Trace(A*B)).
    - **`partial_TraceMult(...)`**: Computes partial traces, resulting in a vector.
    - **`transpose_sub(this, m)`**: Transposes matrix `m` into `this`.
    - **`make_hermitian(this)`**: Makes a matrix Hermitian by setting `this = 0.5 * (this + this')`.
    - **`add_identity(A)`**: Adds the identity matrix to matrix A.
    - **`scale(A, factor)`**: Multiplies all elements of matrix A by a scalar factor.
    - **`multDiag(...)`**: Multiplies a matrix by a diagonal matrix (represented as a vector).
    - **`Re_diag(...)`**: Extracts the real part of the diagonal elements.
    - **`diag_spinor(...)`**: Extracts 2x2 blocks from the diagonal (for spinor representations).

## Important Variables/Constants

- The module primarily defines types and procedures. Constants related to numerical precision (e.g., `dp` for double precision) are typically imported from `System_module`.
- Internal parameters for LAPACK/ScaLAPACK calls (e.g., block sizes like `NB`, `MB` in `matrixany_initialise`) are handled within the routines.

## Usage Examples

This module provides utility functions used throughout the codebase for calculations involving vectors and matrices, e.g., in coordinate transformations, stress calculations, or fitting procedures.

```fortran
use matrix_module
use system_module ! For dp and print routines

type(MatrixD) :: A, A_inv, B, C
real(dp) :: trace_val
real(dp) :: eigenvalues(3)
integer :: n_dim

n_dim = 3
call Initialise(A, N=n_dim, M=n_dim)
call Initialise(B, N=n_dim, M=n_dim)
call Initialise(C, N=n_dim, M=n_dim)

! Assign some values to A%data and B%data
A%data = reshape((/ 1.0_dp, 0.0_dp, 0.0_dp, &
                     0.0_dp, 2.0_dp, 0.0_dp, &
                     0.0_dp, 0.0_dp, 3.0_dp /), (/n_dim, n_dim/))
B%data = reshape((/ 2.0_dp, 1.0_dp, 0.0_dp, &
                     1.0_dp, 2.0_dp, 1.0_dp, &
                     0.0_dp, 1.0_dp, 2.0_dp /), (/n_dim, n_dim/))

! C = A * B
call matrix_product_sub(C, A, B)
call Print(C)

! Calculate inverse of A
call Initialise(A_inv, N=n_dim, M=n_dim)
A_inv%data = A%data ! Copy A to A_inv before inversion
call inverse(A_inv)
call Print(A_inv)

! Calculate eigenvalues of B
call diagonalise(B, eigenvalues)
call print('Eigenvalues of B: ', eigenvalues)

! Calculate Trace(A*C)
trace_val = TraceMult(A, C)
call print('Trace(A*C): ', trace_val)

call Finalise(A)
call Finalise(B)
call Finalise(C)
call Finalise(A_inv)
```

## Dependencies and Interactions

- **`linearalgebra_module`**: This is a fundamental dependency. The `matrix_module` acts as a higher-level interface, likely calling routines from `linearalgebra_module` which, in turn, would wrap actual LAPACK calls for serial operations.
- **`ScaLAPACK_module`**: Provides the interface to ScaLAPACK routines for distributed parallel matrix operations. The `MatrixD` and `MatrixZ` types store ScaLAPACK-specific distribution information.
- **`System_module`**: Used for basic system utilities, including kind parameters for precision (e.g., `dp`), error reporting (`system_abort`), and memory allocation tracing (`ALLOC_TRACE`, `DEALLOC_TRACE`).
- **`error_module`**: For standardized error handling macros.
- **`MPI_context_module`**: Provides MPI context information necessary for ScaLAPACK initialization and operation.

The `matrix_module` is a general-purpose linear algebra library. It is likely used by:
- `Atoms_module`: For operations involving lattice vectors (e.g., calculating cell volume, inverse lattice vectors) or transforming coordinates.
- `DynamicalSystem_module`: For operations like transforming velocities, calculating kinetic energy tensors, or in advanced simulation algorithms.
- Potential calculation modules: Especially in QM/MM or advanced classical potentials where matrix diagonalization or other linear algebra operations might be needed (e.g., for solving electronic structure problems or in fitting procedures).
- Analysis tools: For various calculations like principal component analysis, tensor manipulations, etc.
