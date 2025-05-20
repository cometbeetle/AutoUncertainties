Contribution
============


CI and Unit Testing
-------------------

Development of AutoUncertainties relies on a series of unit tests located in the ``tests`` directory. These
are automatically run using GitHub actions when commits are pushed to the repository. To run the tests
manually, first install the package with testing capabilities:

.. code-block:: bash

   pip install -e .[CI]
   coverage run -m pytest --cov --cov-report=term --ignore=tests/pandas


At the moment, it makes sense to disable the Pandas tests until certain features are finalized.


Documentation
-------------

To build the documentation locally, clone the repository, create a virtual Python environment 
(if desired), and run the following commands within the repository directory:

.. code:: bash

   pip install -e .[docs]
   sphinx-build docs/source docs/build
