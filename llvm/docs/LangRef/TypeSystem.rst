====================
LangRef: Type System
====================

.. contents::
   :local:
   :depth: 3


.. _typesystem:

Type System
===========

The LLVM type system is one of the most important features of the
intermediate representation. Being typed enables a number of
optimizations to be performed on the intermediate representation
directly, without having to do extra analyses on the side before the
transformation. A strong type system makes it easier to read the
generated code and enables novel analyses and transformations that are
not feasible to perform on normal three address code representations.

.. _t_void:

Void Type
---------

:Overview:


The void type does not represent any value and has no size.

:Syntax:


::

      void


.. _t_function:

Function Type
-------------

:Overview:


The function type can be thought of as a function signature. It consists of a
return type and a list of formal parameter types. The return type of a function
type is a void type or first class type --- except for :ref:`label <t_label>`
and :ref:`metadata <t_metadata>` types.

:Syntax:

::

      <returntype> (<parameter list>)

...where '``<parameter list>``' is a comma-separated list of type
specifiers. Optionally, the parameter list may include a type ``...``, which
indicates that the function takes a variable number of arguments. Variable
argument functions can access their arguments with the :ref:`variable argument
handling intrinsic <int_varargs>` functions. '``<returntype>``' is any type
except :ref:`label <t_label>` and :ref:`metadata <t_metadata>`.

:Examples:

+---------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``i32 (i32)``                   | function taking an ``i32``, returning an ``i32``                                                                                                                    |
+---------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``i32 (ptr, ...)``              | A vararg function that takes at least one :ref:`pointer <t_pointer>` argument and returns an integer. This is the signature for ``printf`` in LLVM.                 |
+---------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``{i32, i32} (i32)``            | A function taking an ``i32``, returning a :ref:`structure <t_struct>` containing two ``i32`` values                                                                 |
+---------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+

.. _t_firstclass:

First Class Types
-----------------

The :ref:`first class <t_firstclass>` types are perhaps the most important.
Values of these types are the only ones which can be produced by
instructions.

.. _t_single_value:

Single Value Types
^^^^^^^^^^^^^^^^^^

These are the types that are valid in registers from CodeGen's perspective.

.. _t_integer:

Integer Type
""""""""""""

:Overview:

The integer type is a very simple type that simply specifies an
arbitrary bit width for the integer type desired. Any bit width from 1
bit to 2\ :sup:`23`\ (about 8 million) can be specified.

:Syntax:

::

      iN

The number of bits the integer will occupy is specified by the ``N``
value.

Examples:
*********

+----------------+------------------------------------------------+
| ``i1``         | a single-bit integer.                          |
+----------------+------------------------------------------------+
| ``i32``        | a 32-bit integer.                              |
+----------------+------------------------------------------------+
| ``i1942652``   | a really big integer of over 1 million bits.   |
+----------------+------------------------------------------------+

.. _t_floating:

Floating-Point Types
""""""""""""""""""""

.. list-table::
   :header-rows: 1

   * - Type
     - Description

   * - ``half``
     - 16-bit floating-point value (IEEE-754 binary16)

   * - ``bfloat``
     - 16-bit "brain" floating-point value (7-bit significand).  Provides the
       same number of exponent bits as ``float``, so that it matches its dynamic
       range, but with greatly reduced precision.  Used in Intel's AVX-512 BF16
       extensions and Arm's ARMv8.6-A extensions, among others.

   * - ``float``
     - 32-bit floating-point value (IEEE-754 binary32)

   * - ``double``
     - 64-bit floating-point value (IEEE-754 binary64)

   * - ``fp128``
     - 128-bit floating-point value (IEEE-754 binary128)

   * - ``x86_fp80``
     -  80-bit floating-point value (X87)

   * - ``ppc_fp128``
     - 128-bit floating-point value (two 64-bits)

X86_amx Type
""""""""""""

:Overview:

