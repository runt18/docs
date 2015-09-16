===============
Query Operators
===============

`Cody` provides a range of query operators to facilitate pattern matching on the syntax tree.
In the following a query can be understood as a code check for a certain pattern in the source code.


* :ref:`basics`
* :ref:`references`
* :ref:`boolean`
* :ref:`repetition`
* :ref:`matching`
* :ref:`structure`
* :ref:`regex`
* :ref:`matching`

.. _basics:

Basics
======

For advanced pattern matching, you can make use of special operators in your query.
These always start with a dollar (**$**) sign to distinguish them from normal Cody fields.
The value you pass to the operator should again be a valid pattern dictionary, or for some operators,
a list containing arguments or a simple string. On the same level as the operator, additional arguments might be specified.
An example: The pattern

.. code-block:: yaml

    $store:
      node_type: functiondef
    name : my_function

will call the ``$store`` operator with the dictionary ``node_type: functiondef`` in the variable ``my_function``.
The ``$store`` operator will store a node with ``node_type = functiondef`` in the variable ``my_function``.

Some operators do not require a pattern dictionary and will instead accept arguments.
These **argument-only** are marked in the list below. An example is the ``$ref`` operator,
which you can invoke like this:

.. code-block:: yaml

    $ref:
      name: my_function

This will fetch the tree, i.e. the already matched source code, which is stored in the variable ``my_function``.

Special Cases
-------------

The default behaviour of the matching algorithm can be adjusted with some special arguments,
which you can provide using the special **$** field like this:

.. code-block:: yaml

  $ : {match_first : foo}
  foo: {$store : {$anything : {}},name: foo}
  baz: {$ref : {name: foo}}

Here, we specify that the `foo` branch should get matched first, since the value that we generate
there will be used in the `bar` branch below.

.. _references:

Storing and Retrieving Sub-Patterns
===================================

Often you want to store a part of a tree and reuse it somewhere else in your query.
Let us start with the following example:

.. code-block:: python

  my_variable = my_variable

Obviously, this assignment is pointless. Hence we want to find all such assignments
in our code base. Therefore, we first take a look at the corresponding AST:

.. code-block:: yaml

  node_type: assign
  targets:
    - node_type: name
      id: my_variable
      ctx:
        node_type: store
  value:
    node_type: name
    id: my_variable
    ctx:
      node_type: load

The pattern which is able to match these assignments is:

.. code-block:: yaml

  node_type: assign
  $ : {match_first : targets}
  targets:
    - node_type: name
      id:
        $store_as: varname
  value:
    node_type: name
    id:
      $ref:
        name: varname

It is interpreted in the following manner:

1. Search for all AST nodes with `node_type = assign`
2. Have a look at the `targets` child: check that its `node_type` is `name` and then store its `id` in the variable called `varname`.
3. Have a look at the `value` child: make sure that its `node_type` is `name` and that its `id` matches the previously stored variable name.

$store_as
--------

Stores the subtree, which is defined at this point of the source code in a variable. The stored content can be accessed again, by using the $ref operator.

.. code-block:: yaml

  node_type: name
  id:
    $store_as: variable_name

It should be noted that the following ``$store`` operator is usually more reliable and less
prone to errors than ``$store_as``.

$store
------

This operator is quite similar to $store_as, but it additionally checks if the stored subtree matches a given expression.

The pattern

.. code-block:: yaml

  $store:
    node_type: functiondef
  name: my_function

is just an abbreviation for

.. code-block:: yaml

  $and:
    - node_type: functiondef
    - $store_as: my_function


$ref
----

Retrieves the stored value from a variable.

.. code-block:: yaml

  node_type: name
  id:
    $ref:
      name: stored_function_name


In case one would like to retrieve a specific value from the entire stored dictionary of nodes it is possible to do the following.
For example a node of the node type: `name` has been stored as: `some_name` but we only need to reference the value also called branch: `id`.

.. code-block:: yaml

  node_type: functiondef
  name:
    $ref:
      name: some_name.id

.. _boolean:

Boolean Operators
=================

$or
---

.. code-block:: yaml

  $or:
    -  node_type: functiondef
       id: my_function
    -  node_type: classdef
       id: my_class

$and
----

.. code-block:: yaml

  $and:
    -  node_type: {$ref: my_ref_1}
    -  id: {$ref: my_ref_2}

$not
----

.. code-block:: yaml

  $not:
    node_type: functiondef

One important thing to note at this point is the difference between:

.. code-block:: yaml

  $not:
    node_type: functiondef
    name: foo

and:

.. code-block:: yaml

  node_type: functiondef
  name:
    $not: foo

The first pattern will match if the code contains a function definition with
a name other than ``foo`` but will also match if no node type named ``functiondef`` is present at all.
The latter will only match if a function definition is found with a name other than ``foo``.

