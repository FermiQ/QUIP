# System.f95

## Overview

The `System.f95` file defines the `system_module`, a foundational utility module within the libAtoms/QUIP package. It provides a wide range of low-level system-dependent functionalities essential for the operation of the simulation software. These include robust input/output (I/O) management through the `InOutput` derived type, random number generation, CPU and wall-clock timing, string manipulation utilities, command-line argument parsing, access to environment variables, and basic MPI (Message Passing Interface) environment information. It also defines global constants for numerical precision and verbosity levels used throughout the codebase.

## Key Components

- **`MODULE system_module`**: The main Fortran module.
- **`TYPE InOutput`**: A derived data type that abstracts file I/O operations. It can handle formatted or unformatted files, standard streams (stdout, stdin, stderr), and named files. Key associated routines:
    - `initialise(this, filename, ...)`: Opens/sets up an `InOutput` object.
    - `finalise(this)`: Closes an `InOutput` object.
    - `print(string, verbosity, file, ...)`: Overloaded subroutine to print various data types (strings, integers, reals, logicals) to an `InOutput` object, controlled by verbosity levels.
    - `read_line(this, status)`: Function to read a line of text.
    - `parse_line(this, delimiters, fields, ...)`: Subroutine to parse a line read from an `InOutput` object.
- **Precision Kind Parameters**:
    - `dp`: Double precision real kind.
    - `qp`: Quadruple precision real kind (availability may depend on compiler).
    - `isp`, `idp`: Integer kinds (likely standard and long integers).
- **Standard I/O Objects**:
    - `mainlog`: A public `InOutput` object, typically initialized to standard output (`stdout`). Default target for general messages.
    - `errorlog`: A public `InOutput` object, typically initialized to standard error (`stderr`). Default target for error messages.
    - `mpilog`: A public `InOutput` object for MPI-specific logging, often per-process.
- **Verbosity Level Parameters**: Integer constants like `PRINT_ALWAYS`, `PRINT_SILENT`, `PRINT_NORMAL`, `PRINT_VERBOSE`, `PRINT_NERD`, `PRINT_ANALYSIS` used to control the detail of output.
- **Core Subroutines**:
    - `system_initialise(...)`: Initializes MPI (if enabled), sets up random number generators, prepares `mainlog` and `errorlog`, and prints a welcome banner. Reads command-line arguments.
    - `system_finalise()`: Performs cleanup, including MPI finalization.
    - `system_timer(name, ...)`: A utility to measure elapsed CPU and wall clock time for specified code sections.
    - `system_abort(message)`: Terminates the program after printing an error message.
- **Random Number Generators**:
    - `ran()`: Returns a random integer.
    - `ran_uniform()`: Returns a random real number uniformly distributed in [0,1].
    - `ran_normal()`: Returns a random real number from a Normal distribution (mean 0, stddev 1).
- **String Utilities**:
    - `split_string(...)`, `parse_string(...)`, `split_string_simple(...)`: For tokenizing strings.
    - `upper_case(word)`, `lower_case(word)`: For case conversion.
    - `string_cat_*` (Operator `//` overloads): For concatenating strings with other data types.
    - `a2s(array)`, `s2a(string)`: Convert between character arrays and strings.
- **Command Line and Environment**:
    - `cmd_arg_count()`: Returns the number of command-line arguments.
    - `get_cmd_arg(i, arg, ...)`: Retrieves the i-th command-line argument.
    - `get_env_var(name, value, ...)`: Retrieves the value of an environment variable.
- **MPI Utilities**:
    - `mpi_id()`: Returns the current process's MPI rank.
    - `mpi_n_procs()`: Returns the total number of MPI processes.
    - `get_mpi_size_rank(comm, nproc, rank)`: Gets size and rank for a given communicator.
    - `abort_on_mpi_error(error_code, routine_name)`: Checks MPI error codes and aborts if an error occurred.

## Important Variables/Constants

