.. _config_pull_request_status:

=============================
Configure pull request status
=============================

.. note:: This feature is only available for GitHub users.

Once you sign up, QuantifiedCode automatically installs a pull request hook on GitHub (assuming you've granted us all necessary access rights). Going forward, QuantifiedCode's analysis will be triggered whenever a pull request is opened (also via forks).

To make sure your pull request *fails* or *passes* for issues that you and your team consider important, follow one of the following two strategies:

* Upfront configuration: :ref:`Disable checks <config_disable_checks>` when you set up your project.
* Incremental disablement: :ref:`Disable checks <config_disable_checks>` whenever you see them in a failed pull request.