The x86_amx type represents a value held in an AMX tile register on an x86
machine. The operations allowed on it are quite limited. Only few intrinsics
are allowed: stride load and store, zero and dot product. No instruction is
allowed for this type. There are no arguments, arrays, pointers, vectors
or constants of this type.

:Syntax:

::

      x86_amx



.. _t_pointer:

Pointer Type
""""""""""""

:Overview:

The pointer type ``ptr`` is used to specify memory locations. Pointers are
commonly used to reference objects in memory.

Pointer types may have an optional address space attribute defining
the numbered address space where the pointed-to object resides. For
example, ``ptr addrspace(5)`` is a pointer to address space 5.
In addition to integer constants, ``addrspace`` can also reference one of the
address spaces defined in the :ref:`datalayout string<langref_datalayout>`.
``addrspace("A")`` will use the alloca address space, ``addrspace("G")``
the default globals address space and ``addrspace("P")`` the program address
space.

The default address space is number zero.

The semantics of non-zero address spaces are target-specific. Memory
access through a non-dereferenceable pointer is undefined behavior in
any address space. Pointers with the bit-value 0 are only assumed to
be non-dereferenceable in address space 0, unless the function is
marked with the ``null_pointer_is_valid`` attribute.

If an object can be proven accessible through a pointer with a
different address space, the access may be modified to use that
address space. Exceptions apply if the operation is ``volatile``.

Prior to LLVM 15, pointer types also specified a pointee type, such as
``i8*``, ``[4 x i32]*`` or ``i32 (i32*)*``. In LLVM 15, such "typed
pointers" are still supported under non-default options. See the
`opaque pointers document <../OpaquePointers.html>`__ for more information.

.. _t_target_type:

Target Extension Type
"""""""""""""""""""""

:Overview:

Target extension types represent types that must be preserved through
optimization, but are otherwise generally opaque to the compiler. They may be
used as function parameters or arguments, and in :ref:`phi <i_phi>` or
:ref:`select <i_select>` instructions. Some types may be also used in
:ref:`alloca <i_alloca>` instructions or as global values, and correspondingly
it is legal to use :ref:`load <i_load>` and :ref:`store <i_store>` instructions
on them. Full semantics for these types are defined by the target.

The only constants that target extension types may have are ``zeroinitializer``,
``undef``, and ``poison``. Other possible values for target extension types may
arise from target-specific intrinsics and functions.

These types cannot be converted to other types. As such, it is not legal to use
them in :ref:`bitcast <i_bitcast>` instructions (as a source or target type),
nor is it legal to use them in :ref:`ptrtoint <i_ptrtoint>` or
:ref:`inttoptr <i_inttoptr>` instructions. Similarly, they are not legal to use
in an :ref:`icmp <i_icmp>` instruction.

Target extension types have a name and optional type or integer parameters. The
meanings of name and parameters are defined by the target. When being defined in
LLVM IR, all of the type parameters must precede all of the integer parameters.

Specific target extension types are registered with LLVM as having specific
properties. These properties can be used to restrict the type from appearing in
certain contexts, such as being the type of a global variable or having a
``zeroinitializer`` constant be valid. A complete list of type properties may be
found in the documentation for ``llvm::TargetExtType::Property`` (`doxygen
<https://llvm.org/doxygen/classllvm_1_1TargetExtType.html>`_).

:Syntax:

.. code-block:: llvm

      target("label")
      target("label", void)
      target("label", void, i32)
      target("label", 0, 1, 2)
      target("label", void, i32, 0, 1, 2)


.. _t_vector:

Vector Type
"""""""""""

:Overview:

A vector type is a simple derived type that represents a vector of
elements. Vector types are used when multiple primitive data are
operated in parallel using a single instruction (SIMD). A vector type
requires a size (number of elements), an underlying primitive data type,
and a scalable property to represent vectors where the exact hardware
vector length is unknown at compile time. Vector types are considered
:ref:`first class <t_firstclass>`.

:Memory Layout:

