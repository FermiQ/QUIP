# ParamReader.f95

## Overview

The `ParamReader.f95` file defines the `paramreader_module`, which is a utility for parsing and managing input parameters for simulations. It allows defining expected parameters with their types and default values, and then reading these parameters from input files (typically in a "key = value" format) or from command-line arguments. This module plays a crucial role in setting up and configuring simulations by providing a structured way to handle user inputs.

## Key Components

- **`MODULE paramreader_module`**: The main Fortran module encapsulating all parameter reading functionalities.
- **`param_register` (Interface)**: A collection of subroutines (`param_register_single_integer`, `param_register_single_real`, `param_register_single_string`, `param_register_single_logical`, `param_register_multiple_integer`, `param_register_multiple_real`, `param_register_dontread`) used to declare parameters that the program expects. When registering, one provides:
    - A `Dictionary` object to store parameter definitions.
    - A `key` (string) for the parameter name.
    - A default `value` (string). The special value `PARAM_MANDATORY` indicates the parameter must be supplied by the user.
    - A target Fortran variable (scalar or array) where the parsed value will be stored.
    - A `help_string` describing the parameter.
    - Optionally, a logical target variable (`has_value_target`) that becomes true if the parameter is explicitly set.
    - Optionally, an `altkey` for an alternative name for the parameter.
- **`param_read_line(dict, myline, ...)`**: Function that parses a single input `myline` (string) for "key = value" pairs and updates the provided `Dictionary` (`dict`).
- **`param_read_file(dict, file, ...)`**: Function that reads an entire input `file` (an `Inoutput` object), processing each line (skipping comments and empty lines) using `param_read_line`.
- **`param_read_args(dict, args, ...)`**: Function that parses command-line arguments for "key = value" pairs. It can process all arguments or a specified subset.
- **`param_check(dict, missing_keys)`**: Function to verify that all parameters registered as `PARAM_MANDATORY` have been assigned a value. It can return a list of missing keys.
- **`param_print_help(dict, ...)`**: Subroutine to display help information for all registered parameters, including their descriptions and default values.
- **`param_print(dict, ...)`**: Subroutine to print the current values of all registered parameters.
- **`param_write_string(dict)`**: Function that converts the current state of the parameter dictionary back into a single string (e.g., "key1=value1 key2='quoted value'").
- **`TYPE ParamEntry`**: An internal derived data type used to store details about each registered parameter, such as its string value, type (integer, real, string, logical), expected number of values, pointers to the target Fortran variables, and the help string.

## Important Variables/Constants

- **`PARAM_MANDATORY` (Character Parameter)**: A string constant ("//MANDATORY//"). If used as the default value during parameter registration, it signifies that the parameter must be provided by the user.
- **`PARAM_REAL`, `PARAM_INTEGER`, `PARAM_STRING`, `PARAM_LOGICAL` (Integer Parameters)**: Constants defining the data type of a registered parameter.
- **`PARAM_NO_VALUE` (Integer Parameter)**: A special parameter type that is registered but not parsed for a value (e.g., flags or section headers).
- **`MAX_N_FIELDS` (Integer Parameter)**: Defines the maximum number of fields (space-separated values) expected on a single line when parsing multi-value parameters. Default is 1024.

## Usage Examples

This module is used internally by main programs like `quip.f95` or `md.f95` to read simulation setup files. A typical workflow within a program would be:

1.  Initialize a `Dictionary` object.
2.  Register all expected parameters using `param_register`, providing keys, default values (or `PARAM_MANDATORY`), target variables, and help strings.
    ```fortran
    type(Dictionary) :: params_dict
    real(dp) :: temperature, timestep
    integer :: num_steps
    character(len=STRING_LENGTH) :: potential_file

    call initialise(params_dict)
    call param_register(params_dict, 'temperature', '300.0', temperature, 'Target temperature in Kelvin.')
    call param_register(params_dict, 'timestep', '1.0', timestep, 'MD timestep in fs.')
    call param_register(params_dict, 'num_steps', '1000', num_steps, 'Number of MD steps.')
    call param_register(params_dict, 'potential_file', PARAM_MANDATORY, potential_file, 'Path to potential file.')
    ```
3.  Read parameters from a configuration file using `param_read_file`.
    ```fortran
    type(Inoutput) :: param_file
    call fopen(param_file, 'input.par', 'r')
    if (.not. param_read_file(params_dict, param_file)) then
        ! Handle error
    end if
    call fclose(param_file)
    ```
4.  Optionally, override parameters with command-line arguments using `param_read_args`.
    ```fortran
    if (.not. param_read_args(params_dict)) then
        ! Handle error
    end if
    ```
5.  Check for mandatory parameters using `param_check`.
    ```fortran
    if (.not. param_check(params_dict)) then
        call param_print_help(params_dict)
        call system_abort('Missing mandatory parameters.')
    end if
    ```
6.  The target variables (`temperature`, `timestep`, etc.) will now hold the parsed values.

## Dependencies and Interactions

- **`dictionary_module`**: This is a primary dependency. The `ParamReader` uses a `Dictionary` object to store `ParamEntry` types, effectively managing the schema and current values of all known parameters.
- **`system_module`**: For general system utilities, like `system_abort` and string conversion functions (e.g., `String_to_Int`, `String_to_Real`).
- **`extendable_str_module`**: For advanced string manipulation capabilities, likely used in parsing lines.
- **`table_module`**: May be used for internal parsing tasks, such as splitting strings into fields.
- **Application Code (e.g., `quip.f95`, `md.f95`)**: The main simulation programs use `ParamReader` to define and load their operational parameters.
- **`Atoms_module`, `DynamicalSystem_module`, Potential Modules**: The values read by `ParamReader` are used to configure objects and settings in these and other modules (e.g., setting up atom configurations, simulation timestep, potential parameters).
- **File System**: Interacts with the file system to read parameter files.
- **Command Line Interface**: Interacts with the command line to parse arguments.
