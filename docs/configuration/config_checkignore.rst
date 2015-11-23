.. _config_checkignore:

===================================
Exclude files/folders from analysis
===================================

To exclude files and/or folders from the analysis, add a **.checkignore** file to the root folder of your repository.

Files and folders that are mentioned in the **.checkignore** file will be ignored, once your project is re-analyzed.

Follow the following examples, to exclude files and/or folders:

.. code-block:: text

  test/*
  unittest/*
  my/subfolder/*
  demo.py
  test/demo.py
  **/test.py
  
.. warning:: Please do not add comments ('# ...') to your .checkignore file for now.