In general vector elements are laid out in memory in the same way as
:ref:`array types <t_array>`. Such an analogy works fine as long as the vector
elements are byte sized. However, when the elements of the vector aren't byte
sized it gets a bit more complicated. One way to describe the layout is by
describing what happens when a vector such as <N x iM> is bitcasted to an
integer type with N*M bits, and then following the rules for storing such an
integer to memory.

A bitcast from a vector type to a scalar integer type will see the elements
being packed together (without padding). The order in which elements are
inserted in the integer depends on endianness. For little endian element zero
is put in the least significant bits of the integer, and for big endian
element zero is put in the most significant bits.

Using a vector such as ``<i4 1, i4 2, i4 3, i4 5>`` as an example, together
with the analogy that we can replace a vector store by a bitcast followed by
an integer store, we get this for big endian:

.. code-block:: llvm

      %val = bitcast <4 x i4> <i4 1, i4 2, i4 3, i4 5> to i16

      ; Bitcasting from a vector to an integral type can be seen as
      ; concatenating the values:
      ;   %val now has the hexadecimal value 0x1235.

      store i16 %val, ptr %ptr

      ; In memory the content will be (8-bit addressing):
      ;
      ;    [%ptr + 0]: 00010010  (0x12)
      ;    [%ptr + 1]: 00110101  (0x35)

The same example for little endian:

.. code-block:: llvm

      %val = bitcast <4 x i4> <i4 1, i4 2, i4 3, i4 5> to i16

      ; Bitcasting from a vector to an integral type can be seen as
      ; concatenating the values:
      ;   %val now has the hexadecimal value 0x5321.

      store i16 %val, ptr %ptr

      ; In memory the content will be (8-bit addressing):
      ;
      ;    [%ptr + 0]: 00100001  (0x21)
      ;    [%ptr + 1]: 01010011  (0x53)

When ``<N*M>`` isn't evenly divisible by the byte size the exact memory layout
is unspecified (just like it is for an integral type of the same size). This
is because different targets could put the padding at different positions when
the type size is smaller than the type's store size.

:Syntax:

::

      < <# elements> x <elementtype> >          ; Fixed-length vector
      < vscale x <# elements> x <elementtype> > ; Scalable vector

The number of elements is a constant integer value larger than 0;
elementtype may be any integer, floating-point or pointer type. Vectors
of size zero are not allowed. For scalable vectors, the total number of
elements is a constant multiple (called vscale) of the specified number
of elements; vscale is a positive integer that is unknown at compile time
and the same hardware-dependent constant for all scalable vectors at run
time. The size of a specific scalable vector type is thus constant within
IR, even if the exact size in bytes cannot be determined until run time.

:Examples:

+------------------------+----------------------------------------------------+
| ``<4 x i32>``          | Vector of 4 32-bit integer values.                 |
+------------------------+----------------------------------------------------+
| ``<8 x float>``        | Vector of 8 32-bit floating-point values.          |
+------------------------+----------------------------------------------------+
| ``<2 x i64>``          | Vector of 2 64-bit integer values.                 |
+------------------------+----------------------------------------------------+
| ``<4 x ptr>``          | Vector of 4 pointers                               |
+------------------------+----------------------------------------------------+
| ``<vscale x 4 x i32>`` | Vector with a multiple of 4 32-bit integer values. |
+------------------------+----------------------------------------------------+

.. _t_label:

Label Type
^^^^^^^^^^

:Overview:

The label type represents code labels.

:Syntax:

::

      label

.. _t_token:

Token Type
^^^^^^^^^^

:Overview:

The token type is used when a value is associated with an instruction
but all uses of the value must not attempt to introspect or obscure it.
As such, it is not appropriate to have a :ref:`phi <i_phi>` or
:ref:`select <i_select>` of type token.

:Syntax:

::

      token



.. _t_metadata:

Metadata Type
^^^^^^^^^^^^^

:Overview:

The metadata type represents embedded metadata. No derived types may be
created from metadata except for :ref:`function <t_function>` arguments.

:Syntax:

::

      metadata

.. _t_aggregate:

