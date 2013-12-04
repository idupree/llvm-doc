========================================
Mapping High-Level Constructs to LLVM IR
========================================

.. contents::
   :local:
   :depth: 2


Task List
=========
This chapter serves as an informal task list and is to be deleted before
review:

#. Local Variables: Explain the "alloca trick", which makes it possible for
   a front-end to avoid building SSA.  Include an example.
#. How to enable debug information?  (Line and Function, Variable)
#. How to interface with a garbage collector? (link to existing docs)
#. How to express a custom calling convention? (link to existing docs)
#. Representing constructors, destructors, finalization
#. How to examine the stack at runtime?  How to modify it?  (i.e. reflection,
   interjection)
#. Representing subtyping checks (with full alias info), TBAA, struct-path
   TBAA.
#. How to exploit inlining (external, vs within LLVM)?
#. How to express array bounds checks for best optimization?
#. How to express null pointer checks?
#. How to express domain specific optimizations?  (i.e. lock elision, or
   matrix math simplification) (link to existing docs)
#. How to optimize call dispatch or field access in dynamic languages? (ref
   new patchpoint intrinsics for inline call caching and field access caching)

**TODO:** Ask various front-end implementors (Rust, Haskell (GHC), Rubinius,
and more) to review and/or contribute so as to make the document great.

Introduction
============
In this document we will take a look at how to map various classic high-level
programming language constructs to LLVM IR.  The purpose of the document is to
make the learning curve less steep for aspiring LLVM users.

For the sake of simplicity, we'll be working with a 32-bit target machine so
that pointers and word-sized operands are 32-bits.

Also, for the sake of readability we do not mangle (encode) names.  Rather,
they are given simple, easy-to-read names that reflect their purpose.  A
production compiler for any language but C would generally need to mangle
the names so as to avoid conflicts between symbols.

A big thanks to all those who have contributed code snippets and explanations
of how to do this or that in LLVM.  Without them, this document would not be
half of what is presently is!


A Quick Primer
==============
Here are a few things that you should know before reading this document:

#. LLVM IR is not machine code, but sort of the step just above assembly.
#. LLVM IR is highly typed so expect to be told when you do something wrong.
#. LLVM IR does not differentiate between signed and unsigned integers.
#. LLVM IR assumes two's complement signed integers so that say ``trunc``
   works equally well on signed and unsigned integers.
#. Global symbols begin with an ampersand (@).
#. Local symbols begin with a percent symbol (%).
#. All symbols must be declared or defined.
#. Don't worry that the LLVM IR at times can seem somewhat lengthy when it
   comes to expressing something; the optimizer will ensure the output is
   well optimized and you'll often see two or three LLVM IR instructions be
   coalesced into a single machine code instruction.
#. If in doubt, consult the :doc:`LangRef`.  If there is a conflict between
   the Language Reference and this document, this document is wrong!
#. All LLVM IR examples have been compiled successfully but not run (this is
   going to change so that all samples have been built and run succesfully).
#. All LLVM IR examples assume a POSIX environment with ``malloc``, etc.
#. All LLVM IR examples are presented without a data layout and without a
   target triple.  You need to add those yourself, if you want to actually
   build and run the samples.  You can compile and link using Clang, which
   accepts ``.ll`` files as its input.

Finally:
This is an informal guide, not a formal scientific paper. So your mileage may
vary. Some will find this document highly useful, others will have no use for
it. Hopefully, though, it will aid you in getting up to speed with LLVM.


Some Useful LLVM Tools
======================
The most important LLVM tools for use with this article are as follows:

#. ``llc``: The LLVM IR Compiler.  It compiles a ``.ll`` file into a ``.s``
   file, which is the assembly output from LLVM.
#. ``opt``: The LLVM IR Optimizer.  It optimizes a ``.ll`` file into the same
   ``.ll`` file (unless you specify an output file name using the ``-o=name``
   option).
#. ``llvm-dis``: The LLVM ByteCode Dissambler, which creates a ``.ll`` file
   from a ``.bc`` file.
#. ``clang`` or ``clang++`` with the ``-emit-llvm``, ``-c``, and ``-S``
   options, which generate a ``.ll`` file.

In short, the tools work as follows:

#. ``clang`` reads ``.c`` and writes ``.ll`` (when using
   ``-c -emit-llvm -S``).
#. ``clang++`` reads ``.cpp`` and writes ``.ll`` (when using
   ``-c -emit-llvm -S``).
#. ``llvm-dis`` reads ``.bc`` and writes ``.ll``.
#. ``opt`` reads ``.bc`` or ``.ll`` and writes the same as the input.
#. ``llc`` reads ``.ll`` and writes ``.s``.

While you are playing around with generating or writing LLVM IR, you may want
to add the option `` -fsanitize=undefined`` to Clang/Clang++ insofar you use
either of those.  This option makes Clang/Clang++ insert run-time checks in
places where it would normally output an ``ud2`` instruction.  This will
likely save you some trouble if you happen to generate undefined LLVM IR.
Please notice that this option only works for C and C++ compiles.


Mapping Basic Constructs into LLVM IR
=====================================
In this chapter, we'll look at the most basic and simple constructs that are
part of nearly all imperative/OOP languages out there.


Global Variables
----------------
Global varibles are trivial to implement in LLVM IR:

.. code-block:: cpp

   int variable = 14;

   int test()
   {
      return variable;
   }

.. code-block:: llvm

   @variable = global i32 14

   define i32 @main() nounwind {
      %1 = load i32* @variable
      ret i32 %1
   }

Please notice that LLVM views global variables as pointers; so you must
explicitly dereference the global variable using the ``load`` instruction
when accessing its value, likewise you must explicitly store the value of
a global variable using the ``store`` instruction.


Local Variables
---------------
There are basically two kinds of local variables in LLVM:

#. Register-allocated local variables (temporaries).
#. Stack-allocated local variables.

The former is created by introducing a new symbol for the variable:

.. code-block:: llvm

   %1 = ... result of some computation ...

The latter is created by allocating the variable on the stack:

.. code-block:: llvm

   %2 = alloca i32

Please notice that ``alloca`` yields a pointer to the allocated type.  As is
generally the case in LLVM, you must explicitly use a ``load`` or ``store``
instruction to read or write the value respectively.

The use of ``alloca`` allows for a neat trick that can simplify your code
generator in many cases.  The trick is to explicitly allocate all mutable
variables, including arguments, on the stack, initialize them with the
appropriate initial value and then operate on the stack as if that was your
end goal.  The trick is to run the "memory to register promotion" pass on your
code as part of the optimization phase.  This will make LLVM store as many of
the stack variables in registers as it possibly can.  That way you don't have
to ensure that the generated program is in SSA form but can generate code
without having to worry about this aspect of the code generation.

The trick is described in chapter `7.4, Mutable Variables in Kaleidoscope,
in the OCaml tutorial
<http://llvm.org/docs/tutorial/OCamlLangImpl7.html#mutable-variables-in-kaleidoscope>`_.

**TODO:** Insert proper Sphinx link to the above chapter.


Constants
---------
There are two different kinds of constants:

#. Constants that do *not* occupy allocated memory.
#. Constants that *do* occupy allocated memory.

The former are always expanded inline by the compiler as there is no LLVM IR
equivalent of those.  In other words, the compiler simply inserts the constant
value wherever it is being used in a computation:

.. code-block:: llvm

   %1 = add i32 %0, 17     ; 17 is an inlined constant