.. _repetition:

Repetition and Concatenation
============================

$concat
-------

Concatenate a series of expressions.

Example:

.. code-block:: yaml

  $concat:
    - node_type: functiondef
    - node_type: classdef

Matches a function definition followed by a class definition.

In most situations there is no need to use this operator explicitly,
since lists are implicitly matched using the `$concat` operator.

$repeat
-------

Repeat a given pattern a certain number of times.

Example:

.. code-block:: yaml

  $repeat:
    node_type: assign
  min: 1
  max: 4

Matches a series of 1 to four assign statements.

Parameters:

* **min**: Minimum number of matches. Default: None (will match zero or more occurences)
* **max**: Maximum number of matches. Default: None (will match zero or more occurences)
* **greedy**: Whether the operator should be *greedy*. If set to `True`, it will first match
              expressions with the highest number of repetitions, otherwise with the smallest.

.. _matching:

Matching Nodes
==============

$anything
---------

Will match, well, anything. Use this if you just want to check that a given element is present in
the tree but you don't care what kind of element it actually is.

For example lets say we are looking for a function definition which has a ``pass`` directly before
any other kind of python command. Our code check would then look something like this:

.. code-block:: yaml

  node_type: functiondef
  body:
    - node_type: pass
    - $anything: {}

$empty
------

Will match if the given element in the tree is empty, i.e. if it contains either an empty
dictionary or an empty list.

Now we could check if a function definition which does not have any decorators
exists somewhere in our source code:

.. code-block:: yaml

  node_type: functiondef
  decorator_list: {$empty: {}}

.. _structure:

Position within the Tree
========================

$first
------

Matches the first element in the current list of elements.

For example this could be used to store the first argument of a function for further analysis:

.. code-block:: yaml

  node_type: functiondef
  args:
    name: first_argument
    $store:
      args: {$first: {}}

Here one should mention that in a lot of cases it is sufficient to simply write the condition for the
first element of a list in the form of a list. The matching algorithm will now only find a match
if the first element of a list meets the specified conditions.
For example: We want find all function definitions with a first argument other than ``self``:

.. code-block:: yaml

  node_type: functiondef
  args:
    args:
    - node_type: name
      id: {$not: self}

$last
-----

Matches the last element in the current list of elements.

Similar to the previous example we could now store the last argument of a function definition.

.. code-block:: yaml

  node_type: functiondef
  args:
    name: last_argument
    $store:
      args: {$last: {}}

$parent
-------

Provides a reference to the parent element of the current element.

.. warning:: This works only for nodes whose parent node has been actively traversed during the
             matching operation, and will thus NOT work for the nodes that are in the initial
             list of nodes passed to the regular expression.

.. _contains:

$contains
---------

Produces a match if a certain list contains a given pattern dictionary as element.

For example we could find out whether someone defined a function inside an if condition:

.. code-block:: yaml

  node_type: if
  body:
    $contains:
      node_type: functiondef

$anywhere
---------

Matches a given expression anywhere in the tree. This operator will recursively descend into the
current node tree and try to produce a match with the given expression.

.. warning:: This operator can be expensive when matching large trees, use with care.

Parameters:

* **limit**: Limits the depth up to which the tree will be searched
* **exclude**: All nodes which match the provided expression will not be inspected during the search

This operator is tempting to use because one does not have to worry about the exact location of
the pattern that we want to find.
For example: It would be easy to find out if the ``global`` statement has been used anywhere in our
source code.

.. code-block:: yaml

  $anywhere:
    node_type: global

However a pattern like this should be avoided whenever possible because the matching algorithm would
have to step through every single node of the entire source code and check for each step whether
the node matches our definition. This could take quite some time.

$length
-------

Matches a node with a defined number of elements in one of its parameters.

For example if you would like to match an if statement with exactly one element in its body, you can use:

.. code-block:: yaml

  node_type: if
  body:
    $length: 1

.. _regex:

Regex Matching
==============

$regex
------

Matches the value of a given field against a regular expression.

Parameters:

* **modifiers**: a string containing modifiers for the regular expression.

Supported modifiers:

* ``i``: match the regular expression case-insensitive
* ``m``: enables multiline matching

For example we could match if the assigned name of some object ends with a number:

.. code-block:: yaml

  node_type: assigment
  targets:
    $contains:
      node_type: name
      id: {$regex: ".*[0-9]$"}

This would match the following python code:

.. code-block:: python

  bar2 = 4

But also this:

.. code-block:: python

  foo, bar2 = function_that_returns_two_values()

If we would not have used the ``$contains`` operator the second code block would not have been matched.

Another example:

.. code-block:: yaml

  node_type: functiondef
  name:
    $regex: foo|bar
    modifiers: i

This will match any `node_type`: `name` with an `id` which is either `foo` or `bar`.
