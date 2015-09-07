===============
Query Operators
===============

`Cody` provides a range of query operators to facilitate pattern matching on the syntax tree.


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
These always start with a dollar (**$**) sign to distinguish them from normal query field. 
The value you pass to the operator should again be a valid pattern dict, or for some operators, 
a dictionary containing arguments. On the same level as the operator, additional arguments might be specified.
An example: The pattern

.. code-block:: yaml

    $store:
      node_type: functiondef
    name : my_function

will call the `$store<store>` operator with the the sub-dictionary ``node_type: functiondef`` in the variable `my_function`.
The $store operator will store a node with `node_type = functiondef` in the variable `my_function`.

Some operators do not require a pattern dict and will instead accept arguments. 
These **argument-only** are marked in the list below. An example is the `ref` operator,
which you can invoke like this:``node_type: functiondef``

.. code-block:: yaml

    $ref:
      name: my_function

This will fetch the tree stored in the variable `my_function`.

Special Cases
-------------

The normal matching operator also accepts some special arguments, which you can provide using the
special **$** field like this:

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
2. Have a look at the `targets` child: check that its `node_type` is `name` and then store its `id` in the variable called ``varname``.
3. Have a look at the `value` child: make sure that its `node_type` is `name` and that its `id` matches the previously stored variable name.

$store_as
------

Stores the subtree in a variable. The stored content can be accessed again, by using the $ref operator.

.. code-block:: yaml

  node_type: name
  id:
    $store_as: variable_name

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

.. _repetition:

One important thing to note at this point is the difference between:

.. code-block:: yaml

  $not:
    node_type: functiondef
    name: foo

.. _repetition:

and:

.. code-block:: yaml

  node_type: functiondef
  name:
    $not: foo

.. _repetition: .

The first pattern will match if the code contains a function definition with
a name other than ``foo`` but will also match if no node type named ``functiondef`` is present at all.
The latter will only match if a function definition is found with a name other than ``foo``.



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

In most situations there is no need to use this operator explicitely,
since lists are implicitely matched using the `$concat` operator.

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

$empty
------

Will match if the given element in the tree is empty, i.e. if it contains either an empty
dictionary or an empty list.

.. _structure:

Position within the Tree
========================

$first
------

Matches the first element in the current list of elements.

$last
-----

Matches the last element in the current list of elements.

$parent
-------

Provides a reference to the parent element of the current element.

.. warning:: This works only for nodes whose parent node has been actively traversed during the 
             matching operation, and will thus NOT work for the nodes that are in the initial
             list of nodes passed to the regular expression.

.. _contains:

$contains
---------

Matches a node that contains a given tree somewhere in its children.

$anywhere
---------

Matches a given expression anywhere in the tree. This operator will recursively descend into the
current node tree and try to produce a match with the given expression.

.. warning:: This operator can be expensive when matching large trees, use with care.

Parameters:

* **limit**: Limits the depth up to which the tree will be searched
* **exclude**: All nodes which match the provided expression will not be inspected during the search

.. _regex:

$length
-------

Matches a node with a defined number of elements in one of its parameters.  
For example if you would like to match an if statement with exactly one element in its body, you can use:

.. code-block:: yaml

  node_type: if
  body:
    $length: 1


Regex Matching
==============

$regex
------

Matches the value of a given field against a regular expression.

Parameters:

* **modifiers**: a string containing modifiers for the regular expresion.

Supported modifiers:

* ``i``: match the regular expression case-insensitive
* ``m``: enables multiline matching

Example
_______

.. code-block:: yaml

  node_type: functiondef
  name:
    $regex: foo|bar
    modifiers: i