Constants that do occupy memory are defined using the ``constant`` keyword:

.. code-block:: llvm

   @hello = internal constant [6 x i8] c"hello\00"
   %struct = type { i32, i8 }
   @struct_constant = internal constant %struct { i32 16, i8 4 }

Such a constant is really a global variable whose visibility can be limited
with ``private`` or ``internal`` so that it is invisible outside the current
module.


Constant Expressions
--------------------
**TODO:** Document the various forms of constant expressions that exist and
how they can be very useful.  For instance, ``getelementptr`` constant
expressions are almost unavoidable in all but the simplest programs.


Size-Of Computations
--------------------
Even though the compiler ought to know the exact size of everything in
use (for statically checked languages), it can at times be convenient to ask
LLVM to figure out the size of a structure for you.  This is done with the
following little snippet of code:

.. code-block:: llvm

   %Struct = type { i8, i32, i8* }
   @Struct_size = constant i32 ptrtoint (%Struct* getelementptr (%Struct* null, i32 1)) to i32

``@Struct_size`` will now contain the size of the structure ``%Struct``. The
trick is to compute the offset of the second element in the zero-based array
starting at ``null`` and that way get the size of the structure.


Function Prototypes
-------------------
A function prototype, aka a profile, is translated into an equivalent
``declare`` declaration in LLVM IR:

.. code-block:: cpp

   int Bar(int value);

Becomes:

.. code-block:: llvm

   declare i32 @Bar(i32 %value)

Or you can leave out the descriptive parameter name:

.. code-block:: llvm

   declare i32 @Bar(i32)


Function Definitions
--------------------
The translation of function definitions depends on a range of factors, ranging
from the calling convention in use, whether the function is exception-aware or
not, and if the function is to be publicly available outside the module.


Simple Public Functions
"""""""""""""""""""""""
The most basic model is:

.. code-block:: cpp

   int Bar(void)
   {
      return 17;
   }

Becomes:

.. code-block:: llvm

   define i32 @Bar() nounwind {
      ret i32 17
   }


Simple Private Functions
""""""""""""""""""""""""
A static function is basically a module-private function that cannot be
referenced from outside of the defining module:

.. code-block:: llvm

   define private i32 @Foo() nounwind {
      ret i32 17
   }


Functions with a Variable Number of Parameters
""""""""""""""""""""""""""""""""""""""""""""""
To call a so-called vararg function, you first need to define or declare it
using the elipsis (...) and then you need to make use of a special syntax for
function calls that allows you to explictly list the types of the parameters
of the function that is being called.  This "hack" exists to allow overriding
a call to a function such as a function with variable parameters.  Please
notice that you only need to specify the return type once, not twice as you'd
have to do if it was a true cast:

.. code-block:: llvm

   declare i32 @printf(i8*, ...) nounwind

   @.text = internal constant [20 x i8] c"Argument count: %d\0A\00"

   define i32 @main(i32 %argc, i8** %argv) nounwind {
      ; printf("Argument count: %d\n", argc)
      %1 = call i32 (i8*, ...)* @printf(i8* getelementptr([20 x i8]* @.text, i32 0, i32 0), i32 %argc)
      ret i32 0
   }


Exception-Aware Functions
"""""""""""""""""""""""""
A function that is aware of being part of a larger scheme of exception-
handling is called an exception-aware function.  Depending upon the type of
exception handling being employed, the function may either return a pointer to
an exception instance, create a ``setjmp``/``longjmp`` frame, or simply
specify the ``uwtable`` (for UnWind Table) attribute.  These cases will all be
covered in great detail in the chapter on Exception Handling below.


Function Pointers
-----------------
Function pointers are expressed almost like in C and C++:

.. code-block:: cpp

   int (*Function)(char *buffer);

Becomes:

.. code-block:: llvm

   @Function = global i32(i8*)* null


Casts
-----
There are seven different types of casts:

#. Bitwise casts (type casts).
#. Zero-extending casts (unsigned upcasts).
#. Sign-extending casts (signed upcasts).
#. Truncating casts (signed and unsigned downcasts).
#. Floating-point extending casts (float upcasts).
#. Floating-point truncating casts (float downcasts).
#. Address-space casts (pointer casts).


Bitwise Casts
"""""""""""""
A bitwise cast (``bitcast``) basically reinterprets a given bit pattern
without changing any bits in the operand.  For instance, you could make a
bitcast of a pointer to byte into a pointer to some structure as follows:

.. code-block:: cpp

   typedef struct
   {
      int a;
   } Foo;

   extern void *malloc(size_t size);
   extern void free(void *value);

   void allocate()
   {
      Foo *foo = (Foo *) malloc(sizeof(Foo));
      foo.a = 12;
      free(foo);
   }

Becomes:

.. code-block:: llvm

   %Foo = type { i32 }

   declare i8* @malloc(i32)
   declare void @free(i8*)

   define void @allocate() nounwind {
      %1 = call i8* @malloc(i32 4)
      %foo = bitcast i8* %1 to %Foo*
      %2 = getelementptr %Foo* %foo, i32 0, i32 0
      store i32 2, i32* %2
      call void @free(i8* %1)
      ret void
   }


Zero-Extending Casts (Unsigned Upcasts)
"""""""""""""""""""""""""""""""""""""""
To upcast an unsigned value like in the example below:

.. code-block:: cpp

   uint8 byte = 117;
   uint32 word;

   void main()
   {
      /* The compiler automatically upcasts the byte to a word. */
      word = byte;
   }

You use the ``zext`` instruction:

.. code-block:: llvm

   @byte = global i8 117
   @word = global i32 0

   define void @main() nounwind {
      %1 = load i8* @byte
      %2 = zext i8 %1 to i32
      store i32 %2, i32* @word
      ret void
   }


Sign-Extending Casts (Signed Upcasts)
"""""""""""""""""""""""""""""""""""""
To upcast a signed value, you replace the ``zext`` instruction with the
``sext`` instruction and everything else works just like in the previous
section:

.. code-block:: llvm

   @char = global i8 -17
   @int  = global i32 0

   define void @main() nounwind {
      %1 = load i8* @char
      %2 = sext i8 %1 to i32
      store i32 %2, i32* @int
      ret void
   }


Truncating Casts (Signed and Unsigned Downcasts)
""""""""""""""""""""""""""""""""""""""""""""""""
Both signed and unsigned integers use the same instruction, ``trunc``, to
reduce the size of the number in question.  This is because LLVM IR assumes
that all signed integer values are in two's complement format for which
reason ``trunc`` is sufficient to handle both cases:

.. code-block:: llvm

   @int = global i32 -1
   @char = global i8 0

   define void @main() nounwind {
      %1 = load i32* @int
      %2 = trunc i32 %1 to i8
      store i8 %2, i8* @char
      ret void
   }


Floating-Point Extending Casts (Float Upcasts)
""""""""""""""""""""""""""""""""""""""""""""""
Floating points numbers can be extended using the ``fpext`` instruction:

.. code-block:: cpp

   float small = 1.25;
   double large;

   void main()
   {
      /* The compiler inserts an implicit float upcast. */
      large = small;
   }

Becomes:

.. code-block:: llvm

   @small = global float 1.25
   @large = global double 0.0

   define void @main() nounwind {
      %1 = load float* @small
      %2 = fpext float %1 to double
      store double %2, double* @large
      ret void
   }


