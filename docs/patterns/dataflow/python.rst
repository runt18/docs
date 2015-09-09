======
Python
======

This document gives an overview of the data-flow annotations that we add to the Python
abstract syntax tree (AST) using our abstract interpretation engine. The different annotations
are grouped by node type whenever possible and contain a short description, an estimation of their
fidelity/applicabilit and one or more usage examples.

Annotation Nodes
================

Annotation nodes will always have a `type` attribute that specifies the type of the given attribute,
as well as a number of additional attributes.

Variable Types
--------------

* `dict`: A Python dictionary
* `list`: A Python list
* `tuple`: A Python tuple
* `set`: A Python set


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
    a = 4.1 #this is a float

  print a

would generate the following annotation for the `a` name in the last line:

.. code-block:: yaml

  type: union
  values:
    - branch: af4a..
      type: int
      value: 4
    - branch: 75af..
      type: float
      value: 4.1

Here, the `branch` parameter in the `values` attribute of the union type tells you which code branch
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
the given point in the AST. If this is not possible, it will return a :json:`type: undefined`
annotation. If more than one value is possible, a :json:`type: union` will be returned.

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

  from django.http import HttpRequest

will yield a variable in the form

.. code-block:: yaml

  type: class
  ast: [AST node]
  fully_qualified_name: django.http.request.HttpResponse

Fully qualified names will be generated for all modules, classes, functions and variables that
are assigned to a name in a given scope. 

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