- **`dp`, `qp`, `isp`, `idp`**: Parameters defining numerical precision for real and integer types.
- **`mainlog`, `errorlog`**: Pre-initialized `InOutput` objects for standard output and error streams.
- **`PRINT_ALWAYS`, `PRINT_NORMAL`, `PRINT_VERBOSE`, etc.**: Integer constants defining verbosity levels.
- **`NUMERICAL_ZERO` (Real(dp))**: A small constant representing zero for floating-point comparisons.
- **`RAN_MAX` (Integer)**: The maximum value returned by the `ran()` integer random number generator.
- **`EXEC_NAME` (Character)**: Stores the name of the currently running executable.
- **`COMMAND_ARG` (Character Array)**: Stores the command-line arguments passed to the program.
- **`SYSTEM_STRING_LENGTH`, `SYSTEM_STRING_LENGTH_LONG`**: Parameters defining buffer sizes for strings.

## Usage Examples

This module is used internally by nearly all parts of the libAtoms/QUIP framework. For example, `system_initialise` is typically one ofపంచtool_code
create_file_with_block
docs/System.md
# System.f95

## Overview

The `System.f95` file defines the `system_module`, a foundational utility module within the libAtoms/QUIP package. It provides a wide range of low-level system-dependent functionalities essential for the operation of the simulation software. These include robust input/output (I/O) management through the `InOutput` derived type, random number generation, CPU and wall-clock timing, string manipulation utilities, command-line argument parsing, access to environment variables, and basic MPI (Message Passing Interface) environment information. It also defines global constants for numerical precision and verbosity levels used throughout the codebase. This module does not directly manage physical system properties like simulation cell geometry or global temperature/pressure; those are handled by other modules like `Atoms.f95` and `DynamicalSystem.f95`.

## Key Components

- **`MODULE system_module`**: The main Fortran module.
- **`TYPE InOutput`**: A derived data type that abstracts file I/O operations. It can handle formatted or unformatted files, standard streams (stdout, stdin, stderr), and named files. Key associated routines:
    - `initialise(this, filename, ...)`: Opens/sets up an `InOutput` object.
    - `finalise(this)`: Closes an `InOutput` object.
    - `print(string, verbosity, file, ...)`: Overloaded subroutine to print various data types (strings, integers, reals, logicals) to an `InOutput` object, controlled by verbosity levels.
    - `read_line(this, status)`: Function to read a line of text.
    - `parse_line(this, delimiters, fields, ...)`: Subroutine to parse a line read from an `InOutput` object (now superseded by `split_string` or `split_string_simple`).
- **Precision Kind Parameters**:
    - `dp`: Double precision real kind.
    - `qp`: Quadruple precision real kind (availability may depend on compiler).
    - `isp`, `idp`: Integer kinds (likely standard and long integers).
- **Standard I/O Objects**:
    - `mainlog`: A public `InOutput` object, typically initialized to standard output (`stdout`). Default target for general messages.
    - `errorlog`: A public `InOutput` object, typically initialized to standard error (`stderr`). Default target for error messages.
    - `mpilog`: A public `InOutput` object for MPI-specific logging, often per-process.
- **Verbosity Level Parameters**: Integer constants like `PRINT_ALWAYS`, `PRINT_SILENT`, `PRINT_NORMAL`, `PRINT_VERBOSE`, `PRINT_NERD`, `PRINT_ANALYSIS` used to control the detail of output.
- **Core Subroutines**:
    - `system_initialise(...)`: Initializes MPI (if enabled), sets up random number generators, prepares `mainlog` and `errorlog`, and prints a welcome banner. Reads command-line arguments.
    - `system_finalise()`: Performs cleanup, including MPI finalization.
    - `system_timer(name, ...)`: A utility to measure elapsed CPU and wall clock time for specified code sections.
    - `system_abort(message)`: Terminates the program after printing an error message to `errorlog`.
- **Random Number Generators**:
    - `ran()`: Returns a random integer.
    - `ran_uniform()`: Returns a random real number uniformly distributed in [0,1].
    - `ran_normal()`: Returns a random real number from a Normal distribution (mean 0, stddev 1).
    - `system_set_random_seeds(seed)` / `system_reseed_rng(new_seed)`: Initialize or re-seed the random number generator.