Floating-Point Truncating Casts (Float Downcasts)
"""""""""""""""""""""""""""""""""""""""""""""""""
Likewise, a floating point number can be truncated to a smaller size:

.. code-block:: llvm

   @large = global double 1.25
   @small = global float 0.0

   define void @main() nounwind {
      %1 = load double* @large
      %2 = fptrunc double %1 to float
      store float %2, float* @small
      ret void
   }


Address-Space Casts (Pointer Casts)
"""""""""""""""""""""""""""""""""""
**TODO:** Find a useful example of an address-space casts, using the
``addrspacecast`` instruction, to be included here.


Incomplete Structure Types
--------------------------
Incomplete types are very useful for hiding the details of what fields a given
structure has.  A well-designed C interface can be made so that no details of
the structure are revealed to the client, so that the client cannot inspect or
modify private members inside the structure:

.. code-block:: c

   void Bar(struct Foo *);

Becomes:

.. code-block:: llvm

   %Foo = type opaque
   declare void @Bar(%Foo)


Structures
----------
LLVM IR already includes the concept of structures so there isn't much to do:

.. code-block:: c

   struct Foo
   {
      size_t _length;
   };

It is only a matter of discarding the actual field names and then index by
numerals starting from zero:

.. code-block:: llvm

   %Foo = type { i32 }


Nested Structures
-----------------
Nested structures are straightforward:

.. code-block:: llvm

   %Object = type {
      %Object*,      ; 0: above; the parent pointer
      i32            ; 1: value; the value of the node
   }


Unions
------
**TODO:** Document how create a union and how to use it.


Structure Expressions
---------------------
**TODO:** Document how to perform various computations on structures - get a
member, update a member, get a pointer to a member, load from a
pointer-to-member, and so forth.


Mapping Control Structures to LLVM IR
=====================================
**TODO:** Add common control structures such as ``if``, ``for``, ``switch``,
and ``while``.


Mapping Advanced Constructs to LLVM IR
======================================
In this chapter, we'll look at various non-OOP constructs that are highly
useful and are becoming more and more widespread in use.


Lambda Functions
----------------
A lambda function is basically an anonymous function with the added spice that
it may freely refer to the local variables (including argument variables) in
the containing function.  Lambdas are implemented just like Pascal's nested
functions, except the compiler is responsible for generating an internal name
for the lambda function.  There are a few different ways of implementing
lambda functions (see `Wikipedia on nested functions
<http://en.wikipedia.org/wiki/Nested_function>`_ for more information).

I'll give an example in pseudo-C++ because C++ does not currently support lambda
functions:

.. code-block:: cpp

   int foo(int a)
   {
      auto function = lambda(int x) { return x + a; }
      return function(10);
   }

Here the "problem" is that the lambda function references a local variable of
the caller, namely ``a``, even though the lambda function is a function of its
own.  This can be solved easily by passing the local variables in as implicit
arguments to the lambda function:

.. code-block:: llvm

   define internal i32 @lambda(i32 %a, i32 %x) alwaysinline nounwind {
      %1 = add i32 %a, %x
      ret i32 %1
   }

   define i32 @foo(i32 %a) nounwind {
      %1 = call i32 @lambda(i32 %a, i32 10)
      ret i32 %1
   }

Alternatively, if the lambda function uses more than a few variables, you can
wrap them up in a structure which you pass in a pointer to the lambda
function:

.. code-block:: cpp

   int foo(int a, int b)
   {
      int c = integer_parse();
      auto function = lambda(int x) { return (a + b - c) * x; }
      return function(10);
   }

Becomes:

.. code-block:: llvm

   %Lambda_Arguments = type {
      i32,        ; 0: a (argument)
      i32,        ; 1: b (argument)
      i32         ; 2: c (local)
   }

   define i32 @lambda(%Lambda_Arguments* %args, i32 %x) nounwind {
     %1 = getelementptr %Lambda_Arguments* %args, i32 0, i32 0
     %a = load i32* %1
     %2 = getelementptr %Lambda_Arguments* %args, i32 0, i32 1
     %b = load i32* %2
     %3 = getelementptr %Lambda_Arguments* %args, i32 0, i32 2
     %c = load i32* %3
      %4 = add i32 %a, %b
      %5 = sub i32 %4, %c
      %6 = mul i32 %5, %x
      ret i32 %6
   }

   declare i32 @integer_parse()

   define i32 @foo(i32 %a, i32 %b) nounwind {
      %args = alloca %Lambda_Arguments
      %1 = getelementptr %Lambda_Arguments* %args, i32 0, i32 0
      store i32 %a, i32* %1
      %2 = getelementptr %Lambda_Arguments* %args, i32 0, i32 1
      store i32 %b, i32* %2
      %c = call i32 @integer_parse()
      %3 = getelementptr %Lambda_Arguments* %args, i32 0, i32 2
      store i32 %c, i32* %3
      %4 = call i32 @lambda(%Lambda_Arguments* %args, i32 10)
      ret i32 %4
   }

Obviously there are some possible variations over this theme:

#. You could pass all implicit as explicit arguments as arguments.
#. You could pass all implicit as explicit arguments in the structure.
#. You could pass in a pointer to the frame of the caller and let the lambda
   function extract the arguments and locals from the input frame.


Closures
--------
**TODO:** Describe closures.


Generators
----------
A generator is a function that repeatedly yields a value in such a way that
the function's state is preserved across the repeated calls of the function.

The most straigthforward way to implement a generator is by wrapping all of
its state variables (arguments, local variables, and return values) up into an
ad-hoc structure and then pass the address of that structure to the generator.

I resort to pseudo-C++ because C++ does not directly support generators:

.. code-block:: cpp

   generator int foo(int start, int after)
   {
      for (int index = start; index < after; index++)
         if (i % 2 == 0)
            yield index + 1;
         else
            yield index - 1;
   }

   extern void integer_print(int value);

   void main(void)
   {
      foreach (int i in foo(0, 5))
         integer_print(i);
   }

This becomes something like this:

.. code-block:: llvm

   %foo_context = type {
      i8*,      ; 0: block (state)
      i32,      ; 1: start (argument)
      i32,      ; 2: after (argument)
      i32,      ; 3: index (local)
      i32,      ; 4: value (result)
      i1        ; 5: again (result)
   }

   define void @foo_setup(%foo_context* %context, i32 %start, i32 %after) nounwind {
      ; set up 'block'
      %1 = getelementptr %foo_context* %context, i32 0, i32 0
      store i8* blockaddress(@foo_yield, %.init), i8** %1

      ; set up 'start'
      %2 = getelementptr %foo_context* %context, i32 0, i32 1
      store i32 %start, i32* %2

      ; set up 'after'
      %3 = getelementptr %foo_context* %context, i32 0, i32 2
      store i32 %after, i32* %3

      ret void
   }

   define i1 @foo_yield(%foo_context* %context) nounwind {
      %1 = getelementptr %foo_context* %context, i32 0, i32 0
      %2 = load i8** %1
      indirectbr i8* %2, [ label %.init, label %.head ]

   .init:
      ; copy argument 'start' to the local variable 'index'
      %3 = getelementptr %foo_context* %context, i32 0, i32 1
      %start = load i32* %3
      %4 = getelementptr %foo_context* %context, i32 0, i32 3
      store i32 %start, i32* %4
      br label %.head

   .head:
      ; for (; index < after; )
      %5 = getelementptr %foo_context* %context, i32 0, i32 3
      %index = load i32* %5
      %6 = getelementptr %foo_context* %context, i32 0, i32 2
      %after = load i32* %6
      %again = icmp slt i32 %index, %after
      br i1 %again, label %.body, label %.done

   .body:
      %7 = getelementptr %foo_context* %context, i32 0, i32 0
      store i8* blockaddress(@foo_yield, %.next), i8** %7

   .next:
      ; yield next value
      ; copy 'index' to 'value'
      %7 = getelementptr %foo_context* %context, i32 0, i32 4
      store i32 %index, i32* %7

     ; increment 'index'
      %8 = add i32 1, %index
      store i32 %8, i32* %5
      br label %.tail

   .done:
      ret i1 %again
   }


   declare void @integer_print(i32)


   define void @main() nounwind {
      ; allocate and initialize generator context structure
      %context = alloca %foo_context
      call void @foo_setup(%foo_context* %context, i32 0, i32 5)
      br label %.head

   .head:
      ; for (int i in foo(0, 5))
      %1 = call i1 @foo_yield(%foo_context* %context)
      br i1 %1, label %.body, label %.tail

   .body:
      %2 = getelementptr %foo_context* %context, i32 0, i32 4
      %3 = load i32* %2
      call void @integer_print(i32 %3)
      br label %.head

   .tail:
      ret void
   }


Mapping Exception Handling to LLVM IR
=====================================
Exceptions can be implemented in one of three ways:

#. The simple way, by using a propagated return value.
#. The bulky way, by using ``setjmp`` and ``longjmp``.
#. The efficient way, by using a zero-cost exception ABI.

Please notice that many compiler developers with respect for themselves won't
accept the first method as a proper way of handling exceptions.  However, it
is unbeatable in terms of simplicity and can likely help people to understand
that implementing exceptions does not need to be very difficult.

The second method is used by some production compilers, but it has large
overhead both in terms of code bloat and the cost of a ``try-catch`` statement
(because all CPU registers are saved using ``setjmp`` whenever a ``try``
statement is encountered).

The third method is very advanced but in return does not add any costs to
execution paths where no exceptions are being thrown. This method is the
de-facto "right" way of implementing exceptions, whether you like it or not.
LLVM directly supports this kind of exception handling.

In the three sections below, we'll be using this sample and transform it:

.. code-block:: cpp

   #include <stdio.h>
   #include <stddef.h>

   class Foo
   {
   public:
      int GetLength() const
      {
         return _length;
      }

      void SetLength(int value)
      {
         _length = value;
      }

   private:
      int _length;
   };

   int Bar(bool fail)
   {
      Foo foo;
      foo.SetLength(17);
      if (fail)
            throw new Exception("Exception requested by caller");
      foo.SetLength(24);
      return foo.GetLength();
   }

   int main(int argc, const char *argv[])
   {
      int result;

      try
      {
         /* First call does not throw an exception. */
         int value = Bar(false);

         /* Second call throws an exception. */
         value = Bar(true);

         /* We never get here. */
         result = EXIT_SUCCESS;
      }
      catch (Exception *that)
      {
         printf("Error: %s\n", that->GetText());
         result = EXIT_FAILURE;
      }
      catch (...)
      {
         puts("Internal error: Unhandled exception detected");
         result = EXIT_FAILURE;
      }

      return result;
   }


Exception Handling by Propagated Return Value
---------------------------------------------
This method basically is a compiler-generated way of implicitly checking each
function's return value.  Its main advantage is that it is simple - at the
cost of many mostly unproductive checks of return values.  The great thing
about this method is that it readily interfaces with a host of languages and
environments - it is all a matter of returning a pointer to an exception.

**STATUS:** Compiled and run succesfully on 2013.12.04 by Mikael Lyngvig.

This maps to the following code:

.. code-block:: llvm

   ;********************* External and Utility functions *********************

   declare i8* @malloc(i32) nounwind
   declare void @free(i8*) nounwind
   declare i32 @printf(i8* noalias nocapture, ...) nounwind
   declare i32 @puts(i8* noalias nocapture) nounwind

   define i8* @xfree(i8* %value) nounwind {
      call void @free(i8* %value)
      ret i8* null
   }

   ;***************************** Object class *******************************

   %Object_vtable_type = type {
      %Object_vtable_type*,         ; 0: above: parent class vtable pointer
      i8*                           ; 1: class: class name (usually mangled)
      ; virtual methods would follow here
   }

   @.Object_class_name = private constant [7 x i8] c"Object\00"

   @.Object_vtable = private constant %Object_vtable_type {
     %Object_vtable_type* null,  ; This is the root object of the object hierarchy
      i8* getelementptr([7 x i8]* @.Object_class_name, i32 0, i32 0)
   }

   %Object = type {
     %Object_vtable_type*        ; 0: vtable: class vtable pointer (always non-null)
     ; class data members would follow here
   }

   ; returns true if the specified object is identical to or derived from the
   ; class with the specified name.
   define i1 @Object_IsA(%Object* %object, i8* %name) nounwind {
   .init:
     ; if (object == null) return false
     %0 = icmp ne %Object* %object, null
     br i1 %0, label %.once, label %.exit_false

   .once:
      %1 = getelementptr %Object* %object, i32 0, i32 0
      br label %.body

   .body:
      ; if (vtable->class == name)
      %2 = phi %Object_vtable_type** [ %1, %.once ], [ %7, %.next]
      %3 = load %Object_vtable_type** %2
      %4 = getelementptr %Object_vtable_type* %3, i32 0, i32 1
      %5 = load i8** %4
      %6 = icmp eq i8* %5, %name
      br i1 %6, label %.exit_true, label %.next

   .next:
      ; object = object->above
      %7 = getelementptr %Object_vtable_type* %3, i32 0, i32 0

      ; while (object != null)
      %8 = icmp ne %Object_vtable_type* %3, null
      br i1 %8, label %.body, label %.exit_false

   .exit_true:
      ret i1 true

   .exit_false:
      ret i1 false
   }


   ;*************************** Exception class ******************************

   %Exception_vtable_type = type {
     %Object_vtable_type*,                         ; 0: parent class vtable pointer
     i8*                                           ; 1: class name
     ; virtual methods would follow here.
   }

   @.Exception_class_name = private constant [10 x i8] c"Exception\00"

   @.Exception_vtable = private constant %Exception_vtable_type {
      %Object_vtable_type* @.Object_vtable,        ; the parent of this class is the Object class
      i8* getelementptr([10 x i8]* @.Exception_class_name, i32 0, i32 0)
   }

   %Exception = type {
     %Exception_vtable_type*,                      ; 0: the vtable pointer
     i8*                                           ; 1: the _text member
   }

   define void @Exception_Create_String(%Exception* %this, i8* %text) nounwind {
     ; set up vtable
     %1 = getelementptr %Exception* %this, i32 0, i32 0
     store %Exception_vtable_type* @.Exception_vtable, %Exception_vtable_type** %1

     ; save input text string into _text
     %2 = getelementptr %Exception* %this, i32 0, i32 1
     store i8* %text, i8** %2

     ret void
   }

   define i8* @Exception_GetText(%Exception* %this) nounwind {
     %1 = getelementptr %Exception* %this, i32 0, i32 1
     %2 = load i8** %1
     ret i8* %2
   }


   ;******************************* Foo class ********************************

   %Foo = type { i32 }

   define void @Foo_Create_Default(%Foo* %this) nounwind {
     %1 = getelementptr %Foo* %this, i32 0, i32 0
     store i32 0, i32* %1
     ret void
   }

   define i32 @Foo_GetLength(%Foo* %this) nounwind {
      %1 = getelementptr %Foo* %this, i32 0, i32 0
      %2 = load i32* %1
      ret i32 %2
   }

   define void @Foo_SetLength(%Foo* %this, i32 %value) nounwind {
     %1 = getelementptr %Foo* %this, i32 0, i32 0
     store i32 %value, i32* %1
     ret void
   }


   ;********************************* Foo function ***************************

   @.message1 = internal constant [30 x i8] c"Exception requested by caller\00"

   define %Exception* @Bar(i1 %fail, i32* %result) nounwind {
      ; Allocate Foo instance
      %foo = alloca %Foo
      call void @Foo_Create_Default(%Foo* %foo)

      call void @Foo_SetLength(%Foo* %foo, i32 17)

      ; if (fail)
      %1 = icmp eq i1 %fail, true
      br i1 %1, label %.if_begin, label %.if_close

   .if_begin:
      ; throw new Exception(...)
      %2 = call i8* @malloc(i32 8)
      %3 = bitcast i8* %2 to %Exception*
      %4 = getelementptr [30 x i8]* @.message1, i32 0, i32 0
      call void @Exception_Create_String(%Exception* %3, i8* %4)
      ret %Exception* %3

   .if_close:
      ; foo.SetLength(24)
      call void @Foo_SetLength(%Foo* %foo, i32 24)
      %5 = call i32 @Foo_GetLength(%Foo* %foo)
      store i32 %5, i32* %result
     ret %Exception* null
   }


   ;********************************* Main program ***************************

   @.message2 = internal constant [11 x i8] c"Error: %s\0A\00"
   @.message3 = internal constant [44 x i8] c"Internal error: Unhandled exception detectd\00"

   define i32 @main(i32 %argc, i8** %argv) nounwind {
      ; "try" keyword expands to nothing.

      ; Body of try block.
      ; First call.
      %1 = alloca i32
      %2 = call %Exception* @Bar(i1 false, i32* %1)
      %3 = icmp eq %Exception* %2, null
      br i1 %3, label %.second, label %.catch_block

      ; Second call.
   .second:
      %4 = call %Exception* @Bar(i1 true, i32* %1)
      %5 = icmp eq %Exception* %4, null
      br i1 %5, label %.exit, label %.catch_block

   .catch_block:
      %6 = phi %Exception* [ %2, %0 ], [ %4, %.second ]
      %7 = getelementptr [10 x i8]* @.Exception_class_name, i32 0, i32 0
      %8 = bitcast %Exception* %6 to %Object*
      %9 = call i1 @Object_IsA(%Object* %8, i8* %7)
      br i1 %9, label %.catch_exception, label %.catch_all

   .catch_exception:
      %10 = getelementptr [11 x i8]* @.message2, i32 0, i32 0
      %11 = call i8* @Exception_GetText(%Exception* %6)
      %12 = call i32 (i8*, ...)* @printf(i8* %10, i8* %11)
      br label %.exit

   .catch_all:
      %13 = getelementptr [44 x i8]* @.message3, i32 0, i32 0
      %14 = call i32 @puts(i8* %13)
      br label %.exit

   .exit:
      %result = phi i32 [ 0, %.second ], [ 1, %.catch_exception ], [ 1, %.catch_all ]
      ret i32 %result
   }


``setjmp``/``longjmp`` Exception Handling
-----------------------------------------
The basic idea behind the ``setjmp`` and ``longjmp`` exception handling scheme
is that you save the CPU state whenever you encounter a ``try`` keyword and
then do a ``longjmp`` whenever you throw an exception.  If there are few
``try`` blocks in the program, as is typically the case, the cost of this
method is not as high as it might seem.  However, often there are implicit
exception handlers due to the need to release local resources such as class
instances allocated on the stack and then the cost can become quite high.

``setjmp``/``longjmp`` exception handling is often abbreviated ``SjLj``
for ``SetJmp``/``LongJmp``.

.. code-block:: cpp

   #include <stdio.h>

   class Exception
   {
   public:
      Exception(const char *text)
      {
         _text = text;
      }

      const char *GetText() const
      {
         return _text;
      }

   private:
      const char *_text;
   };

   int main(int argc, const char *argv[])
   {
      int result = EXIT_FAILURE;

      try
      {
         if (argc == 1)
            throw Exception("Syntax: 'program' source-file target-file");

         result = EXIT_SUCCESS;
      }
      catch (Exception that)
      {
         puts(that.GetText());
      }

      return result;
   }

This translates into something like this:

.. code-block:: llvm

   declare int @puts(i8*)

   ; jmp_buf is very platform dependent, this is for illustration only...
   %jmp_buf = type { i32 }
   declare int @setjmp(%jmp_buf* %env)
   declare void @longjmp(%jmp_buf* %env, i32 %val)

   %Exception = type { i8* }

   define void @Exception_Create(%Exception* %this, i8* %text) nounwind {
      %1 = getelementptr %Exception* %this, i32 0, i32 0    ; %1 = &%this._text
      store i8** %1, %text
      ret void
   }

   define i8* @Exception_GetText(%Exception* %this) nounwind {
      %1 = getelementptr %Exception* %this, i32, i32 0      ; %1 = &%this._text
      %2 = load i8** %1
      ret i8* %2
   }

   @.message = internal constant [42 x i8] c"Syntax: 'program' source-file target-file\00"

   define i32 @main(i32 %argc, i8** %argv) nounwind {
      ; "try" keyword expands into this:
      %1 = alloca %jmp_buf
      %2 = call i32 @setjmp(%jmp_buf* %1)

      ; if actual call to setjmp, the result is zero.
      ; if a longjmp, the result is non-zero.
      %3 = icmp eq i32 %2, 0
      br i1 %3, label %.saved, label %.catch

   .saved:
      ; the body of the "try" statement expands to this:
      %4 = icmp eq i32 %argc, 1
      br i1 %4, label %.if_begin, label %.if_close

   .if_begin:
      %5 = alloca %Exception
      %6 = getelementptr [42 x i8]* @.message, i32 0, i32 0
      call void @Exception_Create(%Exception* %5, i8* %6)
      %7 = bitcast %Exception* %5 to i32
      call void @longjmp(%jmp_buf* %1, i32 %7)
      br label %.if_close

   .if_close:
      ret i32 0      ; EXIT_SUCCESS

   .catch:


**TODO:** Finish up ``setjmp``/``longjmp`` example.


Zero Cost Exception Handling
----------------------------
**TODO:** Explain how to implement exception handling using zero cost
exception handling.


Resources
---------

#. `Compiler Internals - Exception Handling
   <http://www.hexblog.com/wp-content/uploads/2012/06/Recon-2012-Skochinsky-Compiler-Internals.pdf>`_.
#. `Exception Handling in C without C++ <http://www.on-time.com/ddj0011.htm>`_.
#. `How a C++ Compiler Implements Exception Handling
   <http://www.codeproject.com/Articles/2126/How-a-C-compiler-implements-exception-handling>`_.
#. `DWARF standard - Exception Handling
   <http://wiki.dwarfstd.org/index.php?title=Exception_Handling>`_.
#. `Itanium C++ ABI <http://refspecs.linuxfoundation.org/cxxabi-1.86.html>`_.


Mapping Object-Oriented Constructs to LLVM IR
=============================================
In this chapter we'll look at various object-oriented constructs and see how
they can be mapped to LLVM IR.


Classes
-------
A class is basically nothing more than a structure with an associated set of
functions that take an implicit first parameter, namely a pointer to the
structure.  Therefore, is is very trivial to map a class to LLVM IR:

.. code-block:: cpp

   #include <stddef.h>

   class Foo
   {
   public:
      Foo()
      {
         _length = 0;
      }

      size_t GetLength() const
      {
         return _length;
      }

      void SetLength(size_t value)
      {
         _length = value;
      }

   private:
      size_t _length;
   };

We first transform this code into two separate pieces:

#. The structure definition.
#. The list of methods, including the constructor.

.. code-block:: llvm

   ; The structure definition for class Foo.
   %Foo = type { i32 }

   ; The default constructor for class Foo.
   define void @Foo_Create_Default(%Foo* %this) nounwind {
      %1 = getelementptr %Foo* %this, i32 0, i32 0
      store i32 0, i32* %1
      ret void
   }

   ; The Foo::GetLength() method.
   define i32 @Foo_GetLength(%Foo* %this) nounwind {
      %1 = getelementptr %Foo* %this, i32 0, i32 0
      %2 = load i32* %this
      ret i32 %2
   }

   ; The Foo::SetLength() method.
   define void @Foo_SetLength(%Foo* %this, i32 %value) nounwind {
      %1 = getelementptr %Foo* %this, i32 0, i32 0
      store i32 %value, i32* %1
      ret void
   }

Then we make sure that the constructor (``Foo_Create_Default``) is invoked
whenever an instance of the structure is created:

.. code-block:: cpp

   Foo foo;

.. code-block:: llvm

   %foo = alloca %Foo
   call void @Foo_Create_Default(%Foo* %foo)


Virtual Methods
---------------
A virtual method is basically no more than a compiler-controlled function
pointer.  Each virtual method is recorded in the ``vtable``, which is a
structure of all the function pointers needed by a given class:

.. code-block:: cpp

   class Foo
   {
   public:
      virtual int GetLengthTimesTwo() const
      {
         return _length * 2;
      }

      void SetLength(size_t value)
      {
         _length = value;
      }

   private:
      int _length;
   };

   Foo foo;
   foo.SetLength(4);
   return foo.GetLengthTimesTwo();

This becomes:

.. code-block:: llvm

   %Foo_vtable_type = type { i32(%Foo*)* }

   %Foo = type { %Foo_vtable_type*, i32 }

   define i32 @Foo_GetLengthTimesTwo(%Foo* %this) nounwind {
      %1 = getelementptr %Foo* %this, i32 0, i32 1
      %2 = load i32* %1
      %3 = mul i32 %2, 2
      ret i32 %3
   }

   @Foo_vtable_data = global %Foo_vtable_type {
      i32(%Foo*)* @Foo_GetLengthTimesTwo
   }

   define void @Foo_Create_Default(%Foo* %this) nounwind {
      %1 = getelementptr %Foo* %this, i32 0, i32 0
      store %Foo_vtable_type* @Foo_vtable_data, %Foo_vtable_type** %1
      %2 = getelementptr %Foo* %this, i32 0, i32 1
      store i32 0, i32* %2
      ret void
   }

   define void @Foo_SetLength(%Foo* %this, i32 %value) nounwind {
      %1 = getelementptr %Foo* %this, i32 0, i32 1
      store i32 %value, i32* %1
      ret void
   }

   define i32 @main(i32 %argc, i8** %argv) nounwind {
      %foo = alloca %Foo
      call void @Foo_Create_Default(%Foo* %foo)
      call void @Foo_SetLength(%Foo* %foo, i32 4)
      %1 = getelementptr %Foo* %foo, i32 0, i32 0
      %2 = load %Foo_vtable_type** %1
      %3 = getelementptr %Foo_vtable_type* %2, i32 0, i32 0
      %4 = load i32(%Foo*)** %3
      %5 = call i32 %4(%Foo* %foo)
      ret i32 %5
   }

Please notice that some C++ compilers store ``_vtable`` at a negative offset
into the structure so that things like ``memcpy(this, 0, sizeof(*this))``
work, even though such commands should always be avoided in an OOP context.


Single Inheritance
------------------
Single inheritance is very straightforward: Each "structure" (class) is laid
out in memory after one another in declaration order.

.. code-block:: cpp

   class Base
   {
   public:
      void SetA(int value)
      {
         _a = value;
      }

   private:
      int _a;
   };

   class Derived: public Base
   {
   public:
      void SetB(int value)
      {
         SetA(value);
         _b = value;
      }

   protected:
      int _b;
   }

Here, ``a`` and ``b`` will be laid out to follow one another in memory so that
inheriting from a class is simply a matter of declaring a the base class as a
first member in the inheriting class:

.. code-block:: llvm

   %Base = type {
      i32         ; '_a' in class Base
   }

   define void @Base_SetA(%Base* %this, i32 %value) nounwind {
      %1 = getelementptr %Base* %this, i32 0, i32 0
      store i32 %value, i32* %1
      ret void
   }

   %Derived = type {
      i32,        ; '_a' from class Base
      i32         ; '_b' from class Derived
   }

   define void @Derived_SetB(%Derived* %this, i32 %value) nounwind {
      %1 = bitcast %Derived* %this to %Base*
      call void @Base_SetA(%Base* %1, i32 %value)
      %2 = getelementptr %Derived* %this, i32 0, i32 1
      store i32 %value, i32* %2
      ret void
   }

So the base class simply becomes plain members of the type declaration for the
derived class.

And then the compiler must insert appropriate type casts whenever the derived
class is being referenced as its base class as shown above with the
``bitcast`` operator.


Multiple Inheritance
--------------------
Multiple inheritance is not that difficult, either, it is merely a question of
laying out the multiply inherited "structures" in order inside each derived
class.

.. code-block:: cpp

   class BaseA
   {
   public:
      void SetA(int value)
      {
         _a = value;
      }

   private:
      int _a;
   };

   class BaseB: public BaseA
   {
   public:
      void SetB(int value)
      {
         SetA(value);
         _b = value;
      }

   private:
      int _b;
   };

   class Derived:
      public BaseA,
      public BaseB
   {
   public:
      void SetC(int value)
      {
         SetB(value);
         _c = value;
      }

   private:
      int _c;
   };

This is equivalent to the following LLVM IR:

.. code-block:: llvm

   %BaseA = type {
      i32         ; '_a' from BaseA
   }

   define void @BaseA_SetA(%BaseA* %this, i32 %value) nounwind {
      %1 = getelementptr %BaseA* %this, i32 0, i32 0
      store i32 %value, i32* %1
      ret void
   }

   %BaseB = type {
      i32,        ; '_a' from BaseA
      i32         ; '_b' from BaseB
   }

   define void @BaseB_SetB(%BaseB* %this, i32 %value) nounwind {
      %1 = bitcast %BaseB* %this to %BaseA*
      call void @BaseA_SetA(%BaseA* %1, i32 %value)
      %2 = getelementptr %BaseB* %this, i32 0, i32 1
      store i32 %value, i32* %2
      ret void
   }

   %Derived = type {
      i32,        ; '_a' from BaseA
      i32,        ; '_b' from BaseB
      i32         ; '_c' from Derived
   }

   define void @Derived_SetC(%Derived* %this, i32 %value) nounwind {
      %1 = bitcast %Derived* %this to %BaseB*
      call void @BaseB_SetB(%BaseB* %1, i32 %value)
      %2 = getelementptr %Derived* %this, i32 0, i32 2
      store i32 %value, i32* %2
      ret void
   }

And the compiler then supplies the needed type casts and pointer arithmentic
whenever ``baseB`` is being referenced as an instance of ``BaseB``.  Please
notice that all it takes is a ``bitcast`` from one class to another as well
as an adjustment of the last argument to ``getelementptr``.


Virtual Inheritance
-------------------
Virtual inheritance is actually quite simple as it dictates that identical
base classes are to be merged into a single occurence.  For instance, given
this:

.. code-block:: cpp

   class BaseA
   {
   public:
      int a;
   };

   class BaseB: public BaseA
   {
   public:
      int b;
   };

   class BaseC: public BaseA
   {
   public:
      int c;
   };

   class Derived:
      public virtual BaseB,
      public virtual BaseC
   {
      int d;
   };

``Derived`` will only contain a single instance of ``BaseA`` even if its
inheritance graph dictates that it should have two instances.  The result
looks something like this:

.. code-block:: cpp

   class Derived
   {
   public:
      int a;
      int b;
      int c;
      int d;
   };

So the second instance of ``a`` is silently ignored because it would cause
multiple instances of ``BaseA`` to exist in ``Derived``, which would clearly
cause lots of confusion and ambiguities.


Interfaces
----------
An interface is basically nothing more than a base class with no data members,
where all the methods are pure virtual (i.e. has no body).

As such, we've already described how to convert an interface to LLVM IR - it
is done precisely the same way that you convert a virtual member function to
LLVM IR.


Boxing
------
Boxing is the process of converting a non-object primitive value into an
object.  It is as easy as it sounds.  You basically create a wrapper class
which you instantiate and initialize with the non-object value:

**TODO:** Document how to box a value (create instance, initialize instance).


Unboxing
--------
Unboxing is the reverse of boxing: You downgrade a full object to a mere
scalar value by retrieving the boxed value from the box object.

**TODO:** Document how to unbox an object.


Class Equivalence Test
----------------------
There are basically two ways of doing this:

#. If you can guarantee that each class a unique ``vtable``, you can simply
   compare the pointers to the ``vtable``.
#. If you cannot guarantee that each class has a unique ``vtable`` (because
   different ``vtables`` may have been merged by the linker), you need to add
   a unique field to the ``vtable`` so that you can compare that instead.

The first variant goes roughly as follows (assuming identical strings aren't
merged by the compiler, something that they are most of the time):

.. code-block:: cpp

   bool equal = (typeid(first) == typeid(other));

.. code-block:: llvm

   %object_vtable_type = type { i8* }
   %object_vtable_data = internal constant { [8 x i8]* c"object\00" }

   define i1 @typeequals(%object* %first, %object* %other) {
      %1 = getelementptr %object* %first, i32 0, i32 0
      %2 = load
      %2 = getelementptr %object* %other, i32 0, i32 0


As far as I know, RTTI is simply done by adding two fields to the ``_vtable``
structure: ``parent`` and ``signature``.  The former is a pointer to the
vtable of the parent class and the latter is the mangled (encoded) name of
the class.  To see if a given class is another class, you simply compare the
``signature`` fields.  To see if a given class is a derived class of some
other class, you simply walk the chain of ``parent`` fields, while checking
if you have found a matching signature.


Class Inheritance Test
----------------------
A class inheritance test is basically a question of the form:

   | Is class X identical to or derived from class Y?

To answer that question, we can use one of two methods:

#. The naive implementation where we search upwards in the chain of parents.
#. The faster implementation where we search a preallocated list of parents.

The naive implementation works as follows:

.. code-block:: llvm

   define @naive_instanceof(%object* %first, %object* %other) nounwind {
      ; compare the two instances
      %first1 = getelementptr %object %first, i32 0, i32 0
      %first2 = load %object* %first1
      %other1 = getelementptr %object %other, i32 0, i32 0
      %other2 = load %object* %other2
      %equal = icmp eq i32 %first2, %other2
      br i1 %equal, label @.match, label @.mismatch
   .match:

      %2 = getelementptr %object %
      ; ascend up the chain of parents

**TODO:** Finish up Class Inheritance Test example.


The ``new`` Operator
--------------------
The ``new`` operator is generally nothing more than a type-safe version of the
C ``malloc`` function - in some implementations of C++, they may even be
called interchangeably without causing unseen or unwanted side-effects.


The Instance ``new`` Operator
"""""""""""""""""""""""""""""
All calls of the form ``new X`` are mapped into:

.. code-block:: llvm

   declare i8* @malloc(i32) nounwind

   %X = type { i8 }

   define void @X_Create_Default(%X* %this) nounwind {
      %1 = getelementptr %X* %this, i32 0, i32 0
      store i8 0, i8* %1
      ret void
   }

   define void @main() nounwind {
      %1 = call i8* @malloc(i32 1)
      %2 = bitcast i8* %1 to %X*
      call void @X_Create_Default(%X* %2)
      ret void
   }

Calls of the form ``new X(Y, Z)`` are the same, except ``Y`` and ``Z`` are
passed into the constructor.


The Array ``new`` Operator
""""""""""""""""""""""""""
New operations involving arrays are equally simple.  The code ``new X[100]``
is mapped into a loop that initializes each array element in turn:

.. code-block:: llvm

   declare i8* @malloc(i32) nounwind

   %X = type { i32 }

   define void @X_Create_Default(%X* %this) nounwind {
      %1 = getelementptr %X* %this, i32 0, i32 0
      store i32 0, i32* %1
      ret void
   }

   define void @main() nounwind {
      %n = alloca i32                  ; %n = ptr to the number of elements in the array
      store i32 100, i32* %n

      %i = alloca i32                  ; %i = ptr to the loop index into the array
      store i32 0, i32* %i

      %1 = load i32* %n                ; %1 = *%n
      %2 = mul i32 %1, 4               ; %2 = %1 * sizeof(X)
      %3 = call i8* @malloc(i32 %2)    ; %3 = malloc(100 * sizeof(X))
      %4 = bitcast i8* %3 to %X*       ; %4 = (X*) %3
      br label %.loop_head

   .loop_head:                         ; for (; %i < %n; %i++)
      %5 = load i32* %i
      %6 = load i32* %n
      %7 = icmp slt i32 %5, %6
      br i1 %7, label %.loop_body, label %.loop_tail

   .loop_body:
      %8 = getelementptr %X* %4, i32 %5
      call void @X_Create_Default(%X* %8)

      %9 = add i32 %5, 1
      store i32 %9 i32* %i

      br label %.loop_head

   .loop_tail:
      ret void
   }


