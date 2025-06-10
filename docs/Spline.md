# Spline.f95

## Overview

The `Spline.f95` file defines the `spline_module`, which implements cubic spline interpolation. Splines are used to create smooth, continuous functions that pass through a given set of data points (knots). This module provides tools to initialize a spline from these points and then evaluate the interpolated function value and its derivative at any arbitrary point. This is particularly useful for representing tabulated functions, such as interatomic potentials or other physical quantities where an analytical form is unavailable or computationally expensive.

## Key Components

- **`MODULE spline_module`**: The main Fortran module for spline operations.
- **`TYPE Spline`**: A derived data type that encapsulates all information defining a cubic spline. Its members include:
    - `n`: Integer, the number of knot points.
    - `x(:)`: Real(dp), allocatable array storing the x-coordinates of the knot points.
    - `y(:)`: Real(dp), allocatable array storing the y-coordinates (function values) at the knot points.
    - `y2(:)`: Real(dp), allocatable array storing the second derivatives of the spline at each knot point. These are calculated internally.
    - `yp1`: Real(dp), the first derivative of the spline at the first knot point `x(1)`.
    - `ypn`: Real(dp), the first derivative of the spline at the last knot point `x(n)`.
    - `y2_initialised`: Logical flag, true if the second derivatives `y2` have been computed.
    - `initialised`: Logical flag, true if the spline object has been properly set up.

- **Core Subroutines and Functions**:
    - **`initialise(this, x, y, yp1, ypn)` (or `spline_init`)**: Subroutine to set up a `Spline` object. It takes the knot coordinates `x`, corresponding function values `y`, and the first derivatives `yp1` and `ypn` at the endpoints as boundary conditions. The input `x` and `y` arrays are sorted by `x` values. This routine then calls `spline_y2calc` to compute the necessary second derivatives.
    - **`finalise(this)` (or `spline_finalise`)**: Subroutine to deallocate the dynamically allocated arrays (`x`, `y`, `y2`) within the `Spline` object.
    - **`spline_y2calc(this)`**: An internal subroutine that calculates the second derivatives (`y2`) at each knot point. This is a crucial step for cubic spline interpolation. It can handle "natural spline" conditions (zero second derivative at an endpoint) if the corresponding `yp1` or `ypn` is set to a value greater than `0.99e30_dp`.
    - **`spline_value(this, x)`**: Function that returns the interpolated y-value of the spline for a given x-coordinate. If `x` is outside the range of the knot points, linear extrapolation is performed using the endpoint derivatives (`yp1`, `ypn`).
    - **`spline_deriv(this, x)`**: Function that returns the first derivative (dy/dx) of the spline at a given x-coordinate. Similar to `spline_value`, it uses linear extrapolation for points outside the knot range.
    - **`min_knot(this)` / `max_knot(this)`**: Functions to retrieve the minimum and maximum x-values of the defined knot points.
    - **`print(this, file)`**: Subroutine to print the details of the spline object, including knot points and second derivatives.
    - **`spline_compute_matrices(this, y2_matrix1, y2_matrix0)`**: An alternative (less commonly used directly) method to formulate the calculation of second derivatives in terms of matrix operations (`Y'' = A_inv * B * Y + A_inv * C`).

## Important Variables/Constants

- **Within `TYPE Spline`**:
    - `x(:)`, `y(:)`: The input data points.
    - `y2(:)`: The internally computed second derivatives, essential for interpolation.
    - `yp1`, `ypn`: User-supplied first derivatives at the endpoints, which define the boundary conditions for the spline.
- **Sentinel Value for Natural Splines**: A value greater than `0.99e30_dp` for `yp1` or `ypn` signals that a "natural spline" boundary condition (zero second derivative) should be used at that endpoint.

## Usage Examples

This module is used to create and evaluate smooth interpolating functions, for example, in tabular potentials or when fitting data.

```fortran
use spline_module
use system_module ! For dp and print

type(Spline) :: my_potential_spline
real(dp), dimension(5) :: r_values = (/ 1.0_dp, 1.5_dp, 2.0_dp, 2.5_dp, 3.0_dp /)
real(dp), dimension(5) :: energy_values = (/ 0.5_dp, -0.8_dp, -1.0_dp, -0.5_dp, 0.1_dp /)
real(dp) :: yp1_val, ypn_val ! Endpoint derivatives (e.g., from analytical form or set for natural spline)
real(dp) :: r_test, interpolated_energy, interpolated_force

! Assume yp1_val and ypn_val are known (e.g., force is zero at large distances)
yp1_val = 0.1_dp  ! Example: derivative at r_values(1)
ypn_val = 0.01_dp ! Example: derivative at r_values(5)

! Initialize the spline
call initialise(my_potential_spline, r_values, energy_values, yp1_val, ypn_val)

! Evaluate the spline at a new point
r_test = 1.75_dp
interpolated_energy = spline_value(my_potential_spline, r_test)
interpolated_force = -spline_deriv(my_potential_spline, r_test) ! Force = -d(Energy)/dr

call print("At r = ", r_test, ", Energy = ", interpolated_energy, ", Force = ", interpolated_force)

! Clean up
call finalise(my_potential_spline)
```

## Dependencies and Interactions

- **`system_module`**: Used for fundamental type definitions (like `dp` for double precision), the `Inoutput` type for printing, the `print` subroutine, `optional_default` for handling optional arguments, and `system_abort` for error handling.
- **`linearalgebra_module`**: Provides array sorting (`sort_array`) used during spline initialization. The `spline_compute_matrices` routine also uses matrix operations like inversion (`inverse`) and multiplication (`.mult.`), implying a deeper use of this module if that specific calculation path is taken.
- **`table_module`**: While not directly evident in the core spline logic, it might be used by other parts of the code to prepare or manage the `x` and `y` data tables before they are passed to the spline initialization.

**Interactions with other modules:**
- **Potential Modules (e.g., in `src/Potentials/`)**: A primary consumer of the `spline_module`. Many interatomic potentials are defined by tabulated values (e.g., pair potentials as a function of distance, or parts of embedded atom method potentials like electron density functions). Splines allow these potentials and their derivatives (forces) to be evaluated smoothly and efficiently.
- **Analysis Tools**: Any tool that needs to interpolate experimental or simulation data can use this module.
- **Data Fitting Routines**: If the simulation package includes curve fitting capabilities, splines can be a target function type.
