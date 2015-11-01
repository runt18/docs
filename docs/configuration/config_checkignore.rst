.. _config_checkignore:

===================================
Exclude files/folders from analysis
===================================

To exclude files and/or folders from the analysis, add a **.checkignore** file to the root folder of your repository.

Files and folders that are mentioned in the **.checkignore** file will be ignored, once your project is re-analyzed.

Follow the following examples, to exclude files and/or folders:

.. code-block:: text

  # .checkignore example
  # Exclude entire folders
  test/*
  unittest/*
  my/subfolder/*
  
  # Exclude demo.py files
  demo.py
  test/demo.py