Interoperating with a Runtime Library
=====================================
Explain that it is common to provide a set of run-time support functions that
are written in another language than LLVM IR and that it is trivially easy to
interface to such a run-time library.  Give examples of this.

The advantages of a custom, non-IR run-time library function is that it can be
optimized by hand to provide the best possible performance under certain
criteria.  The advantages of IR run-time library functions is that they can be
inlined automatically by the optimizer.


Interfacing to the Operating System
===================================
I'll divide this chapter into two sections:

#. How to Interface to POSIX Operating Systems.
#. How to Interface to the Windows Operating System.


How to Interface to POSIX Operating Systems
-------------------------------------------
On POSIX, the presence of the C run-time library is an unavoidable fact for
which reason it makes a lot of sense to directly call such C run-time
functions.


Sample "Hello World" Application
""""""""""""""""""""""""""""""""
On POSIX, it is really very easy to create the ``Hello world`` program:

.. code-block:: llvm

   declare i32 @puts(i8* nocapture) nounwind

   @.hello = private unnamed_addr constant [13 x i8] c"hello world\0A\00"

   define i32 @main(i32 %argc, i8** %argv) {
      %1 = getelementptr [13 x i8]* @.hello, i32 0, i32 0
      call i32 @puts(i8* %1)
      ret i32 0
   }


