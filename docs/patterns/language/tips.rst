==============
Tips and tricks
==============

Document your queries
====================

YAML provides the possibility to include comments. So you should use this opportunity in order
to make your queries more readable.

.. code-block:: yaml

  node_type: name # only match name
  id:
    $regex: ^test|bar$ # test and bar are almost never good variable names

Concise YAML representation
===========================

A pattern can get long quite easily. Luckily, YAML provides a dense syntax which can improve the readability of your patterns.
Instead of

.. code-block:: yaml

  # this pattern checks for some blacklisted variable names
  node_type: name
  id:
    $or:
      - foo
      - bar
      - baz

you could use the more dense notation

.. code-block:: yaml

  # this pattern checks for some blacklisted variable names
  node_type: name
  id: {$or: [foo, bar, baz]}

Setting the exact location where your pattern matches
=====================================================

Let us now assume, you are trying to match variable names which do not follow the Python naming conventions.
The following Cody pattern finds all assignments to such variables

.. code-block:: yaml

  node_type: assign
  targets:
    node_type: name
    id:
      $not:
        $regex: '[a-z_][a-z0-9_]*$|(([A-Z_][A-Z0-9_]*)|(__.*__))$'

This pattern correctly finds all assignments to "non-Pythonic" variable names.
But it always marks the whole assignment as errorneous. We can do better by only
marking the `name` node as deficient. We can do this by storing the corresponding
`name` node in the magic variable ``result``:

.. code-block:: yaml

  node_type: assign
  targets:
    $store:
      node_type: name
      id:
        $not:
          $regex: '[a-z_][a-z0-9_]*$|(([A-Z_][A-Z0-9_]*)|(__.*__))$'
    name: result

Pretty occurrence descriptions
==============================

You can generate more concise occurrence descriptions by providing some context within these messages.

For example the message

.. code-block:: plain

  The variable name 'ODD_variableName' does not obey the Python naming conventions.

is more human-friendly than just

.. code-block:: plain

  The variable name does not obey the Python naming conventions.

In order to provide this context in your error messages, you need to take two simple steps:

1. The Cody pattern must save all the strings which will later be neccessary for the message in a variable with
   a name starting with `data_`.
2. You have to reference the stored data from your occurrence descriptions.

The first part is easy:

.. code-block:: yaml

  node_type: assign
  targets:
    $store:
      node_type: name
      id:
        $and:
          - $not:
              $regex: '[a-z_][a-z0-9_]*$|(([A-Z_][A-Z0-9_]*)|(__.*__))$'
          - $store_as: data_varname
    name: result

The second part is quite easy as well: You just have to embed a placeholder in your occurrence description.
The syntax used for this is the Python ``printf`` syntax. This syntax provides quite a lot of possibilities but
most of the time you will only need string placeholders. In order to get the nifty error message for our
badly chosen variable name, we can use the template

.. code-block: plain

  The variable name '%(occurrence.data.varname)s' does not obey the Python naming conventions.

The part enclosed by the brackets specifies the variable which should be put in this place. Currently the following
variables are available:

* ``occurrence.data.<var_name>`` always contains the content which you stored in the corresponding Cody variable.
