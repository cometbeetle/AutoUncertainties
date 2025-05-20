
# AutoUncertainties

AutoUncertainties is a package that makes handling linear uncertainty propagation for scientific applications 
straightforward and automatic using auto-differentiation.

* View the [full documentation here](https://autouncertainties.readthedocs.io/en/latest/). 

## Supported Features

- [x] Scalars
- [x] Arrays, with support for most NumPy ufuncs and functions
- [ ] Pandas Extension Type (see [here](https://pandas.pydata.org/docs/reference/api/pandas.api.extensions.ExtensionDtype.html))

## Prerequisites

For array support:

* `jax`
* `jaxlib`
* `numpy`

## Installation

To install, simply run:

```
pip install auto-uncertainties
```

## Build Documentation

To build the documentation locally, clone the repository, create a virtual Python environment 
(if desired), and run the following commands within the repository directory:

```bash
pip install -e .[docs]
sphinx-build docs/source docs/build
```

Once built, the docs can be found under the `docs/build` subdirectory.

## CI and Unit Testing

Development of AutoUncertainties relies on a series of unit tests located in the `tests` directory. These
are automatically run using GitHub actions when commits are pushed to the repository. To run the tests
manually, first install the package with testing capabilities:

```bash
pip install -e .[CI]
coverage run -m pytest --cov --cov-report=term --ignore=tests/pandas
```

At the moment, it makes sense to disable the Pandas tests until certain features are finalized.


## Basic Usage

* Creating a scalar `Uncertainty` variable is relatively simple:

  ```python
  >>> from auto_uncertainties import Uncertainty
  >>> value = 1.0
  >>> error = 0.1
  >>> u = Uncertainty(value, error)
  >>> u
  1 +/- 0.1
  ```
  
  As is creating a NumPy array of Uncertainties:

  ```python
  >>> from auto_uncertainties import Uncertainty
  >>> import numpy as np
  >>> value = np.linspace(start=0, stop=10, num=5)
  >>> error = np.ones_like(value)*0.1
  >>> u = Uncertainty(value, error)
  ```

  The `Uncertainty` class automatically determines which methods should be implemented based on 
  whether it represents a vector uncertainty, or a scalar uncertainty. When instantiated with a
  sequence or `NumPy` array, vector-based operations are enabled; when instantiated with scalars,
  only scalar operations are permitted. 

* Scalar uncertainties implement all mathematical and logical 
  [dunder methods](https://docs.python.org/3/reference/datamodel.html#object.__repr__>) explicitly using linear
  uncertainty propagation.

  ```python
  >>> from auto_uncertainties import Uncertainty
  >>> u = Uncertainty(10.0, 3.0)
  >>> v = Uncertainty(20.0, 4.0)
  >>> u + v
  30 +/- 5
  ```

* Array uncertainties implement a large subset of the NumPy ufuncs and methods using `jax.grad` or 
  `jax.jacfwd`, depending on the output shape.

  ```python
  >>> from auto_uncertainties import Uncertainty
  >>> import numpy as np
  >>> value = np.linspace(start=0, stop=10, num=5)
  >>> error = np.ones_like(value)*0.1
  >>> u = Uncertainty(value, error)
  >>> np.exp(u)
  [1 +/- 0.1, 12.1825 +/- 1.21825, 148.413 +/- 14.8413, 1808.04 +/- 180.804, 22026.5 +/- 2202.65]
  >>> np.sum(u)
  25 +/- 0.223607
  >>> u.sum()
  25 +/- 0.223607
  >>> np.sqrt(np.sum(error**2))
  0.223606797749979
  ```

* The central value, uncertainty, and relative error are available as attributes:

  ```python
  >>> from auto_uncertainties import Uncertainty
  >>> u = Uncertainty(10.0, 3.0)
  >>> u.value
  10.0
  >>> u.error
  3.0
  >>> u.rel
  0.3
  ```

* To strip central values and uncertainty from arbitrary variables, accessor functions `nominal_values`
  and `std_devs` are provided:

  ```python
  >>> from auto_uncertainties import nominal_values, std_devs
  >>> u = Uncertainty(10.0, 3.0)
  >>> v = 5.0
  >>> nominal_values(u)
  10.0
  >>> std_devs(u)
  3.0
  >>> nominal_values(v)
  5.0
  >>> std_devs(v)
  0.0
  ```

* Displayed values are automatically rounded according to the `g` format specifier. To enable
  rounding consistent with the Particle Data Group (PDG) standard, the `set_display_rounding` 
  function can be called as follows:

  ```python
  >>> from auto_uncertainties import Uncertainty, set_display_rounding
  >>> import numpy as np
  >>> value = np.linspace(start=0, stop=10, num=5)
  >>> error = np.ones_like(value)*0.1
  >>> u = Uncertainty(value, error)
  >>> set_display_rounding(True)   # enable PDG rules
  >>> np.sum(u)
  25.0 +/- 0.22
  >>> set_display_rounding(False)  # default behavior
  >>> np.sum(u)
  25 +/- 0.223607
  ```

  If enabled, the PDG rounding rules will, in general, cause `Uncertainty` objects to be displayed with:
    - Error to 2 significant digits.
    - Central value to first signficant digit of error, or two significant figures (whichever is more 
      significant digits).

* If `numpy.array` is called on an `Uncertainty` object, it will automatically get cast down to a 
  numpy array (losing all uncertainty information!), and emit a warning. To force an exception to be raised
  instead, use `set_downcast_error`:

  ```python
  >>> from auto_uncertainties import Uncertainty, set_downcast_error
  >>> import numpy as np
  >>> set_downcast_error(True)
  >>> value = np.linspace(start=0, stop=10, num=5)
  >>> error = np.ones_like(value)*0.1
  >>> u = Uncertainty(value, error)
  >>> np.array(u)
  Traceback (most recent call last):
      ...
  auto_uncertainties.exceptions.DowncastError: The uncertainty is stripped when downcasting to ndarray.
  ```
  
## Current Limitations and Future Work

### Dependent Random Variables

To simplify operations on `Uncertainty` objects, `AutoUncertainties` assumes all variables are independent and 
normally distributed. This means that, in the case where a user assumes dependence between two or more `Uncertainty` 
objects, unexpected and counter-intuitive behavior may arise during uncertainty propagation. This is a common 
pitfall when working with `Uncertainty` objects, especially since the package will not prevent you from 
manipulating variables in a manner that implies dependence.

* **Subtracting Equivalent Uncertainties**

  Subtracting an `Uncertainty` from itself will not result in a standard deviation of zero:

  ```python
  x = Uncertainty(5.0, 0.5)
  print(x - x)  # 0 +/- 0.707107
  ```

* **Mean Error Propagation**

  When multiplying a vector by a scalar `Uncertainty` object, each component of the resulting vector
  is assumed to be a multivariate normal distribution with no covariance,
  which may not be the desired behavior. For instance, taking the mean of such a
  vector will return an `Uncertainty` object with an unexpectedly small standard deviation.

  ```python
  u = Uncertainty(5.0, 0.5)
  arr = np.ones(10) * 10
  result = np.mean(u * arr)  # 50 +/- 1.58114, rather than 50 +/- 5 as expected
  ```
  
  To obtain the uncertainty corresponding to the case where each element of the array is fully correlated,
  two workaround techniques can be used:

  1. Separate the central value from the relative error, multiply the vector by the central value, take the mean
     of the resulting vector, and then multiply by the previously stored relative error.

     ```python
     u = Uncertainty(5.0, 0.5)
     scale_error = Uncertainty(1, u.relative)  # collect relative error
     scale_value = u.value                     # collect central value

     arr = np.ones(10) * 10
     result = np.mean(scale_value * arr) * scale_error  # 50 +/- 5
     ```

  2. Take the mean of the vector, and then multiply by the `Uncertainty`:

     ```python
     u = Uncertainty(5.0, 0.5)
     arr = np.ones(10) * 10
     result = u * np.mean(arr)  # 50 +/- 5
     ```

These workarounds are nevertheless cumbersome, and cause `AutoUncertainties` to fall somewhat short of the original
goals of automated error propagation. In principle, this could be addressed by storing a full computational
graph of the result of chained operations, similar to what is done in `uncertainties`. However, the complexity
of such a system places it out of scope for `AutoUncertainties` at this time.


## Inspirations

The class structure of `Uncertainty` and the `NumPy` ufunc implementation is heavily inspired by the 
excellent package [Pint](https://github.com/hgrecco/pint).