How to Interface to the Windows Operating System
------------------------------------------------
On Windows, the C run-time library is mostly considered of relevance to the
C and C++ languages only, so you have a plethora (thousands) of standard
system interfaces that any client application may use.


Sample "Hello World" Application
""""""""""""""""""""""""""""""""
``Hello world`` on Windows is nowhere as straightforward as on POSIX:

.. code-block:: llvm

   target datalayout = "e-p:32:32:32-i1:8:8-i8:8:8-i16:16:16-i32:32:32-i64:64:64-f32:32:32-f64:64:64-f80:128:128-v64:64:64-v128:128:128-a0:0:64-f80:32:32-n8:16:32-S32"
   target triple = "i686-pc-win32"

   %struct._OVERLAPPED = type { i32, i32, %union.anon, i8* }
   %union.anon = type { %struct.anon }
   %struct.anon = type { i32, i32 }

   declare dllimport x86_stdcallcc i8* @"\01_GetStdHandle@4"(i32) #1

   declare dllimport x86_stdcallcc i32 @"\01_WriteFile@20"(i8*, i8*, i32, i32*, %struct._OVERLAPPED*) #1

   @hello = internal constant [13 x i8] c"Hello world\0A\00"

   define i32 @main(i32 %argc, i8** %argv) nounwind {
      %1 = call i8* @"\01_GetStdHandle@4"(i32 -11)    ; -11 = STD_OUTPUT_HANDLE
      %2 = getelementptr [13 x i8]* @hello, i32 0, i32 0
      %3 = call i32 @"\01_WriteFile@20"(i8* %1, i8* %2, i32 12, i32* null, %struct._OVERLAPPED* null)
      ; todo: Check that %4 is not equal to -1 (INVALID_HANDLE_VALUE)
      ret i32 0
   }

   attributes #1 = { "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf"
      "no-infs-fp-math"="fa lse" "no-nans-fp-math"="false" "stack-protector-buffer-size"="8" "unsafe-fp-math"="false"
      "use-soft-float"="false"
   }


