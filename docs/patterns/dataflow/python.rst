======
Python
======

This document gives an overview of the data-flow annotations that we add to the Python
abstract syntax tree (AST) using our abstract interpretation engine. The different annotations
are grouped by node type whenever possible and contain a short description, an estimation of their
fidelity/applicabilit and one or more usage examples. Annotations can be accessed through the
:code:`_flow` branch. If no such branch exists in the AST the node does not have any annotations.

Annotation Nodes
================

Annotation nodes will always have a `type` attribute that specifies the type of the given attribute,
as well as a number of additional attributes. The most important types are listed below.

Variable Types
--------------

* `num`: A integer or float
* `str`: A string
* `dict`: A dictionary
* `list`: A list
* `tuple`: A tuple
* `set`: A set
* `module`: A module
* `call`: A call of a function
* `generator`: A generator function
* `function`: An function definition
* `class`: An class definition
* `binop`: A binary operation
* `boolop`: A boolean operation
* `ambiguous`: Will be set if there are multiple possibilities for the type, e.g. when assigning a variable both in an if and else branch (see below for more details).
* `unknown`: Will be set if the type can not be specified.
* `undefined`: Will be set if, for example a function call or a variable is not defined in the current scope.


Union Types
-----------

Often it is not possible to unambiguously deduce the exact value or type of a variable 
at a given point in the syntax tree. In this case, the Abstract Execution Engine (AIE) will return a so-called **Union type**,
which will contain a list of possible variable values/types for the given element. For example,
the code block

.. code-block:: python

  if condition: #branch ID: af4a..
    a = 4 #this is an int
  else: #branch ID: 75af..
    a = "four" #this is a string

  print a

would generate in part the following annotation for the `a` name in the last line:

.. code-block:: yaml

  type: ambiguous
  values:
    - af4a.. # the first branch
      branch: af4a..
      type: num
      value: 4
    - 75af.. # the second branch
      branch: 75af..
      type: str
      value: four

Here, the `branch` parameter of the shown annotation tells you which code branch
is responsible for the generation of the given value.

.. note:: Branches can get *combined* when there are multiple possible code paths that can be taken.
          Since the number of branch combinations can quickly explode in this scenario, the AIE
          will perform a pruning of branch values whenever possible. This might lead to reduced
          value/type precision in some cases. You can check the `pruned` attribute of the Union 
          type to see if actual branch pruning has happened.

Name Resolution
===============

A `name` node will either create or retrieve a variable of a given `id`. The `ctx` attribute
specifies whether we should store or load a variable. For example, the node

.. code-block:: yaml

  node_type: name
  id: foo
  ctx:
    node_type: load

would tell us that we should look up the variable with the id `foo`, whereas the node

.. code-block:: yaml

  node_type: name
  id: foo
  ctx:
    node_type: store

would tell us that we should store something into the variable with the id `foo`.

Annotations
-----------

For names with a `load` context, the AIE tries to determine the current value of the variable at
the given point in the AST. If this is not possible, it will return a :code:`type: undefined`
annotation. If more than one value is possible, a :code:`type: ambiguous` will be returned.

Imports
=======

Imports will be resolved against the current code environment (including dependencies), importing
the given variable into the current local scope. 

.. warning:: Currently, only absolute imports and dotted relative imports are supported.

Fully Qualified Variable Names
==============================

When importing or using values it is often handy to know where exactly the value has been defined.
The AIE supports this by adding a `fully_qualified_name` attribute to values. For example, the
code block

.. code-block:: python

  from django.http import HttpRequest as HR

  my_http_request = HR

will yield a variable which is named :code:`fully_qualified_name` and is placed in the :code:`_flow`
branch of the :code:`HR` object. The AST of the last line in this code snippet will look in part like the following.

.. code-block:: yaml

    node_type: assign
    _flow:
      target:
        fully_qualified_name: django.http.HttpRequest


Fully qualified names will be generated for all modules, classes, functions and variables that
are assigned to a name in a given scope. 
When writing patterns which use the fully qualified name it is usually the fastest to find the
specific branch name which holds this variable by defining a minimal working example and looking in
the generated AST for the correct branch since these can change depending on the current node.

.. warning::
    
    The fully qualified name works on the level of **variables** and not **names**, hence assigning
    a variable with an existing fully qualified name to something else (e.g., a name in a local
    scope), will NOT change this name. Example:

    .. code-block:: python

        #module name: "foo"

        #HttpRequest will have fully_qualified_name = "django.http.HttpRequest"
        from django.http import HttpRequest

        #this will have fully_qualified_name = "foo.a"
        a = "test"

        #this will have fully_qualified_name = "django.http.HttpRequest"
        my_http_request = HttpRequest

        #this will have fully_qualified_name = "foo.my_http_request"
        my_http_request = "foo"

Function Definitions
====================

Function definition nodes will contain the following additional information:

* The expected call signature of the function. This will contain information about
    * the minimum and maximum number of arguments
    * the minimum and maximum number of keyword arguments
    * the default arguments of the function
    * whether the function modifies a given argument or uses it in its return value
* The inferred return type(s) of the function and whether the function is a generator
* Possible exceptions that the function might throw

.. notice:: The AIE will try to detect recursion within the function body. If it detects a recursive
            call of a given function, it will use that information to infer the return type in case
            it depends on the recursive part of the function. Recursion detection works not only for
            direction recursion (e.g., the function calls itself) but also for indirect one (e.g.,
            the function calls another function which calls the original function)


Class Definitions
=================

Class definition nodes will contain the following additional information:

* All attributes of the class (including inherited attributes from base classes)
* The value of self after calling `__init__`

Function Calls & Class Calls
============================

Call statements will contain the following additional information:

* The function/class definition of the called object
* The result of the function call

Expressions
===========

Expressions will be resolved to their result types whenever possible.

Current Limitations
===================

Currently, the AIE will not provide full or correct information for the following constructs:

* Class and function decorators (coming soon)
* Classes with multiple inheritance (coming soon)
* Python built-in functions and variables (coming soon)
* Classes that make extensive use of Python's metaprogramming facilities (i.e. using __new__)
* Expressions containing overwritten operators from user-defined classes
