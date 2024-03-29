Abstract
========

A refactoring tool for converting Python 2.x code to 3.x.

This is a work in progress! Bugs should be reported to http://bugs.python.org/
under the "2to3" category.


General usage
=============

Run ``./2to3`` to convert stdin (``-``), files or directories given as
arguments.

2to3 must be run with at least Python 2.5. The intended path for migrating to
Python 3.x is to first migrate to 2.6 (in order to take advantage of Python
2.6's runtime compatibility checks).


Files
=====

README                        - this file
lib2to3/refactor.py           - main program; use this to convert files or directory trees
test.py                       - runs all unittests for 2to3
lib2to3/patcomp.py            - pattern compiler
lib2to3/pytree.py             - parse tree nodes (not specific to Python, despite the name!)
lib2to3/pygram.py             - code specific to the Python grammar
scripts/example.py            - example input for play.py and fix_*.py
scripts/find_pattern.py       - script to help determine the PATTERN for a new fix
lib2to3/Grammar.txt           - Python grammar input (accepts 2.x and 3.x syntax)
lib2to3/Grammar.pickle        - pickled grammar tables (generated file, not in subversion)
lib2to3/PatternGrammar.txt    - grammar for the pattern language used by patcomp.py
lib2to3/PatternGrammar.pickle - pickled pattern grammar tables (generated file)
lib2to3/pgen2/                - Parser generator and driver ([1]_, [2]_)
lib2to3/fixes/                - Individual transformations
lib2to3/tests/                - Test files for pytree, fixers, grammar, etc


Capabilities
============

A quick run-through of 2to3's current fixers:

* **fix_apply** - convert apply() calls to real function calls.

* **fix_callable** - converts callable(obj) into hasattr(obj, '__call__').

* **fix_dict** - fix up dict.keys(), .values(), .items() and their iterator
  versions.
  
* **fix_except** - adjust "except" statements to Python 3 syntax (PEP 3110).

* **fix_exec** - convert "exec" statements to exec() function calls.

* **fix_execfile** - execfile(filename, ...) -> exec(open(filename).read())

* **fix_filter** - changes filter(F, X) into list(filter(F, X)).

* **fix_funcattrs** - fix function attribute names (f.func_x -> f.__x__).

* **fix_has_key** - "d.has_key(x)" -> "x in d".

* **fix_idioms** - convert type(x) == T to isinstance(x, T), "while 1:" to
  "while True:", plus others. This fixer must be explicitly requested
  with "-f idioms".

* **fix_imports** - Fix (some) incompatible imports.

* **fix_imports2** - Fix (some) incompatible imports that must run after
                     **test_imports**.

* **fix_input** - "input()" -> "eval(input())" (PEP 3111).

* **fix_intern** - "intern(x)" -> "sys.intern(x)".

* **fix_long** - remove all usage of explicit longs in favor of ints.

* **fix_map** - generally changes map(F, ...) into list(map(F, ...)).

* **fix_ne** - convert the "<>" operator to "!=".

* **fix_next** - fixer for it.next() -> next(it) (PEP 3114).

* **fix_nonzero** - convert __nonzero__() methods to __bool__() methods.

* **fix_numliterals** - tweak certain numeric literals to be 3.0-compliant.

* **fix_paren** - Add parentheses to places where they are needed in list
    comprehensions and generator expressions.

* **fix_operator** - fixer for functions gone from the operator module.

* **fix_print** - convert "print" statements to print() function calls.

* **fix_raise** - convert "raise" statements to Python 3 syntax (PEP 3109).

* **fix_raw_input** - "raw_input()" -> "input()" (PEP 3111).

* **fix_repr** - swap backticks for repr() calls.

* **fix_standarderror** - StandardError -> Exception.

* **fix_sys_exc** - Converts * **"sys.exc_info", "sys.exc_type", and
  "sys.exc_value" to sys.exc_info()

* **fix_throw** - fix generator.throw() calls to be 3.0-compliant (PEP 3109).

* **fix_tuple_params** - remove tuple parameters from function, method and
  lambda declarations (PEP 3113).
  
* **fix_unicode** - convert, e.g., u"..." to "...", unicode(x) to str(x), etc.

* **fix_urllib** - Fix imports for urllib and urllib2.
  
* **fix_xrange** - "xrange()" -> "range()".

* **fix_xreadlines** - "for x in f.xreadlines():" -> "for x in f:". Also,
  "g(f.xreadlines)" -> "g(f.__iter__)".

* **fix_metaclass** - move __metaclass__ = M to class X(metaclass=M)


Limitations
===========

General Limitations
-------------------

* In general, fixers that convert a function or method call will not detect
  something like ::

      a = apply
      a(f, *args)
    
  or ::

      m = d.has_key
      if m(5):
          ...
        
* Fixers that look for attribute references will not detect when getattr() or
  setattr() is used to access those attributes.
  
* The contents of eval() calls and "exec" statements will not be checked by
  2to3.

        
Caveats for Specific Fixers
---------------------------

fix_except
''''''''''

"except" statements like ::

    except Exception, (a, b):
        ...

are not fixed up. The ability to treat exceptions as sequences is being
removed in Python 3, so there is no straightforward, automatic way to
adjust these statements.

This is seen frequently when dealing with OSError.


fix_filter
''''''''''

The transformation is not correct if the original code depended on
filter(F, X) returning a string if X is a string (or a tuple if X is a
tuple, etc).  That would require type inference, which we don't do.  Python
2.6's Python 3 compatibility mode should be used to detect such cases.


fix_has_key
'''''''''''

While the primary target of this fixer is dict.has_key(), the
fixer will change any has_key() method call, regardless of what class it
belongs to. Anyone using non-dictionary classes with has_key() methods is
advised to pay close attention when using this fixer.


fix_map
'''''''

The transformation is not correct if the original code was depending on
map(F, X, Y, ...) to go on until the longest argument is exhausted,
substituting None for missing values -- like zip(), it now stops as
soon as the shortest argument is exhausted.


fix_raise
'''''''''

"raise E, V" will be incorrectly translated if V is an exception instance.
The correct Python 3 idiom is ::
   
    raise E from V
        
but since we can't detect instance-hood by syntax alone and since any client
code would have to be changed as well, we don't automate this.

Another translation problem is this: ::

    t = ((E, E2), E3)
    raise t
    
2to3 has no way of knowing that t is a tuple, and so this code will raise an
exception at runtime since the ability to raise tuples is going away.


Notes
=====

.. [#1] I modified tokenize.py to yield a NL pseudo-token for backslash
        continuations, so the original source can be reproduced exactly.  The
        modified version can be found at lib2to3/pgen2/tokenize.py.

.. [#2] I developed pgen2 while I was at Elemental Security.  I modified
        it while at Google to suit the needs of this refactoring tool.


Development
===========

The HACKING file has a list of TODOs -- some simple, some complex -- that would
make good introductions for anyone new to 2to3.


Licensing
=========

The original pgen2 module is copyrighted by Elemental Security.  All
new code I wrote specifically for this tool is copyrighted by Google.
New code by others is copyrighted by the respective authors.  All code
(whether by me or by others) is licensed to the PSF under a contributor
agreement.

--Guido van Rossum


All code I wrote specifically for this tool before 9 April 2007 is
copyrighted by me. All new code I wrote specifically for this tool after
9 April 2007 is copyrighted by Google. Regardless, my contributions are
licensed to the PSF under a contributor agreement.

--Collin Winter

All of my contributions are copyrighted to me and licensed to PSF under the
Python contributor agreement.

--Benjamin Peterson