**TODO:**
What are the ``\01`` prefixes on the Windows names for?  Do they represent
Windows' way of exporting symbols or are they exclusive to Clang and LLVM?


Appendix A: How to Implement a String Type in LLVM
==================================================
There are two ways to implement a string type in LLVM:

#. To write the implementation in LLVM IR.
#. To write the implementation in a higher-level language that generates IR.

I'd personally much prefer to use the second method, but for the sake of the
example, I'll go ahead and illustrate a simple but useful string type in LLVM
IR.  It assumes a 32-bit architecture, so please replace all occurences of
``i32`` with ``i64`` if you are targetting a 64-bit architecture.

We'll be making a dynamic, mutable string type that can be appended to and
could also be inserted into, converted to lower case, and so on, depending on
which support functions are defined to operate on the string type.

It all boils down to making a suitable type definition for the class and then
define a rich set of functions to operate on the type definition:

.. code-block:: llvm

   ; The actual type definition for our 'String' type.
   %String = type {
      i8*,     ; buffer: pointer to the character buffer
      i32,     ; length: the number of chars in the buffer
      i32,     ; maxlen: the maximum number of chars in the buffer
      i32      ; factor: the number of chars to preallocate when growing
   }

   define fastcc void @String_Create_Default(%String* %this) nounwind {
      ; Initialize 'buffer'.
      %1 = getelementptr %String* %this, i32 0, i32 0
      store i8* null, i8** %1

      ; Initialize 'length'.
      %2 = getelementptr %String* %this, i32 0, i32 1
      store i32 0, i32* %2

      ; Initialize 'maxlen'.
      %3 = getelementptr %String* %this, i32 0, i32 2
      store i32 0, i32* %3

     ; Initialize 'factor'.
     %4 = getelementptr %String* %this, i32 0, i32 3
     store i32 16, i32* %4

     ret void
   }

   declare i8* @malloc(i32)
   declare void @free(i8*)
   declare i8* @memcpy(i8*, i8*, i32)

   define fastcc void @String_Delete(%String* %this) nounwind {
     ; Check if we need to call 'free'.
     %1 = getelementptr %String* %this, i32 0, i32 0
     %2 = load i8** %1
     %3 = icmp ne i8* %2, null
     br i1 %3, label %free_begin, label %free_close

   free_begin:
     call void @free(i8* %2)
     br label %free_close

   free_close:
     ret void
   }

   define fastcc void @String_Resize(%String* %this, i32 %value) {
      ; %output = malloc(%value)
      %output = call i8* @malloc(i32 %value)

      ; todo: check return value

      ; %buffer = this->buffer
      %1 = getelementptr %String* %this, i32 0, i32 0
      %buffer = load i8** %1

      ; %length = this->length
      %2 = getelementptr %String* %this, i32 0, i32 1
      %length = load i32* %2

      ; memcpy(%output, %buffer, %length)
      %3 = call i8* @memcpy(i8* %output, i8* %buffer, i32 %length)

      ; free(%buffer)
      call void @free(i8* %buffer)

      ; this->buffer = %output
      store i8* %output, i8** %1

      ret void
   }

   define fastcc void @String_Add_Char(%String* %this, i8 %value) {
     ; Check if we need to grow the string.
     %1 = getelementptr %String* %this, i32 0, i32 1
     %length = load i32* %1
     %2 = getelementptr %String* %this, i32 0, i32 2
     %maxlen = load i32* %2
     ; if length == maxlen:
     %3 = icmp eq i32 %length, %maxlen
     br i1 %3, label %grow_begin, label %grow_close

   grow_begin:
     %4 = getelementptr %String* %this, i32 0, i32 3
     %factor = load i32* %4
     %5 = add i32 %maxlen, %factor
     call void @String_Resize(%String* %this, i32 %5)
     br label %grow_close

   grow_close:
     %6 = getelementptr %String* %this, i32 0, i32 0
     %buffer = load i8** %6
     %7 = getelementptr i8* %buffer, i32 %length
     store i8 %value, i8* %7
     %8 = add i32 %length, 1
     store i32 %8, i32* %1

     ret void
   }


Resources
=========

#. Modern Compiler Implementation in Java, 2nd Edition.
#. `Alex Darby's series of articles on low-level stuff
   <http://www.altdevblogaday.com/author/alex-darby/>`_.


Epilogue
========
If you discover any errors in this document or you need more information
than given here, please write to the friendly `LLVM developers
<http://lists.cs.uiuc.edu/mailman/listinfo/llvmdev>`_ and they'll surely
help you out or add the requested info to this document.

Please also remember that you can learn a lot by using the ``-emit-llvm``
option to the ``clang++`` compiler.  This gives you a chance to see a live
production compiler in action and precisely how it does things.