- **String Utilities**:
    - `split_string(...)`, `split_string_simple(...)`: For tokenizing strings based on delimiters.
    - `upper_case(word)`, `lower_case(word)`: For case conversion.
    - `string_cat_*` (Operator `//` overloads): For concatenating strings with other data types (e.g., integers, reals) into a new string.
    - `a2s(array)`, `s2a(string)`: Convert between 1D character arrays and fixed-length strings.
- **Command Line and Environment**:
    - `cmd_arg_count()`: Returns the number of command-line arguments.
    - `get_cmd_arg(i, arg, ...)`: Retrieves the i-th command-line argument.
    - `get_env_var(name, value, ...)`: Retrieves the value of an environment variable.
- **MPI Utilities**:
    - `mpi_id()`: Returns the current process's MPI rank (0 if not MPI).
    - `mpi_n_procs()`: Returns the total number of MPI processes (1 if not MPI).
    - `get_mpi_size_rank(comm, nproc, rank)`: Gets size and rank for a given MPI communicator.
    - `abort_on_mpi_error(error_code, routine_name)`: Checks MPI error codes and aborts if an error occurred.

## Important Variables/Constants

- **`dp`, `qp`, `isp`, `idp`**: Parameters defining numerical precision for real and integer types used throughout the QUIP package.
- **`mainlog`, `errorlog`**: Pre-initialized `InOutput` objects providing convenient access to standard output and error streams.
- **`PRINT_ALWAYS`, `PRINT_NORMAL`, `PRINT_VERBOSE`, etc.**: Integer constants defining standard verbosity levels, used with the `print` routines to control output.
- **`NUMERICAL_ZERO` (Real(dp))**: A small constant (1.0e-14_dp) representing zero for floating-point comparisons.
- **`RAN_MAX` (Integer)**: The maximum value returned by the `ran()` integer random number generator (related to `huge(1)`).
- **`EXEC_NAME` (Character)**: Stores the name of the currently running executable, obtained from command-line arguments.
- **`COMMAND_ARG` (Character Array)**: Stores the command-line arguments passed to the program.
- **`SYSTEM_STRING_LENGTH`, `SYSTEM_STRING_LENGTH_LONG`**: Parameters defining default buffer sizes for strings (1024 and 102400 characters, respectively).
- **`quip_new_line` (Character)**: Stores the newline character, determined at runtime.

## Usage Examples

This module is initialized and used by main simulation programs and many other modules. For example, `system_initialise` is typically one of the first calls in any QUIP-based executable:

```fortran
program my_simulation
  use system_module
  implicit none

  call system_initialise(verbosity=PRINT_VERBOSE) ! Initialize with verbose output

  ! ... rest of the program ...
  call print_message('INFO', 'Simulation starting...')
  ! ...
  if (mpi_id() == 0) then
      call print('This message only from rank 0.')
  end if
  ! ...
  call system_finalise()
end program my_simulation
```
The `mainlog` and `errorlog` objects are then used implicitly or explicitly for output:
```fortran
call print('This goes to mainlog by default.')
call print('This error also to mainlog.', file=errorlog, verbosity=PRINT_ALWAYS)
```

## Dependencies and Interactions

- **`error_module`**: Heavily used for structured error handling via macros like `INIT_ERROR` and `RAISE_ERROR`. The `system_abort` routine is a key part of this.
- **`kind_module`**: Likely defines the actual integer and real kinds associated with `isp`, `idp`, `dp`, `qp`.
- **MPI library (via `mpi` module or `mpif.h`)**: If compiled with MPI support, this module initializes and finalizes the MPI environment and provides wrappers for basic MPI inquiry functions.
- **Almost all other modules in libAtoms/QUIP**: Due to its fundamental nature, `system_module` is a common dependency for logging, error reporting, random number needs, and accessing command-line or environment information. It does not directly depend on higher-level modules like `Atoms` or `DynamicalSystem` but provides them with essential services.The file `docs/System.md` has been created successfully with the content generated based on the template and the source file.

The subtask is now complete. I will submit the report.