Aggregate Types
^^^^^^^^^^^^^^^

Aggregate Types are a subset of derived types that can contain multiple
member types. :ref:`Arrays <t_array>` and :ref:`structs <t_struct>` are
aggregate types. :ref:`Vectors <t_vector>` are not considered to be
aggregate types.

.. _t_array:

Array Type
""""""""""

:Overview:

The array type is a very simple derived type that arranges elements
sequentially in memory. The array type requires a size (number of
elements) and an underlying data type.

:Syntax:

::

      [<# elements> x <elementtype>]

The number of elements is a constant integer value; ``elementtype`` may
be any type with a size.

:Examples:

+------------------+--------------------------------------+
| ``[40 x i32]``   | Array of 40 32-bit integer values.   |
+------------------+--------------------------------------+
| ``[41 x i32]``   | Array of 41 32-bit integer values.   |
+------------------+--------------------------------------+
| ``[4 x i8]``     | Array of 4 8-bit integer values.     |
+------------------+--------------------------------------+

Here are some examples of multidimensional arrays:

+-----------------------------+----------------------------------------------------------+
| ``[3 x [4 x i32]]``         | 3x4 array of 32-bit integer values.                      |
+-----------------------------+----------------------------------------------------------+
| ``[12 x [10 x float]]``     | 12x10 array of single precision floating-point values.   |
+-----------------------------+----------------------------------------------------------+
| ``[2 x [3 x [4 x i16]]]``   | 2x3x4 array of 16-bit integer values.                    |
+-----------------------------+----------------------------------------------------------+

There is no restriction on indexing beyond the end of the array implied
by a static type (though there are restrictions on indexing beyond the
bounds of an :ref:`allocated object<allocatedobjects>` in some cases). This
means that single-dimension 'variable sized array' addressing can be implemented
in LLVM with a zero length array type. An implementation of 'pascal style
arrays' in LLVM could use the type "``{ i32, [0 x float]}``", for example.

.. _t_struct:

Structure Type
""""""""""""""

:Overview:

The structure type is used to represent a collection of data members
together in memory. The elements of a structure may be any type that has
a size.

Structures in memory are accessed using '``load``' and '``store``' by
getting a pointer to a field with the '``getelementptr``' instruction.
Structures in registers are accessed using the '``extractvalue``' and
'``insertvalue``' instructions.

Structures may optionally be "packed" structures, which indicate that
the alignment of the struct is one byte, and that there is no padding
between the elements. In non-packed structs, padding between field types
is inserted as defined by the DataLayout string in the module, which is
required to match what the underlying code generator expects.

Structures can either be "literal" or "identified". A literal structure
is defined inline with other types (e.g. ``[2 x {i32, i32}]``) whereas
identified types are always defined at the top level with a name.
Literal types are uniqued by their contents and can never be recursive
or opaque since there is no way to write one. Identified types can be
opaqued and are never uniqued. Identified types must not be recursive.

:Syntax:

::

      %T1 = type { <type list> }     ; Identified normal struct type
      %T2 = type <{ <type list> }>   ; Identified packed struct type

:Examples:

+------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``{ i32, i32, i32 }``        | A triple of three ``i32`` values (this is a "homogeneous" struct as all element types are the same)                                                                                   |
+------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``{ float, ptr }``           | A pair, where the first element is a ``float`` and the second element is a :ref:`pointer <t_pointer>`.                                                                                |
+------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``<{ i8, i32 }>``            | A packed struct known to be 5 bytes in size.                                                                                                                                          |
+------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

.. _t_opaque:

Opaque Structure Types
""""""""""""""""""""""

:Overview:

Opaque structure types are used to represent structure types that
do not have a body specified. This corresponds (for example) to the C
notion of a forward declared structure. They can be named (``%X``) or
unnamed (``%52``).

:Syntax:

::

      %X = type opaque
      %52 = type opaque

:Examples:

+--------------+-------------------+
| ``opaque``   | An opaque type.   |
+--------------+-------------------+

