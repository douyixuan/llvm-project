============================
LangRef: Intrinsic Functions
============================

.. contents::
   :local:
   :depth: 3


.. _intrinsics:

Intrinsic Functions
===================

LLVM supports the notion of an "intrinsic function". These functions
have well known names and semantics and are required to follow certain
restrictions. Overall, these intrinsics represent an extension mechanism
for the LLVM language that does not require changing all of the
transformations in LLVM when adding to the language (or the bitcode
reader/writer, the parser, etc...).

Intrinsic function names must all start with an "``llvm.``" prefix. This
prefix is reserved in LLVM for intrinsic names; thus, function names may
not begin with this prefix. Intrinsic functions must always be external
functions: you cannot define the body of intrinsic functions. Intrinsic
functions may only be used in call or invoke instructions: it is illegal
to take the address of an intrinsic function. Additionally, because
intrinsic functions are part of the LLVM language, it is required if any
are added that they be documented here.

Some intrinsic functions can be overloaded, i.e., the intrinsic
represents a family of functions that perform the same operation but on
different data types. Because LLVM can represent over 8 million
different integer types, overloading is used commonly to allow an
intrinsic function to operate on any integer type. One or more of the
argument types or the result type can be overloaded to accept any
integer type. Argument types may also be defined as exactly matching a
previous argument's type or the result type. This allows an intrinsic
function which accepts multiple arguments, but needs all of them to be
of the same type, to only be overloaded with respect to a single
argument or the result.

Overloaded intrinsics will have the names of its overloaded argument
types encoded into its function name, each preceded by a period. Only
those types which are overloaded result in a name suffix. Arguments
whose type is matched against another type do not. For example, the
``llvm.ctpop`` function can take an integer of any width and returns an
integer of exactly the same integer width. This leads to a family of
functions such as ``i8 @llvm.ctpop.i8(i8 %val)`` and
``i29 @llvm.ctpop.i29(i29 %val)``. Only one type, the return type, is
overloaded, and only one type suffix is required. Because the argument's
type is matched against the return type, it does not require its own
name suffix.

:ref:`Unnamed types <t_opaque>` are encoded as ``s_s``. Overloaded intrinsics
that depend on an unnamed type in one of its overloaded argument types get an
additional ``.<number>`` suffix. This allows differentiating intrinsics with
different unnamed types as arguments. (For example:
``llvm.ssa.copy.p0s_s.2(%42*)``) The number is tracked in the LLVM module and
it ensures unique names in the module. While linking together two modules, it is
still possible to get a name clash. In that case one of the names will be
changed by getting a new number.

For target developers who are defining intrinsics for back-end code
generation, any intrinsic overloads based solely the distinction between
integer or floating point types should not be relied upon for correct
code generation. In such cases, the recommended approach for target
maintainers when defining intrinsics is to create separate integer and
FP intrinsics rather than rely on overloading. For example, if different
codegen is required for ``llvm.target.foo(<4 x i32>)`` and
``llvm.target.foo(<4 x float>)`` then these should be split into
different intrinsics.

To learn how to add an intrinsic function, please see the `Extending
LLVM Guide <../ExtendingLLVM.html>`_.

.. _int_varargs:

Variable Argument Handling Intrinsics
-------------------------------------

Variable argument support is defined in LLVM with the
:ref:`va_arg <i_va_arg>` instruction and these three intrinsic
functions. These functions are related to the similarly named macros
defined in the ``<stdarg.h>`` header file.

All of these functions take as arguments pointers to a target-specific
value type "``va_list``". The LLVM assembly language reference manual
does not define what this type is, so all transformations should be
prepared to handle these functions regardless of the type used. The intrinsics
are overloaded, and can be used for pointers to different address spaces.

This example shows how the :ref:`va_arg <i_va_arg>` instruction and the
variable argument handling intrinsic functions are used.

.. code-block:: llvm

    ; This struct is different for every platform. For most platforms,
    ; it is merely a ptr.
    %struct.va_list = type { ptr }

    ; For Unix x86_64 platforms, va_list is the following struct:
    ; %struct.va_list = type { i32, i32, ptr, ptr }

    define i32 @test(i32 %X, ...) {
      ; Initialize variable argument processing
      %ap = alloca %struct.va_list
      call void @llvm.va_start.p0(ptr %ap)

      ; Read a single integer argument
      %tmp = va_arg ptr %ap, i32

      ; Demonstrate usage of llvm.va_copy and llvm.va_end
      %aq = alloca ptr
      call void @llvm.va_copy.p0(ptr %aq, ptr %ap)
      call void @llvm.va_end.p0(ptr %aq)

      ; Stop processing of arguments.
      call void @llvm.va_end.p0(ptr %ap)
      ret i32 %tmp
    }

    declare void @llvm.va_start.p0(ptr)
    declare void @llvm.va_copy.p0(ptr, ptr)
    declare void @llvm.va_end.p0(ptr)

.. _int_va_start:

'``llvm.va_start``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.va_start.p0(ptr <arglist>)
      declare void @llvm.va_start.p5(ptr addrspace(5) <arglist>)

Overview:
"""""""""

The '``llvm.va_start``' intrinsic initializes ``<arglist>`` for
subsequent use by ``va_arg``.

Arguments:
""""""""""

The argument is a pointer to a ``va_list`` element to initialize.

Semantics:
""""""""""

The '``llvm.va_start``' intrinsic works just like the ``va_start`` macro
available in C. In a target-dependent way, it initializes the
``va_list`` element to which the argument points, so that the next call
to ``va_arg`` will produce the first variable argument passed to the
function. Unlike the C ``va_start`` macro, this intrinsic does not need
to know the last argument of the function as the compiler can figure
that out.

'``llvm.va_end``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.va_end.p0(ptr <arglist>)
      declare void @llvm.va_end.p5(ptr addrspace(5) <arglist>)

Overview:
"""""""""

The '``llvm.va_end``' intrinsic destroys ``<arglist>``, which has been
initialized previously with ``llvm.va_start`` or ``llvm.va_copy``.

Arguments:
""""""""""

The argument is a pointer to a ``va_list`` to destroy.

Semantics:
""""""""""

The '``llvm.va_end``' intrinsic works just like the ``va_end`` macro
available in C. In a target-dependent way, it destroys the ``va_list``
element to which the argument points. Calls to
:ref:`llvm.va_start <int_va_start>` and
:ref:`llvm.va_copy <int_va_copy>` must be matched exactly with calls to
``llvm.va_end``.

.. _int_va_copy:

'``llvm.va_copy``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.va_copy.p0(ptr <destarglist>, ptr <srcarglist>)
      declare void @llvm.va_copy.p5(ptr addrspace(5) <destarglist>, ptr addrspace(5) <srcarglist>)

Overview:
"""""""""

The '``llvm.va_copy``' intrinsic copies the current argument position
from the source argument list to the destination argument list.

Arguments:
""""""""""

The first argument is a pointer to a ``va_list`` element to initialize.
The second argument is a pointer to a ``va_list`` element to copy from.
The address spaces of the two arguments must match.

Semantics:
""""""""""

The '``llvm.va_copy``' intrinsic works just like the ``va_copy`` macro
available in C. In a target-dependent way, it copies the source
``va_list`` element into the destination ``va_list`` element. This
intrinsic is necessary because the `` llvm.va_start`` intrinsic may be
arbitrarily complex and require, for example, memory allocation.

Accurate Garbage Collection Intrinsics
--------------------------------------

LLVM's support for `Accurate Garbage Collection <../GarbageCollection.html>`_
(GC) requires the frontend to generate code containing appropriate intrinsic
calls and select an appropriate GC strategy which knows how to lower these
intrinsics in a manner which is appropriate for the target collector.

These intrinsics allow identification of :ref:`GC roots on the
stack <int_gcroot>`, as well as garbage collector implementations that
require :ref:`read <int_gcread>` and :ref:`write <int_gcwrite>` barriers.
Frontends for type-safe garbage collected languages should generate
these intrinsics to make use of the LLVM garbage collectors. For more
details, see `Garbage Collection with LLVM <../GarbageCollection.html>`_.

LLVM provides an second experimental set of intrinsics for describing garbage
collection safepoints in compiled code. These intrinsics are an alternative
to the ``llvm.gcroot`` intrinsics, but are compatible with the ones for
:ref:`read <int_gcread>` and :ref:`write <int_gcwrite>` barriers. The
differences in approach are covered in the `Garbage Collection with LLVM
<../GarbageCollection.html>`_ documentation. The intrinsics themselves are
described in :doc:`../Statepoints`.

.. _int_gcroot:

'``llvm.gcroot``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.gcroot(ptr %ptrloc, ptr %metadata)

Overview:
"""""""""

The '``llvm.gcroot``' intrinsic declares the existence of a GC root to
the code generator, and allows some metadata to be associated with it.

Arguments:
""""""""""

The first argument specifies the address of a stack object that contains
the root pointer. The second pointer (which must be either a constant or
a global value address) contains the meta-data to be associated with the
root.

Semantics:
""""""""""

At runtime, a call to this intrinsic stores a null pointer into the
"ptrloc" location. At compile-time, the code generator generates
information to allow the runtime to find the pointer at GC safe points.
The '``llvm.gcroot``' intrinsic may only be used in a function which
:ref:`specifies a GC algorithm <gc>`.

.. _int_gcread:

'``llvm.gcread``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare ptr @llvm.gcread(ptr %ObjPtr, ptr %Ptr)

Overview:
"""""""""

The '``llvm.gcread``' intrinsic identifies reads of references from heap
locations, allowing garbage collector implementations that require read
barriers.

Arguments:
""""""""""

The second argument is the address to read from, which should be an
address allocated from the garbage collector. The first object is a
pointer to the start of the referenced object, if needed by the language
runtime (otherwise null).

Semantics:
""""""""""

The '``llvm.gcread``' intrinsic has the same semantics as a load
instruction, but may be replaced with substantially more complex code by
the garbage collector runtime, as needed. The '``llvm.gcread``'
intrinsic may only be used in a function which :ref:`specifies a GC
algorithm <gc>`.

.. _int_gcwrite:

'``llvm.gcwrite``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.gcwrite(ptr %P1, ptr %Obj, ptr %P2)

Overview:
"""""""""

The '``llvm.gcwrite``' intrinsic identifies writes of references to heap
locations, allowing garbage collector implementations that require write
barriers (such as generational or reference counting collectors).

Arguments:
""""""""""

The first argument is the reference to store, the second is the start of
the object to store it to, and the third is the address of the field of
Obj to store to. If the runtime does not require a pointer to the
object, Obj may be null.

Semantics:
""""""""""

The '``llvm.gcwrite``' intrinsic has the same semantics as a store
instruction, but may be replaced with substantially more complex code by
the garbage collector runtime, as needed. The '``llvm.gcwrite``'
intrinsic may only be used in a function which :ref:`specifies a GC
algorithm <gc>`.


.. _gc_statepoint:

'``llvm.experimental.gc.statepoint``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare token
        @llvm.experimental.gc.statepoint(i64 <id>, i32 <num patch bytes>,
                       ptr elementtype(func_type) <target>,
                       i64 <#call args>, i64 <flags>,
                       ... (call parameters),
                       i64 0, i64 0)

Overview:
"""""""""

The statepoint intrinsic represents a call which is parse-able by the
runtime.

Operands:
"""""""""

The 'id' operand is a constant integer that is reported as the ID
field in the generated stackmap.  LLVM does not interpret this
parameter in any way and its meaning is up to the statepoint user to
decide.  Note that LLVM is free to duplicate code containing
statepoint calls, and this may transform IR that had a unique 'id' per
lexical call to statepoint to IR that does not.

If 'num patch bytes' is non-zero then the call instruction
corresponding to the statepoint is not emitted and LLVM emits 'num
patch bytes' bytes of nops in its place.  LLVM will emit code to
prepare the function arguments and retrieve the function return value
in accordance to the calling convention; the former before the nop
sequence and the latter after the nop sequence.  It is expected that
the user will patch over the 'num patch bytes' bytes of nops with a
calling sequence specific to their runtime before executing the
generated machine code.  There are no guarantees with respect to the
alignment of the nop sequence.  Unlike :doc:`../StackMaps` statepoints do
not have a concept of shadow bytes.  Note that semantically the
statepoint still represents a call or invoke to 'target', and the nop
sequence after patching is expected to represent an operation
equivalent to a call or invoke to 'target'.

The 'target' operand is the function actually being called. The operand
must have an :ref:`elementtype <attr_elementtype>` attribute specifying
the function type of the target. The target can be specified as either
a symbolic LLVM function, or as an arbitrary Value of pointer type. Note
that the function type must match the signature of the callee and the
types of the 'call parameters' arguments.

The '#call args' operand is the number of arguments to the actual
call.  It must exactly match the number of arguments passed in the
'call parameters' variable length section.

The 'flags' operand is used to specify extra information about the
statepoint. This is currently only used to mark certain statepoints
as GC transitions. This operand is a 64-bit integer with the following
layout, where bit 0 is the least significant bit:

  +-------+---------------------------------------------------+
  | Bit # | Usage                                             |
  +=======+===================================================+
  |     0 | Set if the statepoint is a GC transition, cleared |
  |       | otherwise.                                        |
  +-------+---------------------------------------------------+
  |  1-63 | Reserved for future use; must be cleared.         |
  +-------+---------------------------------------------------+

The 'call parameters' arguments are simply the arguments which need to
be passed to the call target.  They will be lowered according to the
specified calling convention and otherwise handled like a normal call
instruction.  The number of arguments must exactly match what is
specified in '# call args'.  The types must match the signature of
'target'.

The 'call parameter' attributes must be followed by two 'i64 0' constants.
These were originally the length prefixes for 'gc transition parameter' and
'deopt parameter' arguments, but the role of these parameter sets have been
entirely replaced with the corresponding operand bundles.  In a future
revision, these now redundant arguments will be removed.

Semantics:
""""""""""

A statepoint is assumed to read and write all memory.  As a result,
memory operations can not be reordered past a statepoint.  It is
illegal to mark a statepoint as being either 'readonly' or 'readnone'.

Note that legal IR can not perform any memory operation on a 'gc
pointer' argument of the statepoint in a location statically reachable
from the statepoint.  Instead, the explicitly relocated value (from a
``gc.relocate``) must be used.

'``llvm.experimental.gc.result``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare type
        @llvm.experimental.gc.result(token %statepoint_token)

Overview:
"""""""""

``gc.result`` extracts the result of the original call instruction
which was replaced by the ``gc.statepoint``.  The ``gc.result``
intrinsic is actually a family of three intrinsics due to an
implementation limitation.  Other than the type of the return value,
the semantics are the same.

Operands:
"""""""""

The first and only argument is the ``gc.statepoint`` which starts
the safepoint sequence of which this ``gc.result`` is a part.
Despite the typing of this as a generic token, *only* the value defined
by a ``gc.statepoint`` is legal here.

Semantics:
""""""""""

The ``gc.result`` represents the return value of the call target of
the ``statepoint``.  The type of the ``gc.result`` must exactly match
the type of the target.  If the call target returns void, there will
be no ``gc.result``.

A ``gc.result`` is modeled as a 'readnone' pure function.  It has no
side effects since it is just a projection of the return value of the
previous call represented by the ``gc.statepoint``.

'``llvm.experimental.gc.relocate``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <pointer type>
        @llvm.experimental.gc.relocate(token %statepoint_token,
                                       i32 %base_offset,
                                       i32 %pointer_offset)

Overview:
"""""""""

A ``gc.relocate`` returns the potentially relocated value of a pointer
at the safepoint.

Operands:
"""""""""

The first argument is the ``gc.statepoint`` which starts the
safepoint sequence of which this ``gc.relocation`` is a part.
Despite the typing of this as a generic token, *only* the value defined
by a ``gc.statepoint`` is legal here.

The second and third arguments are both indices into operands of the
corresponding statepoint's :ref:`gc-live <ob_gc_live>` operand bundle.

The second argument is an index which specifies the allocation for the pointer
being relocated. The associated value must be within the object with which the
pointer being relocated is associated. The optimizer is free to change *which*
interior derived pointer is reported, provided that it does not replace an
actual base pointer with another interior derived pointer. Collectors are
allowed to rely on the base pointer operand remaining an actual base pointer if
so constructed.

The third argument is an index which specify the (potentially) derived pointer
being relocated.  It is legal for this index to be the same as the second
argument if-and-only-if a base pointer is being relocated.

Semantics:
""""""""""

The return value of ``gc.relocate`` is the potentially relocated value
of the pointer specified by its arguments.  It is unspecified how the
value of the returned pointer relates to the argument to the
``gc.statepoint`` other than that a) it points to the same source
language object with the same offset, and b) the 'based-on'
relationship of the newly relocated pointers is a projection of the
unrelocated pointers.  In particular, the integer value of the pointer
returned is unspecified.

A ``gc.relocate`` is modeled as a ``readnone`` pure function.  It has no
side effects since it is just a way to extract information about work
done during the actual call modeled by the ``gc.statepoint``.

.. _gc.get.pointer.base:

'``llvm.experimental.gc.get.pointer.base``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <pointer type>
        @llvm.experimental.gc.get.pointer.base(
          <pointer type> readnone captures(none) %derived_ptr)
          nounwind willreturn memory(none)

Overview:
"""""""""

``gc.get.pointer.base`` for a derived pointer returns its base pointer.

Operands:
"""""""""

The only argument is a pointer which is based on some object with
an unknown offset from the base of said object.

Semantics:
""""""""""

This intrinsic is used in the abstract machine model for GC to represent
the base pointer for an arbitrary derived pointer.

This intrinsic is inlined by the :ref:`RewriteStatepointsForGC` pass by
replacing all uses of this callsite with the offset of a derived pointer from
its base pointer value. The replacement is done as part of the lowering to the
explicit statepoint model.

The return pointer type must be the same as the type of the parameter.


'``llvm.experimental.gc.get.pointer.offset``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i64
        @llvm.experimental.gc.get.pointer.offset(
          <pointer type> readnone captures(none) %derived_ptr)
          nounwind willreturn memory(none)

Overview:
"""""""""

``gc.get.pointer.offset`` for a derived pointer returns the offset from its
base pointer.

Operands:
"""""""""

The only argument is a pointer which is based on some object with
an unknown offset from the base of said object.

Semantics:
""""""""""

This intrinsic is used in the abstract machine model for GC to represent
the offset of an arbitrary derived pointer from its base pointer.

This intrinsic is inlined by the :ref:`RewriteStatepointsForGC` pass by
replacing all uses of this callsite with the offset of a derived pointer from
its base pointer value. The replacement is done as part of the lowering to the
explicit statepoint model.

Basically this call calculates difference between the derived pointer and its
base pointer (see :ref:`gc.get.pointer.base`) both ptrtoint casted. But
this cast done outside the :ref:`RewriteStatepointsForGC` pass could result
in the pointers lost for further lowering from the abstract model to the
explicit physical one.

Code Generator Intrinsics
-------------------------

These intrinsics are provided by LLVM to expose special features that
may only be implemented with code generator support.

'``llvm.returnaddress``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare ptr @llvm.returnaddress(i32 <level>)

Overview:
"""""""""

The '``llvm.returnaddress``' intrinsic attempts to compute a
target-specific value indicating the return address of the current
function or one of its callers.

Arguments:
""""""""""

The argument to this intrinsic indicates which function to return the
address for. Zero indicates the calling function, one indicates its
caller, etc. The argument is **required** to be a constant integer
value.

Semantics:
""""""""""

The '``llvm.returnaddress``' intrinsic either returns a pointer
indicating the return address of the specified call frame, or zero if it
cannot be identified. The value returned by this intrinsic is likely to
be incorrect or 0 for arguments other than zero, so it should only be
used for debugging purposes.

Note that calling this intrinsic does not prevent function inlining or
other aggressive transformations, so the value returned may not be that
of the obvious source-language caller.

'``llvm.addressofreturnaddress``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare ptr @llvm.addressofreturnaddress()

Overview:
"""""""""

The '``llvm.addressofreturnaddress``' intrinsic returns a target-specific
pointer to the place in the stack frame where the return address of the
current function is stored.

Semantics:
""""""""""

Note that calling this intrinsic does not prevent function inlining or
other aggressive transformations, so the value returned may not be that
of the obvious source-language caller.

This intrinsic is only implemented for x86 and aarch64.

'``llvm.sponentry``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare ptr @llvm.sponentry()

Overview:
"""""""""

The '``llvm.sponentry``' intrinsic returns the stack pointer value at
the entry of the current function calling this intrinsic.

Semantics:
""""""""""

Note this intrinsic is only verified on AArch64 and ARM.

'``llvm.frameaddress``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare ptr @llvm.frameaddress(i32 <level>)

Overview:
"""""""""

The '``llvm.frameaddress``' intrinsic attempts to return the
target-specific frame pointer value for the specified stack frame.

Arguments:
""""""""""

The argument to this intrinsic indicates which function to return the
frame pointer for. Zero indicates the calling function, one indicates
its caller, etc. The argument is **required** to be a constant integer
value.

Semantics:
""""""""""

The '``llvm.frameaddress``' intrinsic either returns a pointer
indicating the frame address of the specified call frame, or zero if it
cannot be identified. The value returned by this intrinsic is likely to
be incorrect or 0 for arguments other than zero, so it should only be
used for debugging purposes.

Note that calling this intrinsic does not prevent function inlining or
other aggressive transformations, so the value returned may not be that
of the obvious source-language caller.

'``llvm.swift.async.context.addr``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare ptr @llvm.swift.async.context.addr()

Overview:
"""""""""

The '``llvm.swift.async.context.addr``' intrinsic returns a pointer to
the part of the extended frame record containing the asynchronous
context of a Swift execution.

Semantics:
""""""""""

If the caller has a ``swiftasync`` parameter, that argument will initially
be stored at the returned address. If not, it will be initialized to null.

'``llvm.localescape``' and '``llvm.localrecover``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.localescape(...)
      declare ptr @llvm.localrecover(ptr %func, ptr %fp, i32 %idx)

Overview:
"""""""""

The '``llvm.localescape``' intrinsic escapes offsets of a collection of static
allocas, and the '``llvm.localrecover``' intrinsic applies those offsets to a
live frame pointer to recover the address of the allocation. The offset is
computed during frame layout of the caller of ``llvm.localescape``.

Arguments:
""""""""""

All arguments to '``llvm.localescape``' must be pointers to static allocas or
casts of static allocas. Each function can only call '``llvm.localescape``'
once, and it can only do so from the entry block.

The ``func`` argument to '``llvm.localrecover``' must be a constant
bitcasted pointer to a function defined in the current module. The code
generator cannot determine the frame allocation offset of functions defined in
other modules.

The ``fp`` argument to '``llvm.localrecover``' must be a frame pointer of a
call frame that is currently live. The return value of '``llvm.localaddress``'
is one way to produce such a value, but various runtimes also expose a suitable
pointer in platform-specific ways.

The ``idx`` argument to '``llvm.localrecover``' indicates which alloca passed to
'``llvm.localescape``' to recover. It is zero-indexed.

Semantics:
""""""""""

These intrinsics allow a group of functions to share access to a set of local
stack allocations of a one parent function. The parent function may call the
'``llvm.localescape``' intrinsic once from the function entry block, and the
child functions can use '``llvm.localrecover``' to access the escaped allocas.
The '``llvm.localescape``' intrinsic blocks inlining, as inlining changes where
the escaped allocas are allocated, which would break attempts to use
'``llvm.localrecover``'.

'``llvm.seh.try.begin``' and '``llvm.seh.try.end``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.seh.try.begin()
      declare void @llvm.seh.try.end()

Overview:
"""""""""

The '``llvm.seh.try.begin``' and '``llvm.seh.try.end``' intrinsics mark
the boundary of a _try region for Windows SEH Asynchronous Exception Handling.

Semantics:
""""""""""

When a C-function is compiled with Windows SEH Asynchronous Exception option,
-feh_asynch (aka MSVC -EHa), these two intrinsics are injected to mark _try
boundary and to prevent potential exceptions from being moved across boundary.
Any set of operations can then be confined to the region by reading their leaf
inputs via volatile loads and writing their root outputs via volatile stores.

'``llvm.seh.scope.begin``' and '``llvm.seh.scope.end``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.seh.scope.begin()
      declare void @llvm.seh.scope.end()

Overview:
"""""""""

The '``llvm.seh.scope.begin``' and '``llvm.seh.scope.end``' intrinsics mark
the boundary of a CPP object lifetime for Windows SEH Asynchronous Exception
Handling (MSVC option -EHa).

Semantics:
""""""""""

LLVM's ordinary exception-handling representation associates EH cleanups and
handlers only with ``invoke``s, which normally correspond only to call sites.  To
support arbitrary faulting instructions, it must be possible to recover the current
EH scope for any instruction.  Turning every operation in LLVM that could fault
into an ``invoke`` of a new, potentially-throwing intrinsic would require adding a
large number of intrinsics, impede optimization of those operations, and make
compilation slower by introducing many extra basic blocks.  These intrinsics can
be used instead to mark the region protected by a cleanup, such as for a local
C++ object with a non-trivial destructor.  ``llvm.seh.scope.begin`` is used to mark
the start of the region; it is always called with ``invoke``, with the unwind block
being the desired unwind destination for any potentially-throwing instructions
within the region.  `llvm.seh.scope.end` is used to mark when the scope ends
and the EH cleanup is no longer required (e.g. because the destructor is being
called).

.. _int_read_register:
.. _int_read_volatile_register:
.. _int_write_register:

'``llvm.read_register``', '``llvm.read_volatile_register``', and '``llvm.write_register``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i32 @llvm.read_register.i32(metadata)
      declare i64 @llvm.read_register.i64(metadata)
      declare i32 @llvm.read_volatile_register.i32(metadata)
      declare i64 @llvm.read_volatile_register.i64(metadata)
      declare void @llvm.write_register.i32(metadata, i32 @value)
      declare void @llvm.write_register.i64(metadata, i64 @value)
      !0 = !{!"sp\00"}

Overview:
"""""""""

The '``llvm.read_register``', '``llvm.read_volatile_register``', and
'``llvm.write_register``' intrinsics provide access to the named register.
The register must be valid on the architecture being compiled to. The type
needs to be compatible with the register being read.

Semantics:
""""""""""

The '``llvm.read_register``' and '``llvm.read_volatile_register``' intrinsics
return the current value of the register, where possible. The
'``llvm.write_register``' intrinsic sets the current value of the register,
where possible.

A call to '``llvm.read_volatile_register``' is assumed to have side-effects
and possibly return a different value each time (e.g. for a timer register).

This is useful to implement named register global variables that need
to always be mapped to a specific register, as is common practice on
bare-metal programs including OS kernels.

The compiler doesn't check for register availability or use of the used
register in surrounding code, including inline assembly. Because of that,
allocatable registers are not supported.

Warning: So far it only works with the stack pointer on selected
architectures (ARM, AArch64, PowerPC and x86_64). Significant amount of
work is needed to support other registers and even more so, allocatable
registers.

.. _int_stacksave:

'``llvm.stacksave``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare ptr @llvm.stacksave.p0()
      declare ptr addrspace(5) @llvm.stacksave.p5()

Overview:
"""""""""

The '``llvm.stacksave``' intrinsic is used to remember the current state
of the function stack, for use with
:ref:`llvm.stackrestore <int_stackrestore>`. This is useful for
implementing language features like scoped automatic variable sized
arrays in C99.

Semantics:
""""""""""

This intrinsic returns an opaque pointer value that can be passed to
:ref:`llvm.stackrestore <int_stackrestore>`. When an
``llvm.stackrestore`` intrinsic is executed with a value saved from
``llvm.stacksave``, it effectively restores the state of the stack to
the state it was in when the ``llvm.stacksave`` intrinsic executed. In
practice, this pops any :ref:`alloca <i_alloca>` blocks from the stack
that were allocated after the ``llvm.stacksave`` was executed. The
address space should typically be the
:ref:`alloca address space <alloca_addrspace>`.

.. _int_stackrestore:

'``llvm.stackrestore``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.stackrestore.p0(ptr %ptr)
      declare void @llvm.stackrestore.p5(ptr addrspace(5) %ptr)

Overview:
"""""""""

The '``llvm.stackrestore``' intrinsic is used to restore the state of
the function stack to the state it was in when the corresponding
:ref:`llvm.stacksave <int_stacksave>` intrinsic executed. This is
useful for implementing language features like scoped automatic
variable sized arrays in C99. The address space should typically be
the :ref:`alloca address space <alloca_addrspace>`.

Semantics:
""""""""""

See the description for :ref:`llvm.stacksave <int_stacksave>`.

.. _int_get_dynamic_area_offset:

'``llvm.get.dynamic.area.offset``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i32 @llvm.get.dynamic.area.offset.i32()
      declare i64 @llvm.get.dynamic.area.offset.i64()

Overview:
"""""""""

      The '``llvm.get.dynamic.area.offset.*``' intrinsic family is used to
      get the offset from native stack pointer to the address of the most
      recent dynamic alloca on the caller's stack. These intrinsics are
      intended for use in combination with
      :ref:`llvm.stacksave <int_stacksave>` to get a
      pointer to the most recent dynamic alloca. This is useful, for example,
      for AddressSanitizer's stack unpoisoning routines.

Semantics:
""""""""""

      These intrinsics return a non-negative integer value that can be used to
      get the address of the most recent dynamic alloca, allocated by :ref:`alloca <i_alloca>`
      on the caller's stack. In particular, for targets where stack grows downwards,
      adding this offset to the native stack pointer would get the address of the most
      recent dynamic alloca. For targets where stack grows upwards, the situation is a bit more
      complicated, because subtracting this value from stack pointer would get the address
      one past the end of the most recent dynamic alloca.

      Although for most targets `llvm.get.dynamic.area.offset <int_get_dynamic_area_offset>`
      returns just a zero, for others, such as PowerPC and PowerPC64, it returns a
      compile-time-known constant value.

      The return value type of :ref:`llvm.get.dynamic.area.offset <int_get_dynamic_area_offset>`
      must match the target's default address space's (address space 0) pointer type.

'``llvm.prefetch``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.prefetch(ptr <address>, i32 <rw>, i32 <locality>, i32 <cache type>)

Overview:
"""""""""

The '``llvm.prefetch``' intrinsic is a hint to the code generator to
insert a prefetch instruction if supported; otherwise, it is a noop.
Prefetches have no effect on the behavior of the program but can change
its performance characteristics.

Arguments:
""""""""""

``address`` is the address to be prefetched, ``rw`` is the specifier
determining if the fetch should be for a read (0) or write (1), and
``locality`` is a temporal locality specifier ranging from (0) - no
locality, to (3) - extremely local keep in cache. The ``cache type``
specifies whether the prefetch is performed on the data (1) or
instruction (0) cache. The ``rw``, ``locality`` and ``cache type``
arguments must be constant integers.

Semantics:
""""""""""

This intrinsic does not modify the behavior of the program. In
particular, prefetches cannot trap and do not produce a value. On
targets that support this intrinsic, the prefetch can provide hints to
the processor cache for better performance.

'``llvm.pcmarker``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.pcmarker(i32 <id>)

Overview:
"""""""""

The '``llvm.pcmarker``' intrinsic is a method to export a Program
Counter (PC) in a region of code to simulators and other tools. The
method is target specific, but it is expected that the marker will use
exported symbols to transmit the PC of the marker. The marker makes no
guarantees that it will remain with any specific instruction after
optimizations. It is possible that the presence of a marker will inhibit
optimizations. The intended use is to be inserted after optimizations to
allow correlations of simulation runs.

Arguments:
""""""""""

``id`` is a numerical id identifying the marker.

Semantics:
""""""""""

This intrinsic does not modify the behavior of the program. Backends
that do not support this intrinsic may ignore it.

'``llvm.readcyclecounter``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i64 @llvm.readcyclecounter()

Overview:
"""""""""

The '``llvm.readcyclecounter``' intrinsic provides access to the cycle
counter register (or similar low latency, high accuracy clocks) on those
targets that support it. On X86, it should map to RDTSC. On Alpha, it
should map to RPCC. As the backing counters overflow quickly (on the
order of 9 seconds on alpha), this should only be used for small
timings.

Semantics:
""""""""""

When directly supported, reading the cycle counter should not modify any
memory. Implementations are allowed to either return an application
specific value or a system wide value. On backends without support, this
is lowered to a constant 0.

Note that runtime support may be conditional on the privilege-level code is
running at and the host platform.

'``llvm.readsteadycounter``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i64 @llvm.readsteadycounter()

Overview:
"""""""""

The '``llvm.readsteadycounter``' intrinsic provides access to the fixed
frequency clock on targets that support it. Unlike '``llvm.readcyclecounter``',
this clock is expected to tick at a constant rate, making it suitable for
measuring elapsed time. The actual frequency of the clock is implementation
defined.

Semantics:
""""""""""

When directly supported, reading the steady counter should not modify any
memory. Implementations are allowed to either return an application
specific value or a system wide value. On backends without support, this
is lowered to a constant 0.

'``llvm.clear_cache``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.clear_cache(ptr, ptr)

Overview:
"""""""""

The '``llvm.clear_cache``' intrinsic ensures visibility of modifications
in the specified range to the execution unit of the processor. On
targets with non-unified instruction and data cache, the implementation
flushes the instruction cache.

Semantics:
""""""""""

On platforms with coherent instruction and data caches (e.g. x86), this
intrinsic is a nop. On platforms with non-coherent instruction and data
cache (e.g. ARM, MIPS), the intrinsic is lowered either to appropriate
instructions or a system call, if cache flushing requires special
privileges.

The default behavior is to emit a call to ``__clear_cache`` from the run
time library.

This intrinsic does *not* empty the instruction pipeline. Modifications
of the current function are outside the scope of the intrinsic.

'``llvm.instrprof.increment``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.instrprof.increment(ptr <name>, i64 <hash>,
                                             i32 <num-counters>, i32 <index>)

Overview:
"""""""""

The '``llvm.instrprof.increment``' intrinsic can be emitted by a
frontend for use with instrumentation based profiling. These will be
lowered by the ``-instrprof`` pass to generate execution counts of a
program at runtime.

Arguments:
""""""""""

The first argument is a pointer to a global variable containing the
name of the entity being instrumented. This should generally be the
(mangled) function name for a set of counters.

The second argument is a hash value that can be used by the consumer
of the profile data to detect changes to the instrumented source, and
the third is the number of counters associated with ``name``. It is an
error if ``hash`` or ``num-counters`` differ between two instances of
``instrprof.increment`` that refer to the same name.

The last argument refers to which of the counters for ``name`` should
be incremented. It should be a value between 0 and ``num-counters``.

Semantics:
""""""""""

This intrinsic represents an increment of a profiling counter. It will
cause the ``-instrprof`` pass to generate the appropriate data
structures and the code to increment the appropriate value, in a
format that can be written out by a compiler runtime and consumed via
the ``llvm-profdata`` tool.

.. FIXME: write complete doc on contextual instrumentation and link from here
.. and from llvm.instrprof.callsite.

The intrinsic is lowered differently for contextual profiling by the
``-ctx-instr-lower`` pass. Here:

* the entry basic block increment counter is lowered as a call to compiler-rt,
  to either ``__llvm_ctx_profile_start_context`` or
  ``__llvm_ctx_profile_get_context``. Either returns a pointer to a context object
  which contains a buffer into which counter increments can happen. Note that the
  pointer value returned by compiler-rt may have its LSB set - counter increments
  happen offset from the address with the LSB cleared.

* all the other lowerings of ``llvm.instrprof.increment[.step]`` happen within
  that context.

* the context is assumed to be a local value to the function, and no concurrency
  concerns need to be handled by LLVM.

'``llvm.instrprof.increment.step``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.instrprof.increment.step(ptr <name>, i64 <hash>,
                                                  i32 <num-counters>,
                                                  i32 <index>, i64 <step>)

Overview:
"""""""""

The '``llvm.instrprof.increment.step``' intrinsic is an extension to
the '``llvm.instrprof.increment``' intrinsic with an additional fifth
argument to specify the step of the increment.

Arguments:
""""""""""
The first four arguments are the same as '``llvm.instrprof.increment``'
intrinsic.

The last argument specifies the value of the increment of the counter variable.

Semantics:
""""""""""
See description of '``llvm.instrprof.increment``' intrinsic.

'``llvm.instrprof.callsite``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.instrprof.callsite(ptr <name>, i64 <hash>,
                                            i32 <num-counters>,
                                            i32 <index>, ptr <callsite>)

Overview:
"""""""""

The '``llvm.instrprof.callsite``' intrinsic should be emitted before a callsite
that's not to a "fake" callee (like another intrinsic or asm). It is used by
contextual profiling and has side-effects. Its lowering happens in IR, and
target-specific backends should never encounter it.

Arguments:
""""""""""
The first 4 arguments are similar to ``llvm.instrprof.increment``. The indexing
is specific to callsites, meaning callsites are indexed from 0, independent from
the indexes used by the other intrinsics (such as
``llvm.instrprof.increment[.step]``).

The last argument is the called value of the callsite this intrinsic precedes.

Semantics:
""""""""""

This is lowered by contextual profiling. In contextual profiling, functions get,
from compiler-rt, a pointer to a context object. The context object consists of
a buffer LLVM can use to perform counter increments (i.e. the lowering of
``llvm.instrprof.increment[.step]``. The address range following the counter
buffer, ``<num-counters>`` x ``sizeof(ptr)`` - sized, is expected to contain
pointers to contexts of functions called from this function ("subcontexts").
LLVM does not dereference into that memory region, just calculates GEPs.

The lowering of ``llvm.instrprof.callsite`` consists of:

* write to ``__llvm_ctx_profile_expected_callee`` the ``<callsite>`` value;

* write to ``__llvm_ctx_profile_callsite`` the address into this function's
  context of the ``<index>`` position into the subcontexts region.


``__llvm_ctx_profile_{expected_callee|callsite}`` are initialized by compiler-rt
and are TLS. They are both vectors of pointers of size 2. The index into each is
determined when the current function obtains the pointer to its context from
compiler-rt. The pointer's LSB gives the index.


'``llvm.instrprof.timestamp``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.instrprof.timestamp(i8* <name>, i64 <hash>,
                                             i32 <num-counters>, i32 <index>)

Overview:
"""""""""

The '``llvm.instrprof.timestamp``' intrinsic is used to implement temporal
profiling.

Arguments:
""""""""""
The arguments are the same as '``llvm.instrprof.increment``'. The ``index`` is
expected to always be zero.

Semantics:
""""""""""
Similar to the '``llvm.instrprof.increment``' intrinsic, but it stores a
timestamp representing when this function was executed for the first time.

'``llvm.instrprof.cover``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.instrprof.cover(ptr <name>, i64 <hash>,
                                         i32 <num-counters>, i32 <index>)

Overview:
"""""""""

The '``llvm.instrprof.cover``' intrinsic is used to implement coverage
instrumentation.

Arguments:
""""""""""
The arguments are the same as the first four arguments of
'``llvm.instrprof.increment``'.

Semantics:
""""""""""
Similar to the '``llvm.instrprof.increment``' intrinsic, but it stores zero to
the profiling variable to signify that the function has been covered. We store
zero because this is more efficient on some targets.

'``llvm.instrprof.value.profile``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.instrprof.value.profile(ptr <name>, i64 <hash>,
                                                 i64 <value>, i32 <value_kind>,
                                                 i32 <index>)

Overview:
"""""""""

The '``llvm.instrprof.value.profile``' intrinsic can be emitted by a
frontend for use with instrumentation based profiling. This will be
lowered by the ``-instrprof`` pass to find out the target values,
instrumented expressions take in a program at runtime.

Arguments:
""""""""""

The first argument is a pointer to a global variable containing the
name of the entity being instrumented. ``name`` should generally be the
(mangled) function name for a set of counters.

The second argument is a hash value that can be used by the consumer
of the profile data to detect changes to the instrumented source. It
is an error if ``hash`` differs between two instances of
``llvm.instrprof.*`` that refer to the same name.

The third argument is the value of the expression being profiled. The profiled
expression's value should be representable as an unsigned 64-bit value. The
fourth argument represents the kind of value profiling that is being done. The
supported value profiling kinds are enumerated through the
``InstrProfValueKind`` type declared in the
``<include/llvm/ProfileData/InstrProf.h>`` header file. The last argument is the
index of the instrumented expression within ``name``. It should be >= 0.

Semantics:
""""""""""

This intrinsic represents the point where a call to a runtime routine
should be inserted for value profiling of target expressions. ``-instrprof``
pass will generate the appropriate data structures and replace the
``llvm.instrprof.value.profile`` intrinsic with the call to the profile
runtime library with proper arguments.

'``llvm.instrprof.mcdc.parameters``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.instrprof.mcdc.parameters(ptr <name>, i64 <hash>,
                                                   i32 <bitmap-bits>)

Overview:
"""""""""

The '``llvm.instrprof.mcdc.parameters``' intrinsic is used to initiate MC/DC
code coverage instrumentation for a function.

Arguments:
""""""""""

The first argument is a pointer to a global variable containing the
name of the entity being instrumented. This should generally be the
(mangled) function name for a set of counters.

The second argument is a hash value that can be used by the consumer
of the profile data to detect changes to the instrumented source.

The third argument is the number of bitmap bits required by the function to
record the number of test vectors executed for each boolean expression.

Semantics:
""""""""""

This intrinsic represents basic MC/DC parameters initiating one or more MC/DC
instrumentation sequences in a function. It will cause the ``-instrprof`` pass
to generate the appropriate data structures and the code to instrument MC/DC
test vectors in a format that can be written out by a compiler runtime and
consumed via the ``llvm-profdata`` tool.

'``llvm.instrprof.mcdc.tvbitmap.update``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.instrprof.mcdc.tvbitmap.update(ptr <name>, i64 <hash>,
                                                        i32 <bitmap-index>,
                                                        ptr <mcdc-temp-addr>)

Overview:
"""""""""

The '``llvm.instrprof.mcdc.tvbitmap.update``' intrinsic is used to track MC/DC
test vector execution after each boolean expression has been fully executed.
The overall value of the condition bitmap, after it has been successively
updated with the true or false evaluation of each condition, uniquely identifies
an executed MC/DC test vector and is used as a bit index into the global test
vector bitmap.

Arguments:
""""""""""

The first argument is a pointer to a global variable containing the
name of the entity being instrumented. This should generally be the
(mangled) function name for a set of counters.

The second argument is a hash value that can be used by the consumer
of the profile data to detect changes to the instrumented source.

The third argument is the bit index into the global test vector bitmap
corresponding to the function.

The fourth argument is the address of the condition bitmap, which contains a
value representing an executed MC/DC test vector. It is loaded and used as the
bit index of the test vector bitmap.

Semantics:
""""""""""

This intrinsic represents the final operation of an MC/DC instrumentation
sequence and will cause the ``-instrprof`` pass to generate the code to
instrument an update of a function's global test vector bitmap to indicate that
a test vector has been executed. The global test vector bitmap can be consumed
by the ``llvm-profdata`` and ``llvm-cov`` tools.

'``llvm.thread.pointer``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare ptr @llvm.thread.pointer()

Overview:
"""""""""

The '``llvm.thread.pointer``' intrinsic returns the value of the thread
pointer.

Semantics:
""""""""""

The '``llvm.thread.pointer``' intrinsic returns a pointer to the TLS area
for the current thread.  The exact semantics of this value are target
specific: it may point to the start of TLS area, to the end, or somewhere
in the middle.  Depending on the target, this intrinsic may read a register,
call a helper function, read from an alternate memory space, or perform
other operations necessary to locate the TLS area.  Not all targets support
this intrinsic.

'``llvm.call.preallocated.setup``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare token @llvm.call.preallocated.setup(i32 %num_args)

Overview:
"""""""""

The '``llvm.call.preallocated.setup``' intrinsic returns a token which can
be used with a call's ``"preallocated"`` operand bundle to indicate that
certain arguments are allocated and initialized before the call.

Semantics:
""""""""""

The '``llvm.call.preallocated.setup``' intrinsic returns a token which is
associated with at most one call. The token can be passed to
'``@llvm.call.preallocated.arg``' to get a pointer to get that
corresponding argument. The token must be the parameter to a
``"preallocated"`` operand bundle for the corresponding call.

Nested calls to '``llvm.call.preallocated.setup``' are allowed, but must
be properly nested. e.g.

:: code-block:: llvm

      %t1 = call token @llvm.call.preallocated.setup(i32 0)
      %t2 = call token @llvm.call.preallocated.setup(i32 0)
      call void foo() ["preallocated"(token %t2)]
      call void foo() ["preallocated"(token %t1)]

is allowed, but not

:: code-block:: llvm

      %t1 = call token @llvm.call.preallocated.setup(i32 0)
      %t2 = call token @llvm.call.preallocated.setup(i32 0)
      call void foo() ["preallocated"(token %t1)]
      call void foo() ["preallocated"(token %t2)]

.. _int_call_preallocated_arg:

'``llvm.call.preallocated.arg``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare ptr @llvm.call.preallocated.arg(token %setup_token, i32 %arg_index)

Overview:
"""""""""

The '``llvm.call.preallocated.arg``' intrinsic returns a pointer to the
corresponding preallocated argument for the preallocated call.

Semantics:
""""""""""

The '``llvm.call.preallocated.arg``' intrinsic returns a pointer to the
``%arg_index``th argument with the ``preallocated`` attribute for
the call associated with the ``%setup_token``, which must be from
'``llvm.call.preallocated.setup``'.

A call to '``llvm.call.preallocated.arg``' must have a call site
``preallocated`` attribute. The type of the ``preallocated`` attribute must
match the type used by the ``preallocated`` attribute of the corresponding
argument at the preallocated call. The type is used in the case that an
``llvm.call.preallocated.setup`` does not have a corresponding call (e.g. due
to DCE), where otherwise we cannot know how large the arguments are.

It is undefined behavior if this is called with a token from an
'``llvm.call.preallocated.setup``' if another
'``llvm.call.preallocated.setup``' has already been called or if the
preallocated call corresponding to the '``llvm.call.preallocated.setup``'
has already been called.

.. _int_call_preallocated_teardown:

'``llvm.call.preallocated.teardown``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare ptr @llvm.call.preallocated.teardown(token %setup_token)

Overview:
"""""""""

The '``llvm.call.preallocated.teardown``' intrinsic cleans up the stack
created by a '``llvm.call.preallocated.setup``'.

Semantics:
""""""""""

The token argument must be a '``llvm.call.preallocated.setup``'.

The '``llvm.call.preallocated.teardown``' intrinsic cleans up the stack
allocated by the corresponding '``llvm.call.preallocated.setup``'. Exactly
one of this or the preallocated call must be called to prevent stack leaks.
It is undefined behavior to call both a '``llvm.call.preallocated.teardown``'
and the preallocated call for a given '``llvm.call.preallocated.setup``'.

For example, if the stack is allocated for a preallocated call by a
'``llvm.call.preallocated.setup``', then an initializer function called on an
allocated argument throws an exception, there should be a
'``llvm.call.preallocated.teardown``' in the exception handler to prevent
stack leaks.

Following the nesting rules in '``llvm.call.preallocated.setup``', nested
calls to '``llvm.call.preallocated.setup``' and
'``llvm.call.preallocated.teardown``' are allowed but must be properly
nested.

Example:
""""""""

.. code-block:: llvm

        %cs = call token @llvm.call.preallocated.setup(i32 1)
        %x = call ptr @llvm.call.preallocated.arg(token %cs, i32 0) preallocated(i32)
        invoke void @constructor(ptr %x) to label %conta unwind label %contb
    conta:
        call void @foo1(ptr preallocated(i32) %x) ["preallocated"(token %cs)]
        ret void
    contb:
        %s = catchswitch within none [label %catch] unwind to caller
    catch:
        %p = catchpad within %s []
        call void @llvm.call.preallocated.teardown(token %cs)
        ret void

Standard C/C++ Library Intrinsics
---------------------------------

LLVM provides intrinsics for a few important standard C/C++ library
functions. These intrinsics allow source-language front-ends to pass
information about the alignment of the pointer arguments to the code
generator, providing opportunity for more efficient code generation.

.. _int_abs:

'``llvm.abs.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.abs`` on any
integer bit width or any vector of integer elements.

::

      declare i32 @llvm.abs.i32(i32 <src>, i1 <is_int_min_poison>)
      declare <4 x i32> @llvm.abs.v4i32(<4 x i32> <src>, i1 <is_int_min_poison>)

Overview:
"""""""""

The '``llvm.abs``' family of intrinsic functions returns the absolute value
of an argument.

Arguments:
""""""""""

The first argument is the value for which the absolute value is to be returned.
This argument may be of any integer type or a vector with integer element type.
The return type must match the first argument type.

The second argument must be a constant and is a flag to indicate whether the
result value of the '``llvm.abs``' intrinsic is a
:ref:`poison value <poisonvalues>` if the first argument is statically or
dynamically an ``INT_MIN`` value.

Semantics:
""""""""""

The '``llvm.abs``' intrinsic returns the magnitude (always positive) of the
first argument or each element of a vector argument.". If the first argument is
``INT_MIN``, then the result is also ``INT_MIN`` if ``is_int_min_poison == 0``
and ``poison`` otherwise.


.. _int_smax:

'``llvm.smax.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``@llvm.smax`` on any
integer bit width or any vector of integer elements.

::

      declare i32 @llvm.smax.i32(i32 %a, i32 %b)
      declare <4 x i32> @llvm.smax.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview:
"""""""""

Return the larger of ``%a`` and ``%b`` comparing the values as signed integers.
Vector intrinsics operate on a per-element basis. The larger element of ``%a``
and ``%b`` at a given index is returned for that index.

Arguments:
""""""""""

The arguments (``%a`` and ``%b``) may be of any integer type or a vector with
integer element type. The argument types must match each other, and the return
type must match the argument type.


.. _int_smin:

'``llvm.smin.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``@llvm.smin`` on any
integer bit width or any vector of integer elements.

::

      declare i32 @llvm.smin.i32(i32 %a, i32 %b)
      declare <4 x i32> @llvm.smin.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview:
"""""""""

Return the smaller of ``%a`` and ``%b`` comparing the values as signed integers.
Vector intrinsics operate on a per-element basis. The smaller element of ``%a``
and ``%b`` at a given index is returned for that index.

Arguments:
""""""""""

The arguments (``%a`` and ``%b``) may be of any integer type or a vector with
integer element type. The argument types must match each other, and the return
type must match the argument type.


.. _int_umax:

'``llvm.umax.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``@llvm.umax`` on any
integer bit width or any vector of integer elements.

::

      declare i32 @llvm.umax.i32(i32 %a, i32 %b)
      declare <4 x i32> @llvm.umax.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview:
"""""""""

Return the larger of ``%a`` and ``%b`` comparing the values as unsigned
integers. Vector intrinsics operate on a per-element basis. The larger element
of ``%a`` and ``%b`` at a given index is returned for that index.

Arguments:
""""""""""

The arguments (``%a`` and ``%b``) may be of any integer type or a vector with
integer element type. The argument types must match each other, and the return
type must match the argument type.


.. _int_umin:

'``llvm.umin.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``@llvm.umin`` on any
integer bit width or any vector of integer elements.

::

      declare i32 @llvm.umin.i32(i32 %a, i32 %b)
      declare <4 x i32> @llvm.umin.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview:
"""""""""

Return the smaller of ``%a`` and ``%b`` comparing the values as unsigned
integers. Vector intrinsics operate on a per-element basis. The smaller element
of ``%a`` and ``%b`` at a given index is returned for that index.

Arguments:
""""""""""

The arguments (``%a`` and ``%b``) may be of any integer type or a vector with
integer element type. The argument types must match each other, and the return
type must match the argument type.

.. _int_scmp:

'``llvm.scmp.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``@llvm.scmp`` on any
integer bit width or any vector of integer elements.

::

      declare i2 @llvm.scmp.i2.i32(i32 %a, i32 %b)
      declare <4 x i32> @llvm.scmp.v4i32.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview:
"""""""""

Return ``-1`` if ``%a`` is signed less than ``%b``, ``0`` if they are equal, and
``1`` if ``%a`` is signed greater than ``%b``. Vector intrinsics operate on a per-element basis.

Arguments:
""""""""""

The arguments (``%a`` and ``%b``) may be of any integer type or a vector with
integer element type. The argument types must match each other, and the return
type must be at least as wide as ``i2``, to hold the three possible return values.

.. _int_ucmp:

'``llvm.ucmp.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``@llvm.ucmp`` on any
integer bit width or any vector of integer elements.

::

      declare i2 @llvm.ucmp.i2.i32(i32 %a, i32 %b)
      declare <4 x i32> @llvm.ucmp.v4i32.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview:
"""""""""

Return ``-1`` if ``%a`` is unsigned less than ``%b``, ``0`` if they are equal, and
``1`` if ``%a`` is unsigned greater than ``%b``. Vector intrinsics operate on a per-element basis.

Arguments:
""""""""""

The arguments (``%a`` and ``%b``) may be of any integer type or a vector with
integer element type. The argument types must match each other, and the return
type must be at least as wide as ``i2``, to hold the three possible return values.

.. _int_memcpy:

'``llvm.memcpy``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.memcpy`` on any
integer bit width and for different address spaces. Not all targets
support all bit widths however.

::

      declare void @llvm.memcpy.p0.p0.i32(ptr <dest>, ptr <src>,
                                          i32 <len>, i1 <isvolatile>)
      declare void @llvm.memcpy.p0.p0.i64(ptr <dest>, ptr <src>,
                                          i64 <len>, i1 <isvolatile>)

Overview:
"""""""""

The '``llvm.memcpy.*``' intrinsics copy a block of memory from the
source location to the destination location.

Note that, unlike the standard libc function, the ``llvm.memcpy.*``
intrinsics do not return a value, takes extra isvolatile
arguments and the pointers can be in specified address spaces.

Arguments:
""""""""""

The first argument is a pointer to the destination, the second is a
pointer to the source. The third argument is an integer argument
specifying the number of bytes to copy, and the fourth is a
boolean indicating a volatile access.

The :ref:`align <attr_align>` parameter attribute can be provided
for the first and second arguments.

If the ``isvolatile`` parameter is ``true``, the ``llvm.memcpy`` call is
a :ref:`volatile operation <volatile>`. The detailed access behavior is not
very cleanly specified and it is unwise to depend on it.

Semantics:
""""""""""

The '``llvm.memcpy.*``' intrinsics copy a block of memory from the source
location to the destination location, which must either be equal or
non-overlapping. It copies "len" bytes of memory over. If the argument is known
to be aligned to some boundary, this can be specified as an attribute on the
argument.

If ``<len>`` is 0, it is no-op modulo the behavior of attributes attached to
the arguments.
If ``<len>`` is not a well-defined value, the behavior is undefined.
If ``<len>`` is not zero, both ``<dest>`` and ``<src>`` should be well-defined,
otherwise the behavior is undefined.

.. _int_memcpy_inline:

'``llvm.memcpy.inline``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.memcpy.inline`` on any
integer bit width and for different address spaces. Not all targets
support all bit widths however.

::

      declare void @llvm.memcpy.inline.p0.p0.i32(ptr <dest>, ptr <src>,
                                                 i32 <len>, i1 <isvolatile>)
      declare void @llvm.memcpy.inline.p0.p0.i64(ptr <dest>, ptr <src>,
                                                 i64 <len>, i1 <isvolatile>)

Overview:
"""""""""

The '``llvm.memcpy.inline.*``' intrinsics copy a block of memory from the
source location to the destination location and guarantees that no external
functions are called.

Note that, unlike the standard libc function, the ``llvm.memcpy.inline.*``
intrinsics do not return a value, takes extra isvolatile
arguments and the pointers can be in specified address spaces.

Arguments:
""""""""""

The first argument is a pointer to the destination, the second is a
pointer to the source. The third argument is an integer argument
specifying the number of bytes to copy, and the fourth is a
boolean indicating a volatile access.

The :ref:`align <attr_align>` parameter attribute can be provided
for the first and second arguments.

If the ``isvolatile`` parameter is ``true``, the ``llvm.memcpy.inline`` call is
a :ref:`volatile operation <volatile>`. The detailed access behavior is not
very cleanly specified and it is unwise to depend on it.

Semantics:
""""""""""

The '``llvm.memcpy.inline.*``' intrinsics copy a block of memory from the
source location to the destination location, which are not allowed to
overlap. It copies "len" bytes of memory over. If the argument is known
to be aligned to some boundary, this can be specified as an attribute on
the argument.
The behavior of '``llvm.memcpy.inline.*``' is equivalent to the behavior of
'``llvm.memcpy.*``', but the generated code is guaranteed not to call any
external functions.

.. _int_memmove:

'``llvm.memmove``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use llvm.memmove on any integer
bit width and for different address space. Not all targets support all
bit widths however.

::

      declare void @llvm.memmove.p0.p0.i32(ptr <dest>, ptr <src>,
                                           i32 <len>, i1 <isvolatile>)
      declare void @llvm.memmove.p0.p0.i64(ptr <dest>, ptr <src>,
                                           i64 <len>, i1 <isvolatile>)

Overview:
"""""""""

The '``llvm.memmove.*``' intrinsics move a block of memory from the
source location to the destination location. It is similar to the
'``llvm.memcpy``' intrinsic but allows the two memory locations to
overlap.

Note that, unlike the standard libc function, the ``llvm.memmove.*``
intrinsics do not return a value, takes an extra isvolatile
argument and the pointers can be in specified address spaces.

Arguments:
""""""""""

The first argument is a pointer to the destination, the second is a
pointer to the source. The third argument is an integer argument
specifying the number of bytes to copy, and the fourth is a
boolean indicating a volatile access.

The :ref:`align <attr_align>` parameter attribute can be provided
for the first and second arguments.

If the ``isvolatile`` parameter is ``true``, the ``llvm.memmove`` call
is a :ref:`volatile operation <volatile>`. The detailed access behavior is
not very cleanly specified and it is unwise to depend on it.

Semantics:
""""""""""

The '``llvm.memmove.*``' intrinsics copy a block of memory from the
source location to the destination location, which may overlap. It
copies "len" bytes of memory over. If the argument is known to be
aligned to some boundary, this can be specified as an attribute on
the argument.

If ``<len>`` is 0, it is no-op modulo the behavior of attributes attached to
the arguments.
If ``<len>`` is not a well-defined value, the behavior is undefined.
If ``<len>`` is not zero, both ``<dest>`` and ``<src>`` should be well-defined,
otherwise the behavior is undefined.

.. _int_memset:

'``llvm.memset.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use llvm.memset on any integer
bit width and for different address spaces. However, not all targets
support all bit widths.

::

      declare void @llvm.memset.p0.i32(ptr <dest>, i8 <val>,
                                       i32 <len>, i1 <isvolatile>)
      declare void @llvm.memset.p0.i64(ptr <dest>, i8 <val>,
                                       i64 <len>, i1 <isvolatile>)

Overview:
"""""""""

The '``llvm.memset.*``' intrinsics fill a block of memory with a
particular byte value.

Note that, unlike the standard libc function, the ``llvm.memset``
intrinsic does not return a value and takes an extra volatile
argument. Also, the destination can be in an arbitrary address space.

Arguments:
""""""""""

The first argument is a pointer to the destination to fill, the second
is the byte value with which to fill it, the third argument is an
integer argument specifying the number of bytes to fill, and the fourth
is a boolean indicating a volatile access.

The :ref:`align <attr_align>` parameter attribute can be provided
for the first arguments.

If the ``isvolatile`` parameter is ``true``, the ``llvm.memset`` call is
a :ref:`volatile operation <volatile>`. The detailed access behavior is not
very cleanly specified and it is unwise to depend on it.

Semantics:
""""""""""

The '``llvm.memset.*``' intrinsics fill "len" bytes of memory starting
at the destination location. If the argument is known to be
aligned to some boundary, this can be specified as an attribute on
the argument.

If ``<len>`` is 0, it is no-op modulo the behavior of attributes attached to
the arguments.
If ``<len>`` is not a well-defined value, the behavior is undefined.
If ``<len>`` is not zero, ``<dest>`` should be well-defined, otherwise the
behavior is undefined.

.. _int_memset_inline:

'``llvm.memset.inline``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.memset.inline`` on any
integer bit width and for different address spaces. Not all targets
support all bit widths however.

::

      declare void @llvm.memset.inline.p0.p0i8.i32(ptr <dest>, i8 <val>,
                                                   i32 <len>, i1 <isvolatile>)
      declare void @llvm.memset.inline.p0.p0.i64(ptr <dest>, i8 <val>,
                                                 i64 <len>, i1 <isvolatile>)

Overview:
"""""""""

The '``llvm.memset.inline.*``' intrinsics fill a block of memory with a
particular byte value and guarantees that no external functions are called.

Note that, unlike the standard libc function, the ``llvm.memset.inline.*``
intrinsics do not return a value, take an extra isvolatile argument and the
pointer can be in specified address spaces.

Arguments:
""""""""""

The first argument is a pointer to the destination to fill, the second
is the byte value with which to fill it, the third argument is a constant
integer argument specifying the number of bytes to fill, and the fourth
is a boolean indicating a volatile access.

The :ref:`align <attr_align>` parameter attribute can be provided
for the first argument.

If the ``isvolatile`` parameter is ``true``, the ``llvm.memset.inline`` call is
a :ref:`volatile operation <volatile>`. The detailed access behavior is not
very cleanly specified and it is unwise to depend on it.

Semantics:
""""""""""

The '``llvm.memset.inline.*``' intrinsics fill "len" bytes of memory starting
at the destination location. If the argument is known to be
aligned to some boundary, this can be specified as an attribute on
the argument.

If ``<len>`` is 0, it is no-op modulo the behavior of attributes attached to
the arguments.
If ``<len>`` is not a well-defined value, the behavior is undefined.
If ``<len>`` is not zero, ``<dest>`` should be well-defined, otherwise the
behavior is undefined.

The behavior of '``llvm.memset.inline.*``' is equivalent to the behavior of
'``llvm.memset.*``', but the generated code is guaranteed not to call any
external functions.

.. _int_experimental_memset_pattern:

'``llvm.experimental.memset.pattern``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use
``llvm.experimental.memset.pattern`` on any sized type and for different
address spaces.

::

      declare void @llvm.experimental.memset.pattern.p0.i128.i64(ptr <dest>, i128 <val>,
                                                                 i64 <count>, i1 <isvolatile>)

Overview:
"""""""""

The '``llvm.experimental.memset.pattern.*``' intrinsics fill a block of memory
with a particular value. This may be expanded to an inline loop, a sequence of
stores, or a libcall depending on what is available for the target and the
expected performance and code size impact.

Arguments:
""""""""""

The first argument is a pointer to the destination to fill, the second
is the value with which to fill it, the third argument is an integer
argument specifying the number of times to fill the value, and the fourth is a
boolean indicating a volatile access.

The :ref:`align <attr_align>` parameter attribute can be provided
for the first argument.

If the ``isvolatile`` parameter is ``true``, the
``llvm.experimental.memset.pattern`` call is a :ref:`volatile operation
<volatile>`. The detailed access behavior is not very cleanly specified and it
is unwise to depend on it.

Semantics:
""""""""""

The '``llvm.experimental.memset.pattern*``' intrinsic fills memory starting at
the destination location with the given pattern ``<count>`` times,
incrementing by the allocation size of the type each time. The stores follow
the usual semantics of store instructions, including regarding endianness and
padding. If the argument is known to be aligned to some boundary, this can be
specified as an attribute on the argument.

If ``<count>`` is 0, it is no-op modulo the behavior of attributes attached to
the arguments.
If ``<count>`` is not a well-defined value, the behavior is undefined.
If ``<count>`` is not zero, ``<dest>`` should be well-defined, otherwise the
behavior is undefined.

.. _int_sqrt:

'``llvm.sqrt.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.sqrt`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.sqrt.f32(float %Val)
      declare double    @llvm.sqrt.f64(double %Val)
      declare x86_fp80  @llvm.sqrt.f80(x86_fp80 %Val)
      declare fp128     @llvm.sqrt.f128(fp128 %Val)
      declare ppc_fp128 @llvm.sqrt.ppcf128(ppc_fp128 %Val)

Overview:
"""""""""

The '``llvm.sqrt``' intrinsics return the square root of the specified value.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``sqrt``' function but without
trapping or setting ``errno``. For types specified by IEEE-754, the result
matches a conforming libm implementation.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

'``llvm.powi.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.powi`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

Generally, the only supported type for the exponent is the one matching
with the C type ``int``.

::

      declare float     @llvm.powi.f32.i32(float  %Val, i32 %power)
      declare double    @llvm.powi.f64.i16(double %Val, i16 %power)
      declare x86_fp80  @llvm.powi.f80.i32(x86_fp80  %Val, i32 %power)
      declare fp128     @llvm.powi.f128.i32(fp128 %Val, i32 %power)
      declare ppc_fp128 @llvm.powi.ppcf128.i32(ppc_fp128  %Val, i32 %power)

Overview:
"""""""""

The '``llvm.powi.*``' intrinsics return the first operand raised to the
specified (positive or negative) power. The order of evaluation of
multiplications is not defined. When a vector of floating-point type is
used, the second argument remains a scalar integer value.

Arguments:
""""""""""

The second argument is an integer power, and the first is a value to
raise to that power.

Semantics:
""""""""""

This function returns the first value raised to the second power with an
unspecified sequence of rounding operations.

.. _t_llvm_sin:

'``llvm.sin.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.sin`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.sin.f32(float  %Val)
      declare double    @llvm.sin.f64(double %Val)
      declare x86_fp80  @llvm.sin.f80(x86_fp80  %Val)
      declare fp128     @llvm.sin.f128(fp128 %Val)
      declare ppc_fp128 @llvm.sin.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.sin.*``' intrinsics return the sine of the operand.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``sin``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

.. _t_llvm_cos:

'``llvm.cos.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.cos`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.cos.f32(float  %Val)
      declare double    @llvm.cos.f64(double %Val)
      declare x86_fp80  @llvm.cos.f80(x86_fp80  %Val)
      declare fp128     @llvm.cos.f128(fp128 %Val)
      declare ppc_fp128 @llvm.cos.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.cos.*``' intrinsics return the cosine of the operand.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``cos``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

'``llvm.tan.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.tan`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.tan.f32(float  %Val)
      declare double    @llvm.tan.f64(double %Val)
      declare x86_fp80  @llvm.tan.f80(x86_fp80  %Val)
      declare fp128     @llvm.tan.f128(fp128 %Val)
      declare ppc_fp128 @llvm.tan.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.tan.*``' intrinsics return the tangent of the operand.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``tan``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

'``llvm.asin.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.asin`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.asin.f32(float  %Val)
      declare double    @llvm.asin.f64(double %Val)
      declare x86_fp80  @llvm.asin.f80(x86_fp80  %Val)
      declare fp128     @llvm.asin.f128(fp128 %Val)
      declare ppc_fp128 @llvm.asin.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.asin.*``' intrinsics return the arcsine of the operand.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``asin``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

'``llvm.acos.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.acos`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.acos.f32(float  %Val)
      declare double    @llvm.acos.f64(double %Val)
      declare x86_fp80  @llvm.acos.f80(x86_fp80  %Val)
      declare fp128     @llvm.acos.f128(fp128 %Val)
      declare ppc_fp128 @llvm.acos.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.acos.*``' intrinsics return the arccosine of the operand.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``acos``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

'``llvm.atan.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.atan`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.atan.f32(float  %Val)
      declare double    @llvm.atan.f64(double %Val)
      declare x86_fp80  @llvm.atan.f80(x86_fp80  %Val)
      declare fp128     @llvm.atan.f128(fp128 %Val)
      declare ppc_fp128 @llvm.atan.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.atan.*``' intrinsics return the arctangent of the operand.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``atan``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

'``llvm.atan2.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.atan2`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.atan2.f32(float  %Y, float %X)
      declare double    @llvm.atan2.f64(double %Y, double %X)
      declare x86_fp80  @llvm.atan2.f80(x86_fp80  %Y, x86_fp80 %X)
      declare fp128     @llvm.atan2.f128(fp128 %Y, fp128 %X)
      declare ppc_fp128 @llvm.atan2.ppcf128(ppc_fp128  %Y, ppc_fp128 %X)

Overview:
"""""""""

The '``llvm.atan2.*``' intrinsics return the arctangent of ``Y/X`` accounting
for the quadrant.

Arguments:
""""""""""

The arguments and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``atan2``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

'``llvm.sinh.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.sinh`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.sinh.f32(float  %Val)
      declare double    @llvm.sinh.f64(double %Val)
      declare x86_fp80  @llvm.sinh.f80(x86_fp80  %Val)
      declare fp128     @llvm.sinh.f128(fp128 %Val)
      declare ppc_fp128 @llvm.sinh.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.sinh.*``' intrinsics return the hyperbolic sine of the operand.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``sinh``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

'``llvm.cosh.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.cosh`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.cosh.f32(float  %Val)
      declare double    @llvm.cosh.f64(double %Val)
      declare x86_fp80  @llvm.cosh.f80(x86_fp80  %Val)
      declare fp128     @llvm.cosh.f128(fp128 %Val)
      declare ppc_fp128 @llvm.cosh.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.cosh.*``' intrinsics return the hyperbolic cosine of the operand.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``cosh``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

'``llvm.tanh.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.tanh`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.tanh.f32(float  %Val)
      declare double    @llvm.tanh.f64(double %Val)
      declare x86_fp80  @llvm.tanh.f80(x86_fp80  %Val)
      declare fp128     @llvm.tanh.f128(fp128 %Val)
      declare ppc_fp128 @llvm.tanh.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.tanh.*``' intrinsics return the hyperbolic tangent of the operand.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``tanh``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.


'``llvm.sincos.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.sincos`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare { float, float }          @llvm.sincos.f32(float  %Val)
      declare { double, double }        @llvm.sincos.f64(double %Val)
      declare { x86_fp80, x86_fp80 }    @llvm.sincos.f80(x86_fp80  %Val)
      declare { fp128, fp128 }          @llvm.sincos.f128(fp128 %Val)
      declare { ppc_fp128, ppc_fp128 }  @llvm.sincos.ppcf128(ppc_fp128  %Val)
      declare { <4 x float>, <4 x float> } @llvm.sincos.v4f32(<4 x float>  %Val)

Overview:
"""""""""

The '``llvm.sincos.*``' intrinsics returns the sine and cosine of the operand.

Arguments:
""""""""""

The argument is a :ref:`floating-point <t_floating>` value or
:ref:`vector <t_vector>` of floating-point values. Returns two values matching
the argument type in a struct.

Semantics:
""""""""""

This intrinsic is equivalent to a calling both :ref:`llvm.sin <t_llvm_sin>`
and :ref:`llvm.cos <t_llvm_cos>` on the argument.

The first result is the sine of the argument and the second result is the cosine
of the argument.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

'``llvm.sincospi.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.sincospi`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare { float, float }          @llvm.sincospi.f32(float  %Val)
      declare { double, double }        @llvm.sincospi.f64(double %Val)
      declare { x86_fp80, x86_fp80 }    @llvm.sincospi.f80(x86_fp80  %Val)
      declare { fp128, fp128 }          @llvm.sincospi.f128(fp128 %Val)
      declare { ppc_fp128, ppc_fp128 }  @llvm.sincospi.ppcf128(ppc_fp128  %Val)
      declare { <4 x float>, <4 x float> } @llvm.sincospi.v4f32(<4 x float>  %Val)

Overview:
"""""""""

The '``llvm.sincospi.*``' intrinsics returns the sine and cosine of pi*operand.

Arguments:
""""""""""

The argument is a :ref:`floating-point <t_floating>` value or
:ref:`vector <t_vector>` of floating-point values. Returns two values matching
the argument type in a struct.

Semantics:
""""""""""

This is equivalent to the ``llvm.sincos.*`` intrinsic where the argument has been
multiplied by pi, however, it computes the result more accurately especially
for large input values.

.. note::

  Currently, the default lowering of this intrinsic relies on the ``sincospi[f|l]``
  functions being available in the target's runtime (e.g. libc).

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

'``llvm.modf.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.modf`` on any floating-point
or vector of floating-point type. However, not all targets support all types.

::

 declare { float, float }             @llvm.modf.f32(float  %Val)
 declare { double, double }           @llvm.modf.f64(double %Val)
 declare { x86_fp80, x86_fp80 }       @llvm.modf.f80(x86_fp80  %Val)
 declare { fp128, fp128 }             @llvm.modf.f128(fp128 %Val)
 declare { ppc_fp128, ppc_fp128 }     @llvm.modf.ppcf128(ppc_fp128  %Val)
 declare { <4 x float>, <4 x float> } @llvm.modf.v4f32(<4 x float>  %Val)

Overview:
"""""""""

The '``llvm.modf.*``' intrinsics return the operand's integral and fractional
parts.

Arguments:
""""""""""

The argument is a :ref:`floating-point <t_floating>` value or
:ref:`vector <t_vector>` of floating-point values. Returns two values matching
the argument type in a struct.

Semantics:
""""""""""

Return the same values as a corresponding libm '``modf``' function without
trapping or setting ``errno``.

The first result is the fractional part of the operand and the second result is
the integral part of the operand. Both results have the same sign as the operand.

Not including exceptional inputs (listed below), ``llvm.modf.*`` is semantically
equivalent to:

::

  %fp = frem <fptype> %x, 1.0  ; Fractional part
  %ip = fsub <fptype> %x, %fp  ; Integral part

(assuming no floating-point precision errors)

If the argument is a zero, returns a zero with the same sign for both the
fractional and integral parts.

If the argument is an infinity, returns a fractional part of zero with the same
sign, and infinity with the same sign as the integral part.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

'``llvm.pow.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.pow`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.pow.f32(float  %Val, float %Power)
      declare double    @llvm.pow.f64(double %Val, double %Power)
      declare x86_fp80  @llvm.pow.f80(x86_fp80  %Val, x86_fp80 %Power)
      declare fp128     @llvm.pow.f128(fp128 %Val, fp128 %Power)
      declare ppc_fp128 @llvm.pow.ppcf128(ppc_fp128  %Val, ppc_fp128 Power)

Overview:
"""""""""

The '``llvm.pow.*``' intrinsics return the first operand raised to the
specified (positive or negative) power.

Arguments:
""""""""""

The arguments and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``pow``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

.. _int_exp:

'``llvm.exp.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.exp`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.exp.f32(float  %Val)
      declare double    @llvm.exp.f64(double %Val)
      declare x86_fp80  @llvm.exp.f80(x86_fp80  %Val)
      declare fp128     @llvm.exp.f128(fp128 %Val)
      declare ppc_fp128 @llvm.exp.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.exp.*``' intrinsics compute the base-e exponential of the specified
value.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``exp``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

.. _int_exp2:

'``llvm.exp2.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.exp2`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.exp2.f32(float  %Val)
      declare double    @llvm.exp2.f64(double %Val)
      declare x86_fp80  @llvm.exp2.f80(x86_fp80  %Val)
      declare fp128     @llvm.exp2.f128(fp128 %Val)
      declare ppc_fp128 @llvm.exp2.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.exp2.*``' intrinsics compute the base-2 exponential of the
specified value.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``exp2``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

.. _int_exp10:

'``llvm.exp10.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.exp10`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.exp10.f32(float  %Val)
      declare double    @llvm.exp10.f64(double %Val)
      declare x86_fp80  @llvm.exp10.f80(x86_fp80  %Val)
      declare fp128     @llvm.exp10.f128(fp128 %Val)
      declare ppc_fp128 @llvm.exp10.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.exp10.*``' intrinsics compute the base-10 exponential of the
specified value.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``exp10``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.


'``llvm.ldexp.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.ldexp`` on any
floating point or vector of floating point type. Not all targets support
all types however.

::

      declare float     @llvm.ldexp.f32.i32(float %Val, i32 %Exp)
      declare double    @llvm.ldexp.f64.i32(double %Val, i32 %Exp)
      declare x86_fp80  @llvm.ldexp.f80.i32(x86_fp80 %Val, i32 %Exp)
      declare fp128     @llvm.ldexp.f128.i32(fp128 %Val, i32 %Exp)
      declare ppc_fp128 @llvm.ldexp.ppcf128.i32(ppc_fp128 %Val, i32 %Exp)
      declare <2 x float> @llvm.ldexp.v2f32.v2i32(<2 x float> %Val, <2 x i32> %Exp)

Overview:
"""""""""

The '``llvm.ldexp.*``' intrinsics perform the ldexp function.

Arguments:
""""""""""

The first argument and the return value are :ref:`floating-point
<t_floating>` or :ref:`vector <t_vector>` of floating-point values of
the same type. The second argument is an integer with the same number
of elements.

Semantics:
""""""""""

This function multiplies the first argument by 2 raised to the second
argument's power. If the first argument is NaN or infinite, the same
value is returned. If the result underflows a zero with the same sign
is returned. If the result overflows, the result is an infinity with
the same sign.

.. _int_frexp:

'``llvm.frexp.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.frexp`` on any
floating point or vector of floating point type. Not all targets support
all types however.

::

      declare { float, i32 }     @llvm.frexp.f32.i32(float %Val)
      declare { double, i32 }    @llvm.frexp.f64.i32(double %Val)
      declare { x86_fp80, i32 }  @llvm.frexp.f80.i32(x86_fp80 %Val)
      declare { fp128, i32 }     @llvm.frexp.f128.i32(fp128 %Val)
      declare { ppc_fp128, i32 } @llvm.frexp.ppcf128.i32(ppc_fp128 %Val)
      declare { <2 x float>, <2 x i32> }  @llvm.frexp.v2f32.v2i32(<2 x float> %Val)

Overview:
"""""""""

The '``llvm.frexp.*``' intrinsics perform the frexp function.

Arguments:
""""""""""

The argument is a :ref:`floating-point <t_floating>` or
:ref:`vector <t_vector>` of floating-point values. Returns two values
in a struct. The first struct field matches the argument type, and the
second field is an integer or a vector of integer values with the same
number of elements as the argument.

Semantics:
""""""""""

This intrinsic splits a floating point value into a normalized
fractional component and integral exponent.

For a non-zero argument, returns the argument multiplied by some power
of two such that the absolute value of the returned value is in the
range [0.5, 1.0), with the same sign as the argument. The second
result is an integer such that the first result raised to the power of
the second result is the input argument.

If the argument is a zero, returns a zero with the same sign and a 0
exponent.

If the argument is a NaN, a NaN is returned and the returned exponent
is unspecified.

If the argument is an infinity, returns an infinity with the same sign
and an unspecified exponent.

.. _int_log:

'``llvm.log.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.log`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.log.f32(float  %Val)
      declare double    @llvm.log.f64(double %Val)
      declare x86_fp80  @llvm.log.f80(x86_fp80  %Val)
      declare fp128     @llvm.log.f128(fp128 %Val)
      declare ppc_fp128 @llvm.log.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.log.*``' intrinsics compute the base-e logarithm of the specified
value.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``log``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

.. _int_log10:

'``llvm.log10.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.log10`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.log10.f32(float  %Val)
      declare double    @llvm.log10.f64(double %Val)
      declare x86_fp80  @llvm.log10.f80(x86_fp80  %Val)
      declare fp128     @llvm.log10.f128(fp128 %Val)
      declare ppc_fp128 @llvm.log10.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.log10.*``' intrinsics compute the base-10 logarithm of the
specified value.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``log10``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.


.. _int_log2:

'``llvm.log2.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.log2`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.log2.f32(float  %Val)
      declare double    @llvm.log2.f64(double %Val)
      declare x86_fp80  @llvm.log2.f80(x86_fp80  %Val)
      declare fp128     @llvm.log2.f128(fp128 %Val)
      declare ppc_fp128 @llvm.log2.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.log2.*``' intrinsics compute the base-2 logarithm of the specified
value.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as a corresponding libm '``log2``' function but without
trapping or setting ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

.. _int_fma:

'``llvm.fma.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.fma`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.fma.f32(float  %a, float  %b, float  %c)
      declare double    @llvm.fma.f64(double %a, double %b, double %c)
      declare x86_fp80  @llvm.fma.f80(x86_fp80 %a, x86_fp80 %b, x86_fp80 %c)
      declare fp128     @llvm.fma.f128(fp128 %a, fp128 %b, fp128 %c)
      declare ppc_fp128 @llvm.fma.ppcf128(ppc_fp128 %a, ppc_fp128 %b, ppc_fp128 %c)

Overview:
"""""""""

The '``llvm.fma.*``' intrinsics perform the fused multiply-add operation.

Arguments:
""""""""""

The arguments and return value are floating-point numbers of the same type.

Semantics:
""""""""""

Return the same value as the IEEE-754 fusedMultiplyAdd operation. This
is assumed to not trap or set ``errno``.

When specified with the fast-math-flag 'afn', the result may be approximated
using a less accurate calculation.

.. _int_fabs:

'``llvm.fabs.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.fabs`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.fabs.f32(float  %Val)
      declare double    @llvm.fabs.f64(double %Val)
      declare x86_fp80  @llvm.fabs.f80(x86_fp80 %Val)
      declare fp128     @llvm.fabs.f128(fp128 %Val)
      declare ppc_fp128 @llvm.fabs.ppcf128(ppc_fp128 %Val)

Overview:
"""""""""

The '``llvm.fabs.*``' intrinsics return the absolute value of the
operand.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same
type.

Semantics:
""""""""""

This function returns the same values as the libm ``fabs`` functions
would, and handles error conditions in the same way.
The returned value is completely identical to the input except for the sign bit;
in particular, if the input is a NaN, then the quiet/signaling bit and payload
are perfectly preserved.

.. _i_fminmax_family:

'``llvm.min.*``' Intrinsics Comparation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Standard:
"""""""""

IEEE754 and ISO C define some min/max operations, and they have some differences
on working with qNaN/sNaN and +0.0/-0.0. Here is the list:

.. list-table::
   :header-rows: 2

   * - ``ISO C``
     - fmin/fmax
     - fmininum/fmaximum
     - fminimum_num/fmaximum_num

   * - ``IEEE754``
     - minNum/maxNum (2008)
     - minimum/maximum (2019)
     - minimumNumber/maximumNumber (2019)

   * - ``+0.0 vs -0.0``
     - either one
     - +0.0 > -0.0
     - +0.0 > -0.0

   * - ``NUM vs sNaN``
     - qNaN, invalid exception
     - qNaN, invalid exception
     - NUM, invalid exception

   * - ``qNaN vs sNaN``
     - qNaN, invalid exception
     - qNaN, invalid exception
     - qNaN, invalid exception

   * - ``NUM vs qNaN``
     - NUM, no exception
     - qNaN, no exception
     - NUM, no exception

LLVM Implementation:
""""""""""""""""""""

LLVM implements all ISO C flavors as listed in this table, except in the
default floating-point environment exceptions are ignored. The constrained
versions of the intrinsics respect the exception behavior.

.. list-table::
   :header-rows: 1
   :widths: 16 28 28 28

   * - Operation
     - minnum/maxnum
     - minimum/maximum
     - minimumnum/maximumnum

   * - ``NUM vs qNaN``
     - NUM, no exception
     - qNaN, no exception
     - NUM, no exception

   * - ``NUM vs sNaN``
     - qNaN, invalid exception
     - qNaN, invalid exception
     - NUM, invalid exception

   * - ``qNaN vs sNaN``
     - qNaN, invalid exception
     - qNaN, invalid exception
     - qNaN, invalid exception

   * - ``sNaN vs sNaN``
     - qNaN, invalid exception
     - qNaN, invalid exception
     - qNaN, invalid exception

   * - ``+0.0 vs -0.0``
     - +0.0(max)/-0.0(min)
     - +0.0(max)/-0.0(min)
     - +0.0(max)/-0.0(min)

   * - ``NUM vs NUM``
     - larger(max)/smaller(min)
     - larger(max)/smaller(min)
     - larger(max)/smaller(min)

.. _i_minnum:

'``llvm.minnum.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.minnum`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.minnum.f32(float %Val0, float %Val1)
      declare double    @llvm.minnum.f64(double %Val0, double %Val1)
      declare x86_fp80  @llvm.minnum.f80(x86_fp80 %Val0, x86_fp80 %Val1)
      declare fp128     @llvm.minnum.f128(fp128 %Val0, fp128 %Val1)
      declare ppc_fp128 @llvm.minnum.ppcf128(ppc_fp128 %Val0, ppc_fp128 %Val1)

Overview:
"""""""""

The '``llvm.minnum.*``' intrinsics return the minimum of the two
arguments.


Arguments:
""""""""""

The arguments and return value are floating-point numbers of the same
type.

Semantics:
""""""""""
Follows the semantics of minNum in IEEE-754-2008, except that -0.0 < +0.0 for the purposes
of this intrinsic. As for signaling NaNs, per the minNum semantics, if either operand is sNaN,
the result is qNaN. This matches the recommended behavior for the libm
function ``fmin``, although not all implementations have implemented these recommended behaviors.

If either operand is a qNaN, returns the other non-NaN operand. Returns NaN only if both operands are
NaN or if either operand is sNaN. Note that arithmetic on an sNaN doesn't consistently produce a qNaN,
so arithmetic feeding into a minnum can produce inconsistent results. For example,
``minnum(fadd(sNaN, -0.0), 1.0)`` can produce qNaN or 1.0 depending on whether ``fadd`` is folded.

IEEE-754-2008 defines minNum, and it was removed in IEEE-754-2019. As the replacement, IEEE-754-2019
defines :ref:`minimumNumber <i_minimumnum>`.

If the intrinsic is marked with the nsz attribute, then the effect is as in the definition in C
and IEEE-754-2008: the result of ``minnum(-0.0, +0.0)`` may be either -0.0 or +0.0.

Some architectures, such as ARMv8 (FMINNM), LoongArch (fmin), MIPSr6 (min.fmt), PowerPC/VSX (xsmindp),
have instructions that match these semantics exactly; thus it is quite simple for these architectures.
Some architectures have similiar ones while they are not exact equivalent. Such as x86 implements ``MINPS``,
which implements the semantics of C code ``a<b?a:b``: NUM vs qNaN always return qNaN. ``MINPS`` can be used
if ``nsz`` and ``nnan`` are given.

For existing libc implementations, the behaviors of fmin may be quite different on sNaN and signed zero behaviors,
even in the same release of a single libm implemention.

.. _i_maxnum:

'``llvm.maxnum.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.maxnum`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.maxnum.f32(float  %Val0, float  %Val1)
      declare double    @llvm.maxnum.f64(double %Val0, double %Val1)
      declare x86_fp80  @llvm.maxnum.f80(x86_fp80  %Val0, x86_fp80  %Val1)
      declare fp128     @llvm.maxnum.f128(fp128 %Val0, fp128 %Val1)
      declare ppc_fp128 @llvm.maxnum.ppcf128(ppc_fp128  %Val0, ppc_fp128  %Val1)

Overview:
"""""""""

The '``llvm.maxnum.*``' intrinsics return the maximum of the two
arguments.


Arguments:
""""""""""

The arguments and return value are floating-point numbers of the same
type.

Semantics:
""""""""""
Follows the semantics of maxNum in IEEE-754-2008, except that -0.0 < +0.0 for the purposes
of this intrinsic. As for signaling NaNs, per the maxNum semantics, if either operand is sNaN,
the result is qNaN. This matches the recommended behavior for the libm
function ``fmax``, although not all implementations have implemented these recommended behaviors.

If either operand is a qNaN, returns the other non-NaN operand. Returns NaN only if both operands are
NaN or if either operand is sNaN. Note that arithmetic on an sNaN doesn't consistently produce a qNaN,
so arithmetic feeding into a maxnum can produce inconsistent results. For example,
``maxnum(fadd(sNaN, -0.0), 1.0)`` can produce qNaN or 1.0 depending on whether ``fadd`` is folded.

IEEE-754-2008 defines maxNum, and it was removed in IEEE-754-2019. As the replacement, IEEE-754-2019
defines :ref:`maximumNumber <i_maximumnum>`.

If the intrinsic is marked with the nsz attribute, then the effect is as in the definition in C
and IEEE-754-2008: the result of maxnum(-0.0, +0.0) may be either -0.0 or +0.0.

Some architectures, such as ARMv8 (FMAXNM), LoongArch (fmax), MIPSr6 (max.fmt), PowerPC/VSX (xsmaxdp),
have instructions that match these semantics exactly; thus it is quite simple for these architectures.
Some architectures have similiar ones while they are not exact equivalent. Such as x86 implements ``MAXPS``,
which implements the semantics of C code ``a>b?a:b``: NUM vs qNaN always return qNaN. ``MAXPS`` can be used
if ``nsz`` and ``nnan`` are given.

For existing libc implementations, the behaviors of fmin may be quite different on sNaN and signed zero behaviors,
even in the same release of a single libm implemention.

.. _i_minimum:

'``llvm.minimum.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.minimum`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.minimum.f32(float %Val0, float %Val1)
      declare double    @llvm.minimum.f64(double %Val0, double %Val1)
      declare x86_fp80  @llvm.minimum.f80(x86_fp80 %Val0, x86_fp80 %Val1)
      declare fp128     @llvm.minimum.f128(fp128 %Val0, fp128 %Val1)
      declare ppc_fp128 @llvm.minimum.ppcf128(ppc_fp128 %Val0, ppc_fp128 %Val1)

Overview:
"""""""""

The '``llvm.minimum.*``' intrinsics return the minimum of the two
arguments, propagating NaNs and treating -0.0 as less than +0.0.


Arguments:
""""""""""

The arguments and return value are floating-point numbers of the same
type.

Semantics:
""""""""""
If either operand is a NaN, returns NaN. Otherwise returns the lesser
of the two arguments. -0.0 is considered to be less than +0.0 for this
intrinsic. Note that these are the semantics specified in the draft of
IEEE 754-2019.

.. _i_maximum:

'``llvm.maximum.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.maximum`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.maximum.f32(float %Val0, float %Val1)
      declare double    @llvm.maximum.f64(double %Val0, double %Val1)
      declare x86_fp80  @llvm.maximum.f80(x86_fp80 %Val0, x86_fp80 %Val1)
      declare fp128     @llvm.maximum.f128(fp128 %Val0, fp128 %Val1)
      declare ppc_fp128 @llvm.maximum.ppcf128(ppc_fp128 %Val0, ppc_fp128 %Val1)

Overview:
"""""""""

The '``llvm.maximum.*``' intrinsics return the maximum of the two
arguments, propagating NaNs and treating -0.0 as less than +0.0.


Arguments:
""""""""""

The arguments and return value are floating-point numbers of the same
type.

Semantics:
""""""""""
If either operand is a NaN, returns NaN. Otherwise returns the greater
of the two arguments. -0.0 is considered to be less than +0.0 for this
intrinsic. Note that these are the semantics specified in the draft of
IEEE 754-2019.

.. _i_minimumnum:

'``llvm.minimumnum.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.minimumnum`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.minimumnum.f32(float %Val0, float %Val1)
      declare double    @llvm.minimumnum.f64(double %Val0, double %Val1)
      declare x86_fp80  @llvm.minimumnum.f80(x86_fp80 %Val0, x86_fp80 %Val1)
      declare fp128     @llvm.minimumnum.f128(fp128 %Val0, fp128 %Val1)
      declare ppc_fp128 @llvm.minimumnum.ppcf128(ppc_fp128 %Val0, ppc_fp128 %Val1)

Overview:
"""""""""

The '``llvm.minimumnum.*``' intrinsics return the minimum of the two
arguments, not propagating NaNs and treating -0.0 as less than +0.0.


Arguments:
""""""""""

The arguments and return value are floating-point numbers of the same
type.

Semantics:
""""""""""
If both operands are NaNs (including sNaN), returns qNaN. If one operand
is NaN (including sNaN) and another operand is a number, return the number.
Otherwise returns the lesser of the two arguments. -0.0 is considered to
be less than +0.0 for this intrinsic.

Note that these are the semantics of minimumNumber specified in IEEE 754-2019.

It has some differences with '``llvm.minnum.*``':
1)'``llvm.minnum.*``' will return qNaN if either operand is sNaN.
2)'``llvm.minnum*``' may return either one if we compare +0.0 vs -0.0.

.. _i_maximumnum:

'``llvm.maximumnum.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.maximumnum`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.maximumnum.f32(float %Val0, float %Val1)
      declare double    @llvm.maximumnum.f64(double %Val0, double %Val1)
      declare x86_fp80  @llvm.maximumnum.f80(x86_fp80 %Val0, x86_fp80 %Val1)
      declare fp128     @llvm.maximumnum.f128(fp128 %Val0, fp128 %Val1)
      declare ppc_fp128 @llvm.maximumnum.ppcf128(ppc_fp128 %Val0, ppc_fp128 %Val1)

Overview:
"""""""""

The '``llvm.maximumnum.*``' intrinsics return the maximum of the two
arguments, not propagating NaNs and treating -0.0 as less than +0.0.


Arguments:
""""""""""

The arguments and return value are floating-point numbers of the same
type.

Semantics:
""""""""""
If both operands are NaNs (including sNaN), returns qNaN. If one operand
is NaN (including sNaN) and another operand is a number, return the number.
Otherwise returns the greater of the two arguments. -0.0 is considered to
be less than +0.0 for this intrinsic.

Note that these are the semantics of maximumNumber specified in IEEE 754-2019.

It has some differences with '``llvm.maxnum.*``':
1)'``llvm.maxnum.*``' will return qNaN if either operand is sNaN.
2)'``llvm.maxnum*``' may return either one if we compare +0.0 vs -0.0.

.. _int_copysign:

'``llvm.copysign.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.copysign`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.copysign.f32(float  %Mag, float  %Sgn)
      declare double    @llvm.copysign.f64(double %Mag, double %Sgn)
      declare x86_fp80  @llvm.copysign.f80(x86_fp80  %Mag, x86_fp80  %Sgn)
      declare fp128     @llvm.copysign.f128(fp128 %Mag, fp128 %Sgn)
      declare ppc_fp128 @llvm.copysign.ppcf128(ppc_fp128  %Mag, ppc_fp128  %Sgn)

Overview:
"""""""""

The '``llvm.copysign.*``' intrinsics return a value with the magnitude of the
first operand and the sign of the second operand.

Arguments:
""""""""""

The arguments and return value are floating-point numbers of the same
type.

Semantics:
""""""""""

This function returns the same values as the libm ``copysign``
functions would, and handles error conditions in the same way.
The returned value is completely identical to the first operand except for the
sign bit; in particular, if the input is a NaN, then the quiet/signaling bit and
payload are perfectly preserved.

.. _int_floor:

'``llvm.floor.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.floor`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.floor.f32(float  %Val)
      declare double    @llvm.floor.f64(double %Val)
      declare x86_fp80  @llvm.floor.f80(x86_fp80  %Val)
      declare fp128     @llvm.floor.f128(fp128 %Val)
      declare ppc_fp128 @llvm.floor.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.floor.*``' intrinsics return the floor of the operand.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same
type.

Semantics:
""""""""""

This function returns the same values as the libm ``floor`` functions
would, and handles error conditions in the same way.

.. _int_ceil:

'``llvm.ceil.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.ceil`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.ceil.f32(float  %Val)
      declare double    @llvm.ceil.f64(double %Val)
      declare x86_fp80  @llvm.ceil.f80(x86_fp80  %Val)
      declare fp128     @llvm.ceil.f128(fp128 %Val)
      declare ppc_fp128 @llvm.ceil.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.ceil.*``' intrinsics return the ceiling of the operand.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same
type.

Semantics:
""""""""""

This function returns the same values as the libm ``ceil`` functions
would, and handles error conditions in the same way.


.. _int_llvm_trunc:

'``llvm.trunc.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.trunc`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.trunc.f32(float  %Val)
      declare double    @llvm.trunc.f64(double %Val)
      declare x86_fp80  @llvm.trunc.f80(x86_fp80  %Val)
      declare fp128     @llvm.trunc.f128(fp128 %Val)
      declare ppc_fp128 @llvm.trunc.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.trunc.*``' intrinsics returns the operand rounded to the
nearest integer not larger in magnitude than the operand.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same
type.

Semantics:
""""""""""

This function returns the same values as the libm ``trunc`` functions
would, and handles error conditions in the same way.

.. _int_rint:

'``llvm.rint.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.rint`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.rint.f32(float  %Val)
      declare double    @llvm.rint.f64(double %Val)
      declare x86_fp80  @llvm.rint.f80(x86_fp80  %Val)
      declare fp128     @llvm.rint.f128(fp128 %Val)
      declare ppc_fp128 @llvm.rint.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.rint.*``' intrinsics returns the operand rounded to the
nearest integer. It may raise an inexact floating-point exception if the
operand isn't an integer.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same
type.

Semantics:
""""""""""

This function returns the same values as the libm ``rint`` functions
would, and handles error conditions in the same way. Since LLVM assumes the
:ref:`default floating-point environment <floatenv>`, the rounding mode is
assumed to be set to "nearest", so halfway cases are rounded to the even
integer. Use :ref:`Constrained Floating-Point Intrinsics <constrainedfp>`
to avoid that assumption.

.. _int_nearbyint:

'``llvm.nearbyint.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.nearbyint`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.nearbyint.f32(float  %Val)
      declare double    @llvm.nearbyint.f64(double %Val)
      declare x86_fp80  @llvm.nearbyint.f80(x86_fp80  %Val)
      declare fp128     @llvm.nearbyint.f128(fp128 %Val)
      declare ppc_fp128 @llvm.nearbyint.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.nearbyint.*``' intrinsics returns the operand rounded to the
nearest integer.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same
type.

Semantics:
""""""""""

This function returns the same values as the libm ``nearbyint``
functions would, and handles error conditions in the same way. Since LLVM
assumes the :ref:`default floating-point environment <floatenv>`, the rounding
mode is assumed to be set to "nearest", so halfway cases are rounded to the even
integer. Use :ref:`Constrained Floating-Point Intrinsics <constrainedfp>` to
avoid that assumption.

.. _int_round:

'``llvm.round.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.round`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.round.f32(float  %Val)
      declare double    @llvm.round.f64(double %Val)
      declare x86_fp80  @llvm.round.f80(x86_fp80  %Val)
      declare fp128     @llvm.round.f128(fp128 %Val)
      declare ppc_fp128 @llvm.round.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.round.*``' intrinsics returns the operand rounded to the
nearest integer.

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same
type.

Semantics:
""""""""""

This function returns the same values as the libm ``round``
functions would, and handles error conditions in the same way.

.. _int_roundeven:

'``llvm.roundeven.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.roundeven`` on any
floating-point or vector of floating-point type. Not all targets support
all types however.

::

      declare float     @llvm.roundeven.f32(float  %Val)
      declare double    @llvm.roundeven.f64(double %Val)
      declare x86_fp80  @llvm.roundeven.f80(x86_fp80  %Val)
      declare fp128     @llvm.roundeven.f128(fp128 %Val)
      declare ppc_fp128 @llvm.roundeven.ppcf128(ppc_fp128  %Val)

Overview:
"""""""""

The '``llvm.roundeven.*``' intrinsics returns the operand rounded to the nearest
integer in floating-point format rounding halfway cases to even (that is, to the
nearest value that is an even integer).

Arguments:
""""""""""

The argument and return value are floating-point numbers of the same type.

Semantics:
""""""""""

This function implements IEEE-754 operation ``roundToIntegralTiesToEven``. It
also behaves in the same way as C standard function ``roundeven``, including
that it disregards rounding mode and does not raise floating point exceptions.


'``llvm.lround.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.lround`` on any
floating-point type or vector of floating-point type. Not all targets
support all types however.

::

      declare i32 @llvm.lround.i32.f32(float %Val)
      declare i32 @llvm.lround.i32.f64(double %Val)
      declare i32 @llvm.lround.i32.f80(float %Val)
      declare i32 @llvm.lround.i32.f128(double %Val)
      declare i32 @llvm.lround.i32.ppcf128(double %Val)

      declare i64 @llvm.lround.i64.f32(float %Val)
      declare i64 @llvm.lround.i64.f64(double %Val)
      declare i64 @llvm.lround.i64.f80(float %Val)
      declare i64 @llvm.lround.i64.f128(double %Val)
      declare i64 @llvm.lround.i64.ppcf128(double %Val)

Overview:
"""""""""

The '``llvm.lround.*``' intrinsics return the operand rounded to the nearest
integer with ties away from zero.


Arguments:
""""""""""

The argument is a floating-point number and the return value is an integer
type.

Semantics:
""""""""""

This function returns the same values as the libm ``lround`` functions
would, but without setting errno. If the rounded value is too large to
be stored in the result type, the return value is a non-deterministic
value (equivalent to `freeze poison`).

'``llvm.llround.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.llround`` on any
floating-point type. Not all targets support all types however.

::

      declare i64 @llvm.llround.i64.f32(float %Val)
      declare i64 @llvm.llround.i64.f64(double %Val)
      declare i64 @llvm.llround.i64.f80(float %Val)
      declare i64 @llvm.llround.i64.f128(double %Val)
      declare i64 @llvm.llround.i64.ppcf128(double %Val)

Overview:
"""""""""

The '``llvm.llround.*``' intrinsics return the operand rounded to the nearest
integer with ties away from zero.

Arguments:
""""""""""

The argument is a floating-point number and the return value is an integer
type.

Semantics:
""""""""""

This function returns the same values as the libm ``llround``
functions would, but without setting errno. If the rounded value is
too large to be stored in the result type, the return value is a
non-deterministic value (equivalent to `freeze poison`).

.. _int_lrint:

'``llvm.lrint.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.lrint`` on any
floating-point type or vector of floating-point type. Not all targets
support all types however.

::

      declare i32 @llvm.lrint.i32.f32(float %Val)
      declare i32 @llvm.lrint.i32.f64(double %Val)
      declare i32 @llvm.lrint.i32.f80(float %Val)
      declare i32 @llvm.lrint.i32.f128(double %Val)
      declare i32 @llvm.lrint.i32.ppcf128(double %Val)

      declare i64 @llvm.lrint.i64.f32(float %Val)
      declare i64 @llvm.lrint.i64.f64(double %Val)
      declare i64 @llvm.lrint.i64.f80(float %Val)
      declare i64 @llvm.lrint.i64.f128(double %Val)
      declare i64 @llvm.lrint.i64.ppcf128(double %Val)

Overview:
"""""""""

The '``llvm.lrint.*``' intrinsics return the operand rounded to the nearest
integer.


Arguments:
""""""""""

The argument is a floating-point number and the return value is an integer
type.

Semantics:
""""""""""

This function returns the same values as the libm ``lrint`` functions
would, but without setting errno. If the rounded value is too large to
be stored in the result type, the return value is a non-deterministic
value (equivalent to `freeze poison`).

.. _int_llrint:

'``llvm.llrint.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.llrint`` on any
floating-point type or vector of floating-point type. Not all targets
support all types however.

::

      declare i64 @llvm.llrint.i64.f32(float %Val)
      declare i64 @llvm.llrint.i64.f64(double %Val)
      declare i64 @llvm.llrint.i64.f80(float %Val)
      declare i64 @llvm.llrint.i64.f128(double %Val)
      declare i64 @llvm.llrint.i64.ppcf128(double %Val)

Overview:
"""""""""

The '``llvm.llrint.*``' intrinsics return the operand rounded to the nearest
integer.

Arguments:
""""""""""

The argument is a floating-point number and the return value is an integer
type.

Semantics:
""""""""""

This function returns the same values as the libm ``llrint`` functions
would, but without setting errno. If the rounded value is too large to
be stored in the result type, the return value is a non-deterministic
value (equivalent to `freeze poison`).

Bit Manipulation Intrinsics
---------------------------

LLVM provides intrinsics for a few important bit manipulation
operations. These allow efficient code generation for some algorithms.

.. _int_bitreverse:

'``llvm.bitreverse.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic function. You can use bitreverse on any
integer type.

::

      declare i16 @llvm.bitreverse.i16(i16 <id>)
      declare i32 @llvm.bitreverse.i32(i32 <id>)
      declare i64 @llvm.bitreverse.i64(i64 <id>)
      declare <4 x i32> @llvm.bitreverse.v4i32(<4 x i32> <id>)

Overview:
"""""""""

The '``llvm.bitreverse``' family of intrinsics is used to reverse the
bitpattern of an integer value or vector of integer values; for example
``0b10110110`` becomes ``0b01101101``.

Semantics:
""""""""""

The ``llvm.bitreverse.iN`` intrinsic returns an iN value that has bit
``M`` in the input moved to bit ``N-M-1`` in the output. The vector
intrinsics, such as ``llvm.bitreverse.v4i32``, operate on a per-element
basis and the element order is not affected.

.. _int_bswap:

'``llvm.bswap.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic function. You can use bswap on any
integer type that is an even number of bytes (i.e. BitWidth % 16 == 0).

::

      declare i16 @llvm.bswap.i16(i16 <id>)
      declare i32 @llvm.bswap.i32(i32 <id>)
      declare i64 @llvm.bswap.i64(i64 <id>)
      declare <4 x i32> @llvm.bswap.v4i32(<4 x i32> <id>)

Overview:
"""""""""

The '``llvm.bswap``' family of intrinsics is used to byte swap an integer
value or vector of integer values with an even number of bytes (positive
multiple of 16 bits).

Semantics:
""""""""""

The ``llvm.bswap.i16`` intrinsic returns an i16 value that has the high
and low byte of the input i16 swapped. Similarly, the ``llvm.bswap.i32``
intrinsic returns an i32 value that has the four bytes of the input i32
swapped, so that if the input bytes are numbered 0, 1, 2, 3 then the
returned i32 will have its bytes in 3, 2, 1, 0 order. The
``llvm.bswap.i48``, ``llvm.bswap.i64`` and other intrinsics extend this
concept to additional even-byte lengths (6 bytes, 8 bytes and more,
respectively). The vector intrinsics, such as ``llvm.bswap.v4i32``,
operate on a per-element basis and the element order is not affected.

.. _int_ctpop:

'``llvm.ctpop.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use llvm.ctpop on any integer
bit width, or on any vector with integer elements. Not all targets
support all bit widths or vector types, however.

::

      declare i8 @llvm.ctpop.i8(i8  <src>)
      declare i16 @llvm.ctpop.i16(i16 <src>)
      declare i32 @llvm.ctpop.i32(i32 <src>)
      declare i64 @llvm.ctpop.i64(i64 <src>)
      declare i256 @llvm.ctpop.i256(i256 <src>)
      declare <2 x i32> @llvm.ctpop.v2i32(<2 x i32> <src>)

Overview:
"""""""""

The '``llvm.ctpop``' family of intrinsics counts the number of bits set
in a value.

Arguments:
""""""""""

The only argument is the value to be counted. The argument may be of any
integer type, or a vector with integer elements. The return type must
match the argument type.

Semantics:
""""""""""

The '``llvm.ctpop``' intrinsic counts the 1's in a variable, or within
each element of a vector.

.. _int_ctlz:

'``llvm.ctlz.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.ctlz`` on any
integer bit width, or any vector whose elements are integers. Not all
targets support all bit widths or vector types, however.

::

      declare i8   @llvm.ctlz.i8  (i8   <src>, i1 <is_zero_poison>)
      declare <2 x i37> @llvm.ctlz.v2i37(<2 x i37> <src>, i1 <is_zero_poison>)

Overview:
"""""""""

The '``llvm.ctlz``' family of intrinsic functions counts the number of
leading zeros in a variable.

Arguments:
""""""""""

The first argument is the value to be counted. This argument may be of
any integer type, or a vector with integer element type. The return
type must match the first argument type.

The second argument is a constant flag that indicates whether the intrinsic
returns a valid result if the first argument is zero. If the first
argument is zero and the second argument is true, the result is poison.
Historically some architectures did not provide a defined result for zero
values as efficiently, and many algorithms are now predicated on avoiding
zero-value inputs.

Semantics:
""""""""""

The '``llvm.ctlz``' intrinsic counts the leading (most significant)
zeros in a variable, or within each element of the vector. If
``src == 0`` then the result is the size in bits of the type of ``src``
if ``is_zero_poison == 0`` and ``poison`` otherwise. For example,
``llvm.ctlz(i32 2) = 30``.

.. _int_cttz:

'``llvm.cttz.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.cttz`` on any
integer bit width, or any vector of integer elements. Not all targets
support all bit widths or vector types, however.

::

      declare i42   @llvm.cttz.i42  (i42   <src>, i1 <is_zero_poison>)
      declare <2 x i32> @llvm.cttz.v2i32(<2 x i32> <src>, i1 <is_zero_poison>)

Overview:
"""""""""

The '``llvm.cttz``' family of intrinsic functions counts the number of
trailing zeros.

Arguments:
""""""""""

The first argument is the value to be counted. This argument may be of
any integer type, or a vector with integer element type. The return
type must match the first argument type.

The second argument is a constant flag that indicates whether the intrinsic
returns a valid result if the first argument is zero. If the first
argument is zero and the second argument is true, the result is poison.
Historically some architectures did not provide a defined result for zero
values as efficiently, and many algorithms are now predicated on avoiding
zero-value inputs.

Semantics:
""""""""""

The '``llvm.cttz``' intrinsic counts the trailing (least significant)
zeros in a variable, or within each element of a vector. If ``src == 0``
then the result is the size in bits of the type of ``src`` if
``is_zero_poison == 0`` and ``poison`` otherwise. For example,
``llvm.cttz(2) = 1``.

.. _int_overflow:

.. _int_fshl:

'``llvm.fshl.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.fshl`` on any
integer bit width or any vector of integer elements. Not all targets
support all bit widths or vector types, however.

::

      declare i8  @llvm.fshl.i8 (i8 %a, i8 %b, i8 %c)
      declare i64 @llvm.fshl.i64(i64 %a, i64 %b, i64 %c)
      declare <2 x i32> @llvm.fshl.v2i32(<2 x i32> %a, <2 x i32> %b, <2 x i32> %c)

Overview:
"""""""""

The '``llvm.fshl``' family of intrinsic functions performs a funnel shift left:
the first two values are concatenated as { %a : %b } (%a is the most significant
bits of the wide value), the combined value is shifted left, and the most
significant bits are extracted to produce a result that is the same size as the
original arguments. If the first 2 arguments are identical, this is equivalent
to a rotate left operation. For vector types, the operation occurs for each
element of the vector. The shift argument is treated as an unsigned amount
modulo the element size of the arguments.

Arguments:
""""""""""

The first two arguments are the values to be concatenated. The third
argument is the shift amount. The arguments may be any integer type or a
vector with integer element type. All arguments and the return value must
have the same type.

Example:
""""""""

.. code-block:: text

      %r = call i8 @llvm.fshl.i8(i8 %x, i8 %y, i8 %z)  ; %r = i8: msb_extract((concat(x, y) << (z % 8)), 8)
      %r = call i8 @llvm.fshl.i8(i8 255, i8 0, i8 15)  ; %r = i8: 128 (0b10000000)
      %r = call i8 @llvm.fshl.i8(i8 15, i8 15, i8 11)  ; %r = i8: 120 (0b01111000)
      %r = call i8 @llvm.fshl.i8(i8 0, i8 255, i8 8)   ; %r = i8: 0   (0b00000000)

.. _int_fshr:

'``llvm.fshr.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.fshr`` on any
integer bit width or any vector of integer elements. Not all targets
support all bit widths or vector types, however.

::

      declare i8  @llvm.fshr.i8 (i8 %a, i8 %b, i8 %c)
      declare i64 @llvm.fshr.i64(i64 %a, i64 %b, i64 %c)
      declare <2 x i32> @llvm.fshr.v2i32(<2 x i32> %a, <2 x i32> %b, <2 x i32> %c)

Overview:
"""""""""

The '``llvm.fshr``' family of intrinsic functions performs a funnel shift right:
the first two values are concatenated as { %a : %b } (%a is the most significant
bits of the wide value), the combined value is shifted right, and the least
significant bits are extracted to produce a result that is the same size as the
original arguments. If the first 2 arguments are identical, this is equivalent
to a rotate right operation. For vector types, the operation occurs for each
element of the vector. The shift argument is treated as an unsigned amount
modulo the element size of the arguments.

Arguments:
""""""""""

The first two arguments are the values to be concatenated. The third
argument is the shift amount. The arguments may be any integer type or a
vector with integer element type. All arguments and the return value must
have the same type.

Example:
""""""""

.. code-block:: text

      %r = call i8 @llvm.fshr.i8(i8 %x, i8 %y, i8 %z)  ; %r = i8: lsb_extract((concat(x, y) >> (z % 8)), 8)
      %r = call i8 @llvm.fshr.i8(i8 255, i8 0, i8 15)  ; %r = i8: 254 (0b11111110)
      %r = call i8 @llvm.fshr.i8(i8 15, i8 15, i8 11)  ; %r = i8: 225 (0b11100001)
      %r = call i8 @llvm.fshr.i8(i8 0, i8 255, i8 8)   ; %r = i8: 255 (0b11111111)

Arithmetic with Overflow Intrinsics
-----------------------------------

LLVM provides intrinsics for fast arithmetic overflow checking.

Each of these intrinsics returns a two-element struct. The first
element of this struct contains the result of the corresponding
arithmetic operation modulo 2\ :sup:`n`\ , where n is the bit width of
the result. Therefore, for example, the first element of the struct
returned by ``llvm.sadd.with.overflow.i32`` is always the same as the
result of a 32-bit ``add`` instruction with the same operands, where
the ``add`` is *not* modified by an ``nsw`` or ``nuw`` flag.

The second element of the result is an ``i1`` that is 1 if the
arithmetic operation overflowed and 0 otherwise. An operation
overflows if, for any values of its operands ``A`` and ``B`` and for
any ``N`` larger than the operands' width, ``ext(A op B) to iN`` is
not equal to ``(ext(A) to iN) op (ext(B) to iN)`` where ``ext`` is
``sext`` for signed overflow and ``zext`` for unsigned overflow, and
``op`` is the underlying arithmetic operation.

The behavior of these intrinsics is well-defined for all argument
values.

'``llvm.sadd.with.overflow.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.sadd.with.overflow``
on any integer bit width or vectors of integers.

::

      declare {i16, i1} @llvm.sadd.with.overflow.i16(i16 %a, i16 %b)
      declare {i32, i1} @llvm.sadd.with.overflow.i32(i32 %a, i32 %b)
      declare {i64, i1} @llvm.sadd.with.overflow.i64(i64 %a, i64 %b)
      declare {<4 x i32>, <4 x i1>} @llvm.sadd.with.overflow.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview:
"""""""""

The '``llvm.sadd.with.overflow``' family of intrinsic functions perform
a signed addition of the two arguments, and indicate whether an overflow
occurred during the signed summation.

Arguments:
""""""""""

The arguments (%a and %b) and the first element of the result structure
may be of integer types of any bit width, but they must have the same
bit width. The second element of the result structure must be of type
``i1``. ``%a`` and ``%b`` are the two values that will undergo signed
addition.

Semantics:
""""""""""

The '``llvm.sadd.with.overflow``' family of intrinsic functions perform
a signed addition of the two variables. They return a structure --- the
first element of which is the signed summation, and the second element
of which is a bit specifying if the signed summation resulted in an
overflow.

Examples:
"""""""""

.. code-block:: llvm

      %res = call {i32, i1} @llvm.sadd.with.overflow.i32(i32 %a, i32 %b)
      %sum = extractvalue {i32, i1} %res, 0
      %obit = extractvalue {i32, i1} %res, 1
      br i1 %obit, label %overflow, label %normal

'``llvm.uadd.with.overflow.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.uadd.with.overflow``
on any integer bit width or vectors of integers.

::

      declare {i16, i1} @llvm.uadd.with.overflow.i16(i16 %a, i16 %b)
      declare {i32, i1} @llvm.uadd.with.overflow.i32(i32 %a, i32 %b)
      declare {i64, i1} @llvm.uadd.with.overflow.i64(i64 %a, i64 %b)
      declare {<4 x i32>, <4 x i1>} @llvm.uadd.with.overflow.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview:
"""""""""

The '``llvm.uadd.with.overflow``' family of intrinsic functions perform
an unsigned addition of the two arguments, and indicate whether a carry
occurred during the unsigned summation.

Arguments:
""""""""""

The arguments (%a and %b) and the first element of the result structure
may be of integer types of any bit width, but they must have the same
bit width. The second element of the result structure must be of type
``i1``. ``%a`` and ``%b`` are the two values that will undergo unsigned
addition.

Semantics:
""""""""""

The '``llvm.uadd.with.overflow``' family of intrinsic functions perform
an unsigned addition of the two arguments. They return a structure --- the
first element of which is the sum, and the second element of which is a
bit specifying if the unsigned summation resulted in a carry.

Examples:
"""""""""

.. code-block:: llvm

      %res = call {i32, i1} @llvm.uadd.with.overflow.i32(i32 %a, i32 %b)
      %sum = extractvalue {i32, i1} %res, 0
      %obit = extractvalue {i32, i1} %res, 1
      br i1 %obit, label %carry, label %normal

'``llvm.ssub.with.overflow.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.ssub.with.overflow``
on any integer bit width or vectors of integers.

::

      declare {i16, i1} @llvm.ssub.with.overflow.i16(i16 %a, i16 %b)
      declare {i32, i1} @llvm.ssub.with.overflow.i32(i32 %a, i32 %b)
      declare {i64, i1} @llvm.ssub.with.overflow.i64(i64 %a, i64 %b)
      declare {<4 x i32>, <4 x i1>} @llvm.ssub.with.overflow.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview:
"""""""""

The '``llvm.ssub.with.overflow``' family of intrinsic functions perform
a signed subtraction of the two arguments, and indicate whether an
overflow occurred during the signed subtraction.

Arguments:
""""""""""

The arguments (%a and %b) and the first element of the result structure
may be of integer types of any bit width, but they must have the same
bit width. The second element of the result structure must be of type
``i1``. ``%a`` and ``%b`` are the two values that will undergo signed
subtraction.

Semantics:
""""""""""

The '``llvm.ssub.with.overflow``' family of intrinsic functions perform
a signed subtraction of the two arguments. They return a structure --- the
first element of which is the subtraction, and the second element of
which is a bit specifying if the signed subtraction resulted in an
overflow.

Examples:
"""""""""

.. code-block:: llvm

      %res = call {i32, i1} @llvm.ssub.with.overflow.i32(i32 %a, i32 %b)
      %sum = extractvalue {i32, i1} %res, 0
      %obit = extractvalue {i32, i1} %res, 1
      br i1 %obit, label %overflow, label %normal

'``llvm.usub.with.overflow.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.usub.with.overflow``
on any integer bit width or vectors of integers.

::

      declare {i16, i1} @llvm.usub.with.overflow.i16(i16 %a, i16 %b)
      declare {i32, i1} @llvm.usub.with.overflow.i32(i32 %a, i32 %b)
      declare {i64, i1} @llvm.usub.with.overflow.i64(i64 %a, i64 %b)
      declare {<4 x i32>, <4 x i1>} @llvm.usub.with.overflow.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview:
"""""""""

The '``llvm.usub.with.overflow``' family of intrinsic functions perform
an unsigned subtraction of the two arguments, and indicate whether an
overflow occurred during the unsigned subtraction.

Arguments:
""""""""""

The arguments (%a and %b) and the first element of the result structure
may be of integer types of any bit width, but they must have the same
bit width. The second element of the result structure must be of type
``i1``. ``%a`` and ``%b`` are the two values that will undergo unsigned
subtraction.

Semantics:
""""""""""

The '``llvm.usub.with.overflow``' family of intrinsic functions perform
an unsigned subtraction of the two arguments. They return a structure ---
the first element of which is the subtraction, and the second element of
which is a bit specifying if the unsigned subtraction resulted in an
overflow.

Examples:
"""""""""

.. code-block:: llvm

      %res = call {i32, i1} @llvm.usub.with.overflow.i32(i32 %a, i32 %b)
      %sum = extractvalue {i32, i1} %res, 0
      %obit = extractvalue {i32, i1} %res, 1
      br i1 %obit, label %overflow, label %normal

'``llvm.smul.with.overflow.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.smul.with.overflow``
on any integer bit width or vectors of integers.

::

      declare {i16, i1} @llvm.smul.with.overflow.i16(i16 %a, i16 %b)
      declare {i32, i1} @llvm.smul.with.overflow.i32(i32 %a, i32 %b)
      declare {i64, i1} @llvm.smul.with.overflow.i64(i64 %a, i64 %b)
      declare {<4 x i32>, <4 x i1>} @llvm.smul.with.overflow.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview:
"""""""""

The '``llvm.smul.with.overflow``' family of intrinsic functions perform
a signed multiplication of the two arguments, and indicate whether an
overflow occurred during the signed multiplication.

Arguments:
""""""""""

The arguments (%a and %b) and the first element of the result structure
may be of integer types of any bit width, but they must have the same
bit width. The second element of the result structure must be of type
``i1``. ``%a`` and ``%b`` are the two values that will undergo signed
multiplication.

Semantics:
""""""""""

The '``llvm.smul.with.overflow``' family of intrinsic functions perform
a signed multiplication of the two arguments. They return a structure ---
the first element of which is the multiplication, and the second element
of which is a bit specifying if the signed multiplication resulted in an
overflow.

Examples:
"""""""""

.. code-block:: llvm

      %res = call {i32, i1} @llvm.smul.with.overflow.i32(i32 %a, i32 %b)
      %sum = extractvalue {i32, i1} %res, 0
      %obit = extractvalue {i32, i1} %res, 1
      br i1 %obit, label %overflow, label %normal

'``llvm.umul.with.overflow.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.umul.with.overflow``
on any integer bit width or vectors of integers.

::

      declare {i16, i1} @llvm.umul.with.overflow.i16(i16 %a, i16 %b)
      declare {i32, i1} @llvm.umul.with.overflow.i32(i32 %a, i32 %b)
      declare {i64, i1} @llvm.umul.with.overflow.i64(i64 %a, i64 %b)
      declare {<4 x i32>, <4 x i1>} @llvm.umul.with.overflow.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview:
"""""""""

The '``llvm.umul.with.overflow``' family of intrinsic functions perform
a unsigned multiplication of the two arguments, and indicate whether an
overflow occurred during the unsigned multiplication.

Arguments:
""""""""""

The arguments (%a and %b) and the first element of the result structure
may be of integer types of any bit width, but they must have the same
bit width. The second element of the result structure must be of type
``i1``. ``%a`` and ``%b`` are the two values that will undergo unsigned
multiplication.

Semantics:
""""""""""

The '``llvm.umul.with.overflow``' family of intrinsic functions perform
an unsigned multiplication of the two arguments. They return a structure ---
the first element of which is the multiplication, and the second
element of which is a bit specifying if the unsigned multiplication
resulted in an overflow.

Examples:
"""""""""

.. code-block:: llvm

      %res = call {i32, i1} @llvm.umul.with.overflow.i32(i32 %a, i32 %b)
      %sum = extractvalue {i32, i1} %res, 0
      %obit = extractvalue {i32, i1} %res, 1
      br i1 %obit, label %overflow, label %normal

Saturation Arithmetic Intrinsics
---------------------------------

Saturation arithmetic is a version of arithmetic in which operations are
limited to a fixed range between a minimum and maximum value. If the result of
an operation is greater than the maximum value, the result is set (or
"clamped") to this maximum. If it is below the minimum, it is clamped to this
minimum.

.. _int_sadd_sat:

'``llvm.sadd.sat.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax
"""""""

This is an overloaded intrinsic. You can use ``llvm.sadd.sat``
on any integer bit width or vectors of integers.

::

      declare i16 @llvm.sadd.sat.i16(i16 %a, i16 %b)
      declare i32 @llvm.sadd.sat.i32(i32 %a, i32 %b)
      declare i64 @llvm.sadd.sat.i64(i64 %a, i64 %b)
      declare <4 x i32> @llvm.sadd.sat.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview
"""""""""

The '``llvm.sadd.sat``' family of intrinsic functions perform signed
saturating addition on the 2 arguments.

Arguments
""""""""""

The arguments (%a and %b) and the result may be of integer types of any bit
width, but they must have the same bit width. ``%a`` and ``%b`` are the two
values that will undergo signed addition.

Semantics:
""""""""""

The maximum value this operation can clamp to is the largest signed value
representable by the bit width of the arguments. The minimum value is the
smallest signed value representable by this bit width.


Examples
"""""""""

.. code-block:: llvm

      %res = call i4 @llvm.sadd.sat.i4(i4 1, i4 2)  ; %res = 3
      %res = call i4 @llvm.sadd.sat.i4(i4 5, i4 6)  ; %res = 7
      %res = call i4 @llvm.sadd.sat.i4(i4 -4, i4 2)  ; %res = -2
      %res = call i4 @llvm.sadd.sat.i4(i4 -4, i4 -5)  ; %res = -8


.. _int_uadd_sat:

'``llvm.uadd.sat.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax
"""""""

This is an overloaded intrinsic. You can use ``llvm.uadd.sat``
on any integer bit width or vectors of integers.

::

      declare i16 @llvm.uadd.sat.i16(i16 %a, i16 %b)
      declare i32 @llvm.uadd.sat.i32(i32 %a, i32 %b)
      declare i64 @llvm.uadd.sat.i64(i64 %a, i64 %b)
      declare <4 x i32> @llvm.uadd.sat.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview
"""""""""

The '``llvm.uadd.sat``' family of intrinsic functions perform unsigned
saturating addition on the 2 arguments.

Arguments
""""""""""

The arguments (%a and %b) and the result may be of integer types of any bit
width, but they must have the same bit width. ``%a`` and ``%b`` are the two
values that will undergo unsigned addition.

Semantics:
""""""""""

The maximum value this operation can clamp to is the largest unsigned value
representable by the bit width of the arguments. Because this is an unsigned
operation, the result will never saturate towards zero.


Examples
"""""""""

.. code-block:: llvm

      %res = call i4 @llvm.uadd.sat.i4(i4 1, i4 2)  ; %res = 3
      %res = call i4 @llvm.uadd.sat.i4(i4 5, i4 6)  ; %res = 11
      %res = call i4 @llvm.uadd.sat.i4(i4 8, i4 8)  ; %res = 15


.. _int_ssub_sat:

'``llvm.ssub.sat.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax
"""""""

This is an overloaded intrinsic. You can use ``llvm.ssub.sat``
on any integer bit width or vectors of integers.

::

      declare i16 @llvm.ssub.sat.i16(i16 %a, i16 %b)
      declare i32 @llvm.ssub.sat.i32(i32 %a, i32 %b)
      declare i64 @llvm.ssub.sat.i64(i64 %a, i64 %b)
      declare <4 x i32> @llvm.ssub.sat.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview
"""""""""

The '``llvm.ssub.sat``' family of intrinsic functions perform signed
saturating subtraction on the 2 arguments.

Arguments
""""""""""

The arguments (%a and %b) and the result may be of integer types of any bit
width, but they must have the same bit width. ``%a`` and ``%b`` are the two
values that will undergo signed subtraction.

Semantics:
""""""""""

The maximum value this operation can clamp to is the largest signed value
representable by the bit width of the arguments. The minimum value is the
smallest signed value representable by this bit width.


Examples
"""""""""

.. code-block:: llvm

      %res = call i4 @llvm.ssub.sat.i4(i4 2, i4 1)  ; %res = 1
      %res = call i4 @llvm.ssub.sat.i4(i4 2, i4 6)  ; %res = -4
      %res = call i4 @llvm.ssub.sat.i4(i4 -4, i4 5)  ; %res = -8
      %res = call i4 @llvm.ssub.sat.i4(i4 4, i4 -5)  ; %res = 7


.. _int_usub_sat:

'``llvm.usub.sat.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax
"""""""

This is an overloaded intrinsic. You can use ``llvm.usub.sat``
on any integer bit width or vectors of integers.

::

      declare i16 @llvm.usub.sat.i16(i16 %a, i16 %b)
      declare i32 @llvm.usub.sat.i32(i32 %a, i32 %b)
      declare i64 @llvm.usub.sat.i64(i64 %a, i64 %b)
      declare <4 x i32> @llvm.usub.sat.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview
"""""""""

The '``llvm.usub.sat``' family of intrinsic functions perform unsigned
saturating subtraction on the 2 arguments.

Arguments
""""""""""

The arguments (%a and %b) and the result may be of integer types of any bit
width, but they must have the same bit width. ``%a`` and ``%b`` are the two
values that will undergo unsigned subtraction.

Semantics:
""""""""""

The minimum value this operation can clamp to is 0, which is the smallest
unsigned value representable by the bit width of the unsigned arguments.
Because this is an unsigned operation, the result will never saturate towards
the largest possible value representable by this bit width.


Examples
"""""""""

.. code-block:: llvm

      %res = call i4 @llvm.usub.sat.i4(i4 2, i4 1)  ; %res = 1
      %res = call i4 @llvm.usub.sat.i4(i4 2, i4 6)  ; %res = 0


'``llvm.sshl.sat.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax
"""""""

This is an overloaded intrinsic. You can use ``llvm.sshl.sat``
on integers or vectors of integers of any bit width.

::

      declare i16 @llvm.sshl.sat.i16(i16 %a, i16 %b)
      declare i32 @llvm.sshl.sat.i32(i32 %a, i32 %b)
      declare i64 @llvm.sshl.sat.i64(i64 %a, i64 %b)
      declare <4 x i32> @llvm.sshl.sat.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview
"""""""""

The '``llvm.sshl.sat``' family of intrinsic functions perform signed
saturating left shift on the first argument.

Arguments
""""""""""

The arguments (``%a`` and ``%b``) and the result may be of integer types of any
bit width, but they must have the same bit width. ``%a`` is the value to be
shifted, and ``%b`` is the amount to shift by. If ``b`` is (statically or
dynamically) equal to or larger than the integer bit width of the arguments,
the result is a :ref:`poison value <poisonvalues>`. If the arguments are
vectors, each vector element of ``a`` is shifted by the corresponding shift
amount in ``b``.


Semantics:
""""""""""

The maximum value this operation can clamp to is the largest signed value
representable by the bit width of the arguments. The minimum value is the
smallest signed value representable by this bit width.


Examples
"""""""""

.. code-block:: llvm

      %res = call i4 @llvm.sshl.sat.i4(i4 2, i4 1)  ; %res = 4
      %res = call i4 @llvm.sshl.sat.i4(i4 2, i4 2)  ; %res = 7
      %res = call i4 @llvm.sshl.sat.i4(i4 -5, i4 1)  ; %res = -8
      %res = call i4 @llvm.sshl.sat.i4(i4 -1, i4 1)  ; %res = -2


'``llvm.ushl.sat.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax
"""""""

This is an overloaded intrinsic. You can use ``llvm.ushl.sat``
on integers or vectors of integers of any bit width.

::

      declare i16 @llvm.ushl.sat.i16(i16 %a, i16 %b)
      declare i32 @llvm.ushl.sat.i32(i32 %a, i32 %b)
      declare i64 @llvm.ushl.sat.i64(i64 %a, i64 %b)
      declare <4 x i32> @llvm.ushl.sat.v4i32(<4 x i32> %a, <4 x i32> %b)

Overview
"""""""""

The '``llvm.ushl.sat``' family of intrinsic functions perform unsigned
saturating left shift on the first argument.

Arguments
""""""""""

The arguments (``%a`` and ``%b``) and the result may be of integer types of any
bit width, but they must have the same bit width. ``%a`` is the value to be
shifted, and ``%b`` is the amount to shift by. If ``b`` is (statically or
dynamically) equal to or larger than the integer bit width of the arguments,
the result is a :ref:`poison value <poisonvalues>`. If the arguments are
vectors, each vector element of ``a`` is shifted by the corresponding shift
amount in ``b``.

Semantics:
""""""""""

The maximum value this operation can clamp to is the largest unsigned value
representable by the bit width of the arguments.


Examples
"""""""""

.. code-block:: llvm

      %res = call i4 @llvm.ushl.sat.i4(i4 2, i4 1)  ; %res = 4
      %res = call i4 @llvm.ushl.sat.i4(i4 3, i4 3)  ; %res = 15


Fixed Point Arithmetic Intrinsics
---------------------------------

A fixed point number represents a real data type for a number that has a fixed
number of digits after a radix point (equivalent to the decimal point '.').
The number of digits after the radix point is referred as the `scale`. These
are useful for representing fractional values to a specific precision. The
following intrinsics perform fixed point arithmetic operations on 2 operands
of the same scale, specified as the third argument.

The ``llvm.*mul.fix`` family of intrinsic functions represents a multiplication
of fixed point numbers through scaled integers. Therefore, fixed point
multiplication can be represented as

.. code-block:: llvm

        %result = call i4 @llvm.smul.fix.i4(i4 %a, i4 %b, i32 %scale)

        ; Expands to
        %a2 = sext i4 %a to i8
        %b2 = sext i4 %b to i8
        %mul = mul nsw nuw i8 %a2, %b2
        %scale2 = trunc i32 %scale to i8
        %r = ashr i8 %mul, i8 %scale2  ; this is for a target rounding down towards negative infinity
        %result = trunc i8 %r to i4

The ``llvm.*div.fix`` family of intrinsic functions represents a division of
fixed point numbers through scaled integers. Fixed point division can be
represented as:

.. code-block:: llvm

        %result call i4 @llvm.sdiv.fix.i4(i4 %a, i4 %b, i32 %scale)

        ; Expands to
        %a2 = sext i4 %a to i8
        %b2 = sext i4 %b to i8
        %scale2 = trunc i32 %scale to i8
        %a3 = shl i8 %a2, %scale2
        %r = sdiv i8 %a3, %b2 ; this is for a target rounding towards zero
        %result = trunc i8 %r to i4

For each of these functions, if the result cannot be represented exactly with
the provided scale, the result is rounded. Rounding is unspecified since
preferred rounding may vary for different targets. Rounding is specified
through a target hook. Different pipelines should legalize or optimize this
using the rounding specified by this hook if it is provided. Operations like
constant folding, instruction combining, KnownBits, and ValueTracking should
also use this hook, if provided, and not assume the direction of rounding. A
rounded result must always be within one unit of precision from the true
result. That is, the error between the returned result and the true result must
be less than 1/2^(scale).


'``llvm.smul.fix.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax
"""""""

This is an overloaded intrinsic. You can use ``llvm.smul.fix``
on any integer bit width or vectors of integers.

::

      declare i16 @llvm.smul.fix.i16(i16 %a, i16 %b, i32 %scale)
      declare i32 @llvm.smul.fix.i32(i32 %a, i32 %b, i32 %scale)
      declare i64 @llvm.smul.fix.i64(i64 %a, i64 %b, i32 %scale)
      declare <4 x i32> @llvm.smul.fix.v4i32(<4 x i32> %a, <4 x i32> %b, i32 %scale)

Overview
"""""""""

The '``llvm.smul.fix``' family of intrinsic functions perform signed
fixed point multiplication on 2 arguments of the same scale.

Arguments
""""""""""

The arguments (%a and %b) and the result may be of integer types of any bit
width, but they must have the same bit width. The arguments may also work with
int vectors of the same length and int size. ``%a`` and ``%b`` are the two
values that will undergo signed fixed point multiplication. The argument
``%scale`` represents the scale of both operands, and must be a constant
integer.

Semantics:
""""""""""

This operation performs fixed point multiplication on the 2 arguments of a
specified scale. The result will also be returned in the same scale specified
in the third argument.

If the result value cannot be precisely represented in the given scale, the
value is rounded up or down to the closest representable value. The rounding
direction is unspecified.

It is undefined behavior if the result value does not fit within the range of
the fixed point type.


Examples
"""""""""

.. code-block:: llvm

      %res = call i4 @llvm.smul.fix.i4(i4 3, i4 2, i32 0)  ; %res = 6 (2 x 3 = 6)
      %res = call i4 @llvm.smul.fix.i4(i4 3, i4 2, i32 1)  ; %res = 3 (1.5 x 1 = 1.5)
      %res = call i4 @llvm.smul.fix.i4(i4 3, i4 -2, i32 1)  ; %res = -3 (1.5 x -1 = -1.5)

      ; The result in the following could be rounded up to -2 or down to -2.5
      %res = call i4 @llvm.smul.fix.i4(i4 3, i4 -3, i32 1)  ; %res = -5 (or -4) (1.5 x -1.5 = -2.25)


'``llvm.umul.fix.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax
"""""""

This is an overloaded intrinsic. You can use ``llvm.umul.fix``
on any integer bit width or vectors of integers.

::

      declare i16 @llvm.umul.fix.i16(i16 %a, i16 %b, i32 %scale)
      declare i32 @llvm.umul.fix.i32(i32 %a, i32 %b, i32 %scale)
      declare i64 @llvm.umul.fix.i64(i64 %a, i64 %b, i32 %scale)
      declare <4 x i32> @llvm.umul.fix.v4i32(<4 x i32> %a, <4 x i32> %b, i32 %scale)

Overview
"""""""""

The '``llvm.umul.fix``' family of intrinsic functions perform unsigned
fixed point multiplication on 2 arguments of the same scale.

Arguments
""""""""""

The arguments (%a and %b) and the result may be of integer types of any bit
width, but they must have the same bit width. The arguments may also work with
int vectors of the same length and int size. ``%a`` and ``%b`` are the two
values that will undergo unsigned fixed point multiplication. The argument
``%scale`` represents the scale of both operands, and must be a constant
integer.

Semantics:
""""""""""

This operation performs unsigned fixed point multiplication on the 2 arguments of a
specified scale. The result will also be returned in the same scale specified
in the third argument.

If the result value cannot be precisely represented in the given scale, the
value is rounded up or down to the closest representable value. The rounding
direction is unspecified.

It is undefined behavior if the result value does not fit within the range of
the fixed point type.


Examples
"""""""""

.. code-block:: llvm

      %res = call i4 @llvm.umul.fix.i4(i4 3, i4 2, i32 0)  ; %res = 6 (2 x 3 = 6)
      %res = call i4 @llvm.umul.fix.i4(i4 3, i4 2, i32 1)  ; %res = 3 (1.5 x 1 = 1.5)

      ; The result in the following could be rounded down to 3.5 or up to 4
      %res = call i4 @llvm.umul.fix.i4(i4 15, i4 1, i32 1)  ; %res = 7 (or 8) (7.5 x 0.5 = 3.75)


'``llvm.smul.fix.sat.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax
"""""""

This is an overloaded intrinsic. You can use ``llvm.smul.fix.sat``
on any integer bit width or vectors of integers.

::

      declare i16 @llvm.smul.fix.sat.i16(i16 %a, i16 %b, i32 %scale)
      declare i32 @llvm.smul.fix.sat.i32(i32 %a, i32 %b, i32 %scale)
      declare i64 @llvm.smul.fix.sat.i64(i64 %a, i64 %b, i32 %scale)
      declare <4 x i32> @llvm.smul.fix.sat.v4i32(<4 x i32> %a, <4 x i32> %b, i32 %scale)

Overview
"""""""""

The '``llvm.smul.fix.sat``' family of intrinsic functions perform signed
fixed point saturating multiplication on 2 arguments of the same scale.

Arguments
""""""""""

The arguments (%a and %b) and the result may be of integer types of any bit
width, but they must have the same bit width. ``%a`` and ``%b`` are the two
values that will undergo signed fixed point multiplication. The argument
``%scale`` represents the scale of both operands, and must be a constant
integer.

Semantics:
""""""""""

This operation performs fixed point multiplication on the 2 arguments of a
specified scale. The result will also be returned in the same scale specified
in the third argument.

If the result value cannot be precisely represented in the given scale, the
value is rounded up or down to the closest representable value. The rounding
direction is unspecified.

The maximum value this operation can clamp to is the largest signed value
representable by the bit width of the first 2 arguments. The minimum value is the
smallest signed value representable by this bit width.


Examples
"""""""""

.. code-block:: llvm

      %res = call i4 @llvm.smul.fix.sat.i4(i4 3, i4 2, i32 0)  ; %res = 6 (2 x 3 = 6)
      %res = call i4 @llvm.smul.fix.sat.i4(i4 3, i4 2, i32 1)  ; %res = 3 (1.5 x 1 = 1.5)
      %res = call i4 @llvm.smul.fix.sat.i4(i4 3, i4 -2, i32 1)  ; %res = -3 (1.5 x -1 = -1.5)

      ; The result in the following could be rounded up to -2 or down to -2.5
      %res = call i4 @llvm.smul.fix.sat.i4(i4 3, i4 -3, i32 1)  ; %res = -5 (or -4) (1.5 x -1.5 = -2.25)

      ; Saturation
      %res = call i4 @llvm.smul.fix.sat.i4(i4 7, i4 2, i32 0)  ; %res = 7
      %res = call i4 @llvm.smul.fix.sat.i4(i4 7, i4 4, i32 2)  ; %res = 7
      %res = call i4 @llvm.smul.fix.sat.i4(i4 -8, i4 5, i32 2)  ; %res = -8
      %res = call i4 @llvm.smul.fix.sat.i4(i4 -8, i4 -2, i32 1)  ; %res = 7

      ; Scale can affect the saturation result
      %res = call i4 @llvm.smul.fix.sat.i4(i4 2, i4 4, i32 0)  ; %res = 7 (2 x 4 -> clamped to 7)
      %res = call i4 @llvm.smul.fix.sat.i4(i4 2, i4 4, i32 1)  ; %res = 4 (1 x 2 = 2)


'``llvm.umul.fix.sat.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax
"""""""

This is an overloaded intrinsic. You can use ``llvm.umul.fix.sat``
on any integer bit width or vectors of integers.

::

      declare i16 @llvm.umul.fix.sat.i16(i16 %a, i16 %b, i32 %scale)
      declare i32 @llvm.umul.fix.sat.i32(i32 %a, i32 %b, i32 %scale)
      declare i64 @llvm.umul.fix.sat.i64(i64 %a, i64 %b, i32 %scale)
      declare <4 x i32> @llvm.umul.fix.sat.v4i32(<4 x i32> %a, <4 x i32> %b, i32 %scale)

Overview
"""""""""

The '``llvm.umul.fix.sat``' family of intrinsic functions perform unsigned
fixed point saturating multiplication on 2 arguments of the same scale.

Arguments
""""""""""

The arguments (%a and %b) and the result may be of integer types of any bit
width, but they must have the same bit width. ``%a`` and ``%b`` are the two
values that will undergo unsigned fixed point multiplication. The argument
``%scale`` represents the scale of both operands, and must be a constant
integer.

Semantics:
""""""""""

This operation performs fixed point multiplication on the 2 arguments of a
specified scale. The result will also be returned in the same scale specified
in the third argument.

If the result value cannot be precisely represented in the given scale, the
value is rounded up or down to the closest representable value. The rounding
direction is unspecified.

The maximum value this operation can clamp to is the largest unsigned value
representable by the bit width of the first 2 arguments. The minimum value is the
smallest unsigned value representable by this bit width (zero).


Examples
"""""""""

.. code-block:: llvm

      %res = call i4 @llvm.umul.fix.sat.i4(i4 3, i4 2, i32 0)  ; %res = 6 (2 x 3 = 6)
      %res = call i4 @llvm.umul.fix.sat.i4(i4 3, i4 2, i32 1)  ; %res = 3 (1.5 x 1 = 1.5)

      ; The result in the following could be rounded down to 2 or up to 2.5
      %res = call i4 @llvm.umul.fix.sat.i4(i4 3, i4 3, i32 1)  ; %res = 4 (or 5) (1.5 x 1.5 = 2.25)

      ; Saturation
      %res = call i4 @llvm.umul.fix.sat.i4(i4 8, i4 2, i32 0)  ; %res = 15 (8 x 2 -> clamped to 15)
      %res = call i4 @llvm.umul.fix.sat.i4(i4 8, i4 8, i32 2)  ; %res = 15 (2 x 2 -> clamped to 3.75)

      ; Scale can affect the saturation result
      %res = call i4 @llvm.umul.fix.sat.i4(i4 2, i4 4, i32 0)  ; %res = 7 (2 x 4 -> clamped to 7)
      %res = call i4 @llvm.umul.fix.sat.i4(i4 2, i4 4, i32 1)  ; %res = 4 (1 x 2 = 2)


'``llvm.sdiv.fix.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax
"""""""

This is an overloaded intrinsic. You can use ``llvm.sdiv.fix``
on any integer bit width or vectors of integers.

::

      declare i16 @llvm.sdiv.fix.i16(i16 %a, i16 %b, i32 %scale)
      declare i32 @llvm.sdiv.fix.i32(i32 %a, i32 %b, i32 %scale)
      declare i64 @llvm.sdiv.fix.i64(i64 %a, i64 %b, i32 %scale)
      declare <4 x i32> @llvm.sdiv.fix.v4i32(<4 x i32> %a, <4 x i32> %b, i32 %scale)

Overview
"""""""""

The '``llvm.sdiv.fix``' family of intrinsic functions perform signed
fixed point division on 2 arguments of the same scale.

Arguments
""""""""""

The arguments (%a and %b) and the result may be of integer types of any bit
width, but they must have the same bit width. The arguments may also work with
int vectors of the same length and int size. ``%a`` and ``%b`` are the two
values that will undergo signed fixed point division. The argument
``%scale`` represents the scale of both operands, and must be a constant
integer.

Semantics:
""""""""""

This operation performs fixed point division on the 2 arguments of a
specified scale. The result will also be returned in the same scale specified
in the third argument.

If the result value cannot be precisely represented in the given scale, the
value is rounded up or down to the closest representable value. The rounding
direction is unspecified.

It is undefined behavior if the result value does not fit within the range of
the fixed point type, or if the second argument is zero.


Examples
"""""""""

.. code-block:: llvm

      %res = call i4 @llvm.sdiv.fix.i4(i4 6, i4 2, i32 0)  ; %res = 3 (6 / 2 = 3)
      %res = call i4 @llvm.sdiv.fix.i4(i4 6, i4 4, i32 1)  ; %res = 3 (3 / 2 = 1.5)
      %res = call i4 @llvm.sdiv.fix.i4(i4 3, i4 -2, i32 1) ; %res = -3 (1.5 / -1 = -1.5)

      ; The result in the following could be rounded up to 1 or down to 0.5
      %res = call i4 @llvm.sdiv.fix.i4(i4 3, i4 4, i32 1)  ; %res = 2 (or 1) (1.5 / 2 = 0.75)


'``llvm.udiv.fix.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax
"""""""

This is an overloaded intrinsic. You can use ``llvm.udiv.fix``
on any integer bit width or vectors of integers.

::

      declare i16 @llvm.udiv.fix.i16(i16 %a, i16 %b, i32 %scale)
      declare i32 @llvm.udiv.fix.i32(i32 %a, i32 %b, i32 %scale)
      declare i64 @llvm.udiv.fix.i64(i64 %a, i64 %b, i32 %scale)
      declare <4 x i32> @llvm.udiv.fix.v4i32(<4 x i32> %a, <4 x i32> %b, i32 %scale)

Overview
"""""""""

The '``llvm.udiv.fix``' family of intrinsic functions perform unsigned
fixed point division on 2 arguments of the same scale.

Arguments
""""""""""

The arguments (%a and %b) and the result may be of integer types of any bit
width, but they must have the same bit width. The arguments may also work with
int vectors of the same length and int size. ``%a`` and ``%b`` are the two
values that will undergo unsigned fixed point division. The argument
``%scale`` represents the scale of both operands, and must be a constant
integer.

Semantics:
""""""""""

This operation performs fixed point division on the 2 arguments of a
specified scale. The result will also be returned in the same scale specified
in the third argument.

If the result value cannot be precisely represented in the given scale, the
value is rounded up or down to the closest representable value. The rounding
direction is unspecified.

It is undefined behavior if the result value does not fit within the range of
the fixed point type, or if the second argument is zero.


Examples
"""""""""

.. code-block:: llvm

      %res = call i4 @llvm.udiv.fix.i4(i4 6, i4 2, i32 0)  ; %res = 3 (6 / 2 = 3)
      %res = call i4 @llvm.udiv.fix.i4(i4 6, i4 4, i32 1)  ; %res = 3 (3 / 2 = 1.5)
      %res = call i4 @llvm.udiv.fix.i4(i4 1, i4 -8, i32 4) ; %res = 2 (0.0625 / 0.5 = 0.125)

      ; The result in the following could be rounded up to 1 or down to 0.5
      %res = call i4 @llvm.udiv.fix.i4(i4 3, i4 4, i32 1)  ; %res = 2 (or 1) (1.5 / 2 = 0.75)


'``llvm.sdiv.fix.sat.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax
"""""""

This is an overloaded intrinsic. You can use ``llvm.sdiv.fix.sat``
on any integer bit width or vectors of integers.

::

      declare i16 @llvm.sdiv.fix.sat.i16(i16 %a, i16 %b, i32 %scale)
      declare i32 @llvm.sdiv.fix.sat.i32(i32 %a, i32 %b, i32 %scale)
      declare i64 @llvm.sdiv.fix.sat.i64(i64 %a, i64 %b, i32 %scale)
      declare <4 x i32> @llvm.sdiv.fix.sat.v4i32(<4 x i32> %a, <4 x i32> %b, i32 %scale)

Overview
"""""""""

The '``llvm.sdiv.fix.sat``' family of intrinsic functions perform signed
fixed point saturating division on 2 arguments of the same scale.

Arguments
""""""""""

The arguments (%a and %b) and the result may be of integer types of any bit
width, but they must have the same bit width. ``%a`` and ``%b`` are the two
values that will undergo signed fixed point division. The argument
``%scale`` represents the scale of both operands, and must be a constant
integer.

Semantics:
""""""""""

This operation performs fixed point division on the 2 arguments of a
specified scale. The result will also be returned in the same scale specified
in the third argument.

If the result value cannot be precisely represented in the given scale, the
value is rounded up or down to the closest representable value. The rounding
direction is unspecified.

The maximum value this operation can clamp to is the largest signed value
representable by the bit width of the first 2 arguments. The minimum value is the
smallest signed value representable by this bit width.

It is undefined behavior if the second argument is zero.


Examples
"""""""""

.. code-block:: llvm

      %res = call i4 @llvm.sdiv.fix.sat.i4(i4 6, i4 2, i32 0)  ; %res = 3 (6 / 2 = 3)
      %res = call i4 @llvm.sdiv.fix.sat.i4(i4 6, i4 4, i32 1)  ; %res = 3 (3 / 2 = 1.5)
      %res = call i4 @llvm.sdiv.fix.sat.i4(i4 3, i4 -2, i32 1) ; %res = -3 (1.5 / -1 = -1.5)

      ; The result in the following could be rounded up to 1 or down to 0.5
      %res = call i4 @llvm.sdiv.fix.sat.i4(i4 3, i4 4, i32 1)  ; %res = 2 (or 1) (1.5 / 2 = 0.75)

      ; Saturation
      %res = call i4 @llvm.sdiv.fix.sat.i4(i4 -8, i4 -1, i32 0)  ; %res = 7 (-8 / -1 = 8 => 7)
      %res = call i4 @llvm.sdiv.fix.sat.i4(i4 4, i4 2, i32 2)  ; %res = 7 (1 / 0.5 = 2 => 1.75)
      %res = call i4 @llvm.sdiv.fix.sat.i4(i4 -4, i4 1, i32 2)  ; %res = -8 (-1 / 0.25 = -4 => -2)


'``llvm.udiv.fix.sat.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax
"""""""

This is an overloaded intrinsic. You can use ``llvm.udiv.fix.sat``
on any integer bit width or vectors of integers.

::

      declare i16 @llvm.udiv.fix.sat.i16(i16 %a, i16 %b, i32 %scale)
      declare i32 @llvm.udiv.fix.sat.i32(i32 %a, i32 %b, i32 %scale)
      declare i64 @llvm.udiv.fix.sat.i64(i64 %a, i64 %b, i32 %scale)
      declare <4 x i32> @llvm.udiv.fix.sat.v4i32(<4 x i32> %a, <4 x i32> %b, i32 %scale)

Overview
"""""""""

The '``llvm.udiv.fix.sat``' family of intrinsic functions perform unsigned
fixed point saturating division on 2 arguments of the same scale.

Arguments
""""""""""

The arguments (%a and %b) and the result may be of integer types of any bit
width, but they must have the same bit width. ``%a`` and ``%b`` are the two
values that will undergo unsigned fixed point division. The argument
``%scale`` represents the scale of both operands, and must be a constant
integer.

Semantics:
""""""""""

This operation performs fixed point division on the 2 arguments of a
specified scale. The result will also be returned in the same scale specified
in the third argument.

If the result value cannot be precisely represented in the given scale, the
value is rounded up or down to the closest representable value. The rounding
direction is unspecified.

The maximum value this operation can clamp to is the largest unsigned value
representable by the bit width of the first 2 arguments. The minimum value is the
smallest unsigned value representable by this bit width (zero).

It is undefined behavior if the second argument is zero.

Examples
"""""""""

.. code-block:: llvm

      %res = call i4 @llvm.udiv.fix.sat.i4(i4 6, i4 2, i32 0)  ; %res = 3 (6 / 2 = 3)
      %res = call i4 @llvm.udiv.fix.sat.i4(i4 6, i4 4, i32 1)  ; %res = 3 (3 / 2 = 1.5)

      ; The result in the following could be rounded down to 0.5 or up to 1
      %res = call i4 @llvm.udiv.fix.sat.i4(i4 3, i4 4, i32 1)  ; %res = 1 (or 2) (1.5 / 2 = 0.75)

      ; Saturation
      %res = call i4 @llvm.udiv.fix.sat.i4(i4 8, i4 2, i32 2)  ; %res = 15 (2 / 0.5 = 4 => 3.75)


Specialized Arithmetic Intrinsics
---------------------------------

.. _i_intr_llvm_canonicalize:

'``llvm.canonicalize.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare float @llvm.canonicalize.f32(float %a)
      declare double @llvm.canonicalize.f64(double %b)

Overview:
"""""""""

The '``llvm.canonicalize.*``' intrinsic returns the platform specific canonical
encoding of a floating-point number. This canonicalization is useful for
implementing certain numeric primitives such as frexp. The canonical encoding is
defined by IEEE-754-2008 to be:

::

      2.1.8 canonical encoding: The preferred encoding of a floating-point
      representation in a format. Applied to declets, significands of finite
      numbers, infinities, and NaNs, especially in decimal formats.

This operation can also be considered equivalent to the IEEE-754-2008
conversion of a floating-point value to the same format. NaNs are handled
according to section 6.2.

Examples of non-canonical encodings:

- x87 pseudo denormals, pseudo NaNs, pseudo Infinity, Unnormals. These are
  converted to a canonical representation per hardware-specific protocol.
- Many normal decimal floating-point numbers have non-canonical alternative
  encodings.
- Some machines, like GPUs or ARMv7 NEON, do not support subnormal values.
  These are treated as non-canonical encodings of zero and will be flushed to
  a zero of the same sign by this operation.

Note that per IEEE-754-2008 6.2, systems that support signaling NaNs with
default exception handling must signal an invalid exception, and produce a
quiet NaN result.

This function should always be implementable as multiplication by 1.0, provided
that the compiler does not constant fold the operation. Likewise, division by
1.0 and ``llvm.minnum(x, x)`` are possible implementations. Addition with
-0.0 is also sufficient provided that the rounding mode is not -Infinity.

``@llvm.canonicalize`` must preserve the equality relation. That is:

- ``(@llvm.canonicalize(x) == x)`` is equivalent to ``(x == x)``
- ``(@llvm.canonicalize(x) == @llvm.canonicalize(y))`` is equivalent
  to ``(x == y)``

Additionally, the sign of zero must be conserved:
``@llvm.canonicalize(-0.0) = -0.0`` and ``@llvm.canonicalize(+0.0) = +0.0``

The payload bits of a NaN must be conserved, with two exceptions.
First, environments which use only a single canonical representation of NaN
must perform said canonicalization. Second, SNaNs must be quieted per the
usual methods.

The canonicalization operation may be optimized away if:

- The input is known to be canonical. For example, it was produced by a
  floating-point operation that is required by the standard to be canonical.
- The result is consumed only by (or fused with) other floating-point
  operations. That is, the bits of the floating-point value are not examined.

.. _int_fmuladd:

'``llvm.fmuladd.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare float @llvm.fmuladd.f32(float %a, float %b, float %c)
      declare double @llvm.fmuladd.f64(double %a, double %b, double %c)

Overview:
"""""""""

The '``llvm.fmuladd.*``' intrinsic functions represent multiply-add
expressions that can be fused if the code generator determines that (a) the
target instruction set has support for a fused operation, and (b) that the
fused operation is more efficient than the equivalent, separate pair of mul
and add instructions.

Arguments:
""""""""""

The '``llvm.fmuladd.*``' intrinsics each take three arguments: two
multiplicands, a and b, and an addend c.

Semantics:
""""""""""

The expression:

::

      %0 = call float @llvm.fmuladd.f32(%a, %b, %c)

is equivalent to the expression a \* b + c, except that it is unspecified
whether rounding will be performed between the multiplication and addition
steps. Fusion is not guaranteed, even if the target platform supports it.
If a fused multiply-add is required, the corresponding
:ref:`llvm.fma <int_fma>` intrinsic function should be used instead.
This never sets errno, just as '``llvm.fma.*``'.

Examples:
"""""""""

.. code-block:: llvm

      %r2 = call float @llvm.fmuladd.f32(float %a, float %b, float %c) ; yields float:r2 = (a * b) + c


Hardware-Loop Intrinsics
------------------------

LLVM support several intrinsics to mark a loop as a hardware-loop. They are
hints to the backend which are required to lower these intrinsics further to target
specific instructions, or revert the hardware-loop to a normal loop if target
specific restriction are not met and a hardware-loop can't be generated.

These intrinsics may be modified in the future and are not intended to be used
outside the backend. Thus, front-end and mid-level optimizations should not be
generating these intrinsics.


'``llvm.set.loop.iterations.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic.

::

      declare void @llvm.set.loop.iterations.i32(i32)
      declare void @llvm.set.loop.iterations.i64(i64)

Overview:
"""""""""

The '``llvm.set.loop.iterations.*``' intrinsics are used to specify the
hardware-loop trip count. They are placed in the loop preheader basic block and
are marked as ``IntrNoDuplicate`` to avoid optimizers duplicating these
instructions.

Arguments:
""""""""""

The integer operand is the loop trip count of the hardware-loop, and thus
not e.g. the loop back-edge taken count.

Semantics:
""""""""""

The '``llvm.set.loop.iterations.*``' intrinsics do not perform any arithmetic
on their operand. It's a hint to the backend that can use this to set up the
hardware-loop count with a target specific instruction, usually a move of this
value to a special register or a hardware-loop instruction.


'``llvm.start.loop.iterations.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic.

::

      declare i32 @llvm.start.loop.iterations.i32(i32)
      declare i64 @llvm.start.loop.iterations.i64(i64)

Overview:
"""""""""

The '``llvm.start.loop.iterations.*``' intrinsics are similar to the
'``llvm.set.loop.iterations.*``' intrinsics, used to specify the
hardware-loop trip count but also produce a value identical to the input
that can be used as the input to the loop. They are placed in the loop
preheader basic block and the output is expected to be the input to the
phi for the induction variable of the loop, decremented by the
'``llvm.loop.decrement.reg.*``'.

Arguments:
""""""""""

The integer operand is the loop trip count of the hardware-loop, and thus
not e.g. the loop back-edge taken count.

Semantics:
""""""""""

The '``llvm.start.loop.iterations.*``' intrinsics do not perform any arithmetic
on their operand. It's a hint to the backend that can use this to set up the
hardware-loop count with a target specific instruction, usually a move of this
value to a special register or a hardware-loop instruction.

'``llvm.test.set.loop.iterations.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic.

::

      declare i1 @llvm.test.set.loop.iterations.i32(i32)
      declare i1 @llvm.test.set.loop.iterations.i64(i64)

Overview:
"""""""""

The '``llvm.test.set.loop.iterations.*``' intrinsics are used to specify the
the loop trip count, and also test that the given count is not zero, allowing
it to control entry to a while-loop.  They are placed in the loop preheader's
predecessor basic block, and are marked as ``IntrNoDuplicate`` to avoid
optimizers duplicating these instructions.

Arguments:
""""""""""

The integer operand is the loop trip count of the hardware-loop, and thus
not e.g. the loop back-edge taken count.

Semantics:
""""""""""

The '``llvm.test.set.loop.iterations.*``' intrinsics do not perform any
arithmetic on their operand. It's a hint to the backend that can use this to
set up the hardware-loop count with a target specific instruction, usually a
move of this value to a special register or a hardware-loop instruction.
The result is the conditional value of whether the given count is not zero.


'``llvm.test.start.loop.iterations.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic.

::

      declare {i32, i1} @llvm.test.start.loop.iterations.i32(i32)
      declare {i64, i1} @llvm.test.start.loop.iterations.i64(i64)

Overview:
"""""""""

The '``llvm.test.start.loop.iterations.*``' intrinsics are similar to the
'``llvm.test.set.loop.iterations.*``' and '``llvm.start.loop.iterations.*``'
intrinsics, used to specify the hardware-loop trip count, but also produce a
value identical to the input that can be used as the input to the loop. The
second i1 output controls entry to a while-loop.

Arguments:
""""""""""

The integer operand is the loop trip count of the hardware-loop, and thus
not e.g. the loop back-edge taken count.

Semantics:
""""""""""

The '``llvm.test.start.loop.iterations.*``' intrinsics do not perform any
arithmetic on their operand. It's a hint to the backend that can use this to
set up the hardware-loop count with a target specific instruction, usually a
move of this value to a special register or a hardware-loop instruction.
The result is a pair of the input and a conditional value of whether the
given count is not zero.


'``llvm.loop.decrement.reg.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic.

::

      declare i32 @llvm.loop.decrement.reg.i32(i32, i32)
      declare i64 @llvm.loop.decrement.reg.i64(i64, i64)

Overview:
"""""""""

The '``llvm.loop.decrement.reg.*``' intrinsics are used to lower the loop
iteration counter and return an updated value that will be used in the next
loop test check.

Arguments:
""""""""""

Both arguments must have identical integer types. The first operand is the
loop iteration counter. The second operand is the maximum number of elements
processed in an iteration.

Semantics:
""""""""""

The '``llvm.loop.decrement.reg.*``' intrinsics do an integer ``SUB`` of its
two operands, which is not allowed to wrap. They return the remaining number of
iterations still to be executed, and can be used together with a ``PHI``,
``ICMP`` and ``BR`` to control the number of loop iterations executed. Any
optimizations are allowed to treat it is a ``SUB``, and it is supported by
SCEV, so it's the backends responsibility to handle cases where it may be
optimized. These intrinsics are marked as ``IntrNoDuplicate`` to avoid
optimizers duplicating these instructions.


'``llvm.loop.decrement.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic.

::

      declare i1 @llvm.loop.decrement.i32(i32)
      declare i1 @llvm.loop.decrement.i64(i64)

Overview:
"""""""""

The HardwareLoops pass allows the loop decrement value to be specified with an
option. It defaults to a loop decrement value of 1, but it can be an unsigned
integer value provided by this option.  The '``llvm.loop.decrement.*``'
intrinsics decrement the loop iteration counter with this value, and return a
false predicate if the loop should exit, and true otherwise.
This is emitted if the loop counter is not updated via a ``PHI`` node, which
can also be controlled with an option.

Arguments:
""""""""""

The integer argument is the loop decrement value used to decrement the loop
iteration counter.

Semantics:
""""""""""

The '``llvm.loop.decrement.*``' intrinsics do a ``SUB`` of the loop iteration
counter with the given loop decrement value, and return false if the loop
should exit, this ``SUB`` is not allowed to wrap. The result is a condition
that is used by the conditional branch controlling the loop.


Vector Reduction Intrinsics
---------------------------

Horizontal reductions of vectors can be expressed using the following
intrinsics. Each one takes a vector operand as an input and applies its
respective operation across all elements of the vector, returning a single
scalar result of the same element type.

.. _int_vector_reduce_add:

'``llvm.vector.reduce.add.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i32 @llvm.vector.reduce.add.v4i32(<4 x i32> %a)
      declare i64 @llvm.vector.reduce.add.v2i64(<2 x i64> %a)

Overview:
"""""""""

The '``llvm.vector.reduce.add.*``' intrinsics do an integer ``ADD``
reduction of a vector, returning the result as a scalar. The return type matches
the element-type of the vector input.

Arguments:
""""""""""
The argument to this intrinsic must be a vector of integer values.

.. _int_vector_reduce_fadd:

'``llvm.vector.reduce.fadd.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare float @llvm.vector.reduce.fadd.v4f32(float %start_value, <4 x float> %a)
      declare double @llvm.vector.reduce.fadd.v2f64(double %start_value, <2 x double> %a)

Overview:
"""""""""

The '``llvm.vector.reduce.fadd.*``' intrinsics do a floating-point
``ADD`` reduction of a vector, returning the result as a scalar. The return type
matches the element-type of the vector input.

If the intrinsic call has the 'reassoc' flag set, then the reduction will not
preserve the associativity of an equivalent scalarized counterpart. Otherwise
the reduction will be *sequential*, thus implying that the operation respects
the associativity of a scalarized reduction. That is, the reduction begins with
the start value and performs an fadd operation with consecutively increasing
vector element indices. See the following pseudocode:

::

    float sequential_fadd(start_value, input_vector)
      result = start_value
      for i = 0 to length(input_vector)
        result = result + input_vector[i]
      return result


Arguments:
""""""""""
The first argument to this intrinsic is a scalar start value for the reduction.
The type of the start value matches the element-type of the vector input.
The second argument must be a vector of floating-point values.

To ignore the start value, negative zero (``-0.0``) can be used, as it is
the neutral value of floating point addition.

Examples:
"""""""""

::

      %unord = call reassoc float @llvm.vector.reduce.fadd.v4f32(float -0.0, <4 x float> %input) ; relaxed reduction
      %ord = call float @llvm.vector.reduce.fadd.v4f32(float %start_value, <4 x float> %input) ; sequential reduction


.. _int_vector_reduce_mul:

'``llvm.vector.reduce.mul.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i32 @llvm.vector.reduce.mul.v4i32(<4 x i32> %a)
      declare i64 @llvm.vector.reduce.mul.v2i64(<2 x i64> %a)

Overview:
"""""""""

The '``llvm.vector.reduce.mul.*``' intrinsics do an integer ``MUL``
reduction of a vector, returning the result as a scalar. The return type matches
the element-type of the vector input.

Arguments:
""""""""""
The argument to this intrinsic must be a vector of integer values.

.. _int_vector_reduce_fmul:

'``llvm.vector.reduce.fmul.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare float @llvm.vector.reduce.fmul.v4f32(float %start_value, <4 x float> %a)
      declare double @llvm.vector.reduce.fmul.v2f64(double %start_value, <2 x double> %a)

Overview:
"""""""""

The '``llvm.vector.reduce.fmul.*``' intrinsics do a floating-point
``MUL`` reduction of a vector, returning the result as a scalar. The return type
matches the element-type of the vector input.

If the intrinsic call has the 'reassoc' flag set, then the reduction will not
preserve the associativity of an equivalent scalarized counterpart. Otherwise
the reduction will be *sequential*, thus implying that the operation respects
the associativity of a scalarized reduction. That is, the reduction begins with
the start value and performs an fmul operation with consecutively increasing
vector element indices. See the following pseudocode:

::

    float sequential_fmul(start_value, input_vector)
      result = start_value
      for i = 0 to length(input_vector)
        result = result * input_vector[i]
      return result


Arguments:
""""""""""
The first argument to this intrinsic is a scalar start value for the reduction.
The type of the start value matches the element-type of the vector input.
The second argument must be a vector of floating-point values.

To ignore the start value, one (``1.0``) can be used, as it is the neutral
value of floating point multiplication.

Examples:
"""""""""

::

      %unord = call reassoc float @llvm.vector.reduce.fmul.v4f32(float 1.0, <4 x float> %input) ; relaxed reduction
      %ord = call float @llvm.vector.reduce.fmul.v4f32(float %start_value, <4 x float> %input) ; sequential reduction

.. _int_vector_reduce_and:

'``llvm.vector.reduce.and.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i32 @llvm.vector.reduce.and.v4i32(<4 x i32> %a)

Overview:
"""""""""

The '``llvm.vector.reduce.and.*``' intrinsics do a bitwise ``AND``
reduction of a vector, returning the result as a scalar. The return type matches
the element-type of the vector input.

Arguments:
""""""""""
The argument to this intrinsic must be a vector of integer values.

.. _int_vector_reduce_or:

'``llvm.vector.reduce.or.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i32 @llvm.vector.reduce.or.v4i32(<4 x i32> %a)

Overview:
"""""""""

The '``llvm.vector.reduce.or.*``' intrinsics do a bitwise ``OR`` reduction
of a vector, returning the result as a scalar. The return type matches the
element-type of the vector input.

Arguments:
""""""""""
The argument to this intrinsic must be a vector of integer values.

.. _int_vector_reduce_xor:

'``llvm.vector.reduce.xor.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i32 @llvm.vector.reduce.xor.v4i32(<4 x i32> %a)

Overview:
"""""""""

The '``llvm.vector.reduce.xor.*``' intrinsics do a bitwise ``XOR``
reduction of a vector, returning the result as a scalar. The return type matches
the element-type of the vector input.

Arguments:
""""""""""
The argument to this intrinsic must be a vector of integer values.

.. _int_vector_reduce_smax:

'``llvm.vector.reduce.smax.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i32 @llvm.vector.reduce.smax.v4i32(<4 x i32> %a)

Overview:
"""""""""

The '``llvm.vector.reduce.smax.*``' intrinsics do a signed integer
``MAX`` reduction of a vector, returning the result as a scalar. The return type
matches the element-type of the vector input.

Arguments:
""""""""""
The argument to this intrinsic must be a vector of integer values.

.. _int_vector_reduce_smin:

'``llvm.vector.reduce.smin.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i32 @llvm.vector.reduce.smin.v4i32(<4 x i32> %a)

Overview:
"""""""""

The '``llvm.vector.reduce.smin.*``' intrinsics do a signed integer
``MIN`` reduction of a vector, returning the result as a scalar. The return type
matches the element-type of the vector input.

Arguments:
""""""""""
The argument to this intrinsic must be a vector of integer values.

.. _int_vector_reduce_umax:

'``llvm.vector.reduce.umax.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i32 @llvm.vector.reduce.umax.v4i32(<4 x i32> %a)

Overview:
"""""""""

The '``llvm.vector.reduce.umax.*``' intrinsics do an unsigned
integer ``MAX`` reduction of a vector, returning the result as a scalar. The
return type matches the element-type of the vector input.

Arguments:
""""""""""
The argument to this intrinsic must be a vector of integer values.

.. _int_vector_reduce_umin:

'``llvm.vector.reduce.umin.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i32 @llvm.vector.reduce.umin.v4i32(<4 x i32> %a)

Overview:
"""""""""

The '``llvm.vector.reduce.umin.*``' intrinsics do an unsigned
integer ``MIN`` reduction of a vector, returning the result as a scalar. The
return type matches the element-type of the vector input.

Arguments:
""""""""""
The argument to this intrinsic must be a vector of integer values.

.. _int_vector_reduce_fmax:

'``llvm.vector.reduce.fmax.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare float @llvm.vector.reduce.fmax.v4f32(<4 x float> %a)
      declare double @llvm.vector.reduce.fmax.v2f64(<2 x double> %a)

Overview:
"""""""""

The '``llvm.vector.reduce.fmax.*``' intrinsics do a floating-point
``MAX`` reduction of a vector, returning the result as a scalar. The return type
matches the element-type of the vector input.

This instruction has the same comparison semantics as the '``llvm.maxnum.*``'
intrinsic.  If the intrinsic call has the ``nnan`` fast-math flag, then the
operation can assume that NaNs are not present in the input vector.

Arguments:
""""""""""
The argument to this intrinsic must be a vector of floating-point values.

.. _int_vector_reduce_fmin:

'``llvm.vector.reduce.fmin.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare float @llvm.vector.reduce.fmin.v4f32(<4 x float> %a)
      declare double @llvm.vector.reduce.fmin.v2f64(<2 x double> %a)

Overview:
"""""""""

The '``llvm.vector.reduce.fmin.*``' intrinsics do a floating-point
``MIN`` reduction of a vector, returning the result as a scalar. The return type
matches the element-type of the vector input.

This instruction has the same comparison semantics as the '``llvm.minnum.*``'
intrinsic. If the intrinsic call has the ``nnan`` fast-math flag, then the
operation can assume that NaNs are not present in the input vector.

Arguments:
""""""""""
The argument to this intrinsic must be a vector of floating-point values.

.. _int_vector_reduce_fmaximum:

'``llvm.vector.reduce.fmaximum.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare float @llvm.vector.reduce.fmaximum.v4f32(<4 x float> %a)
      declare double @llvm.vector.reduce.fmaximum.v2f64(<2 x double> %a)

Overview:
"""""""""

The '``llvm.vector.reduce.fmaximum.*``' intrinsics do a floating-point
``MAX`` reduction of a vector, returning the result as a scalar. The return type
matches the element-type of the vector input.

This instruction has the same comparison semantics as the '``llvm.maximum.*``'
intrinsic. That is, this intrinsic propagates NaNs and +0.0 is considered
greater than -0.0. If any element of the vector is a NaN, the result is NaN.

Arguments:
""""""""""
The argument to this intrinsic must be a vector of floating-point values.

.. _int_vector_reduce_fminimum:

'``llvm.vector.reduce.fminimum.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare float @llvm.vector.reduce.fminimum.v4f32(<4 x float> %a)
      declare double @llvm.vector.reduce.fminimum.v2f64(<2 x double> %a)

Overview:
"""""""""

The '``llvm.vector.reduce.fminimum.*``' intrinsics do a floating-point
``MIN`` reduction of a vector, returning the result as a scalar. The return type
matches the element-type of the vector input.

This instruction has the same comparison semantics as the '``llvm.minimum.*``'
intrinsic. That is, this intrinsic propagates NaNs and -0.0 is considered less
than +0.0. If any element of the vector is a NaN, the result is NaN.

Arguments:
""""""""""
The argument to this intrinsic must be a vector of floating-point values.

'``llvm.vector.insert``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      ; Insert fixed type into scalable type
      declare <vscale x 4 x float> @llvm.vector.insert.nxv4f32.v4f32(<vscale x 4 x float> %vec, <4 x float> %subvec, i64 <idx>)
      declare <vscale x 2 x double> @llvm.vector.insert.nxv2f64.v2f64(<vscale x 2 x double> %vec, <2 x double> %subvec, i64 <idx>)

      ; Insert scalable type into scalable type
      declare <vscale x 4 x float> @llvm.vector.insert.nxv4f64.nxv2f64(<vscale x 4 x float> %vec, <vscale x 2 x float> %subvec, i64 <idx>)

      ; Insert fixed type into fixed type
      declare <4 x double> @llvm.vector.insert.v4f64.v2f64(<4 x double> %vec, <2 x double> %subvec, i64 <idx>)

Overview:
"""""""""

The '``llvm.vector.insert.*``' intrinsics insert a vector into another vector
starting from a given index. The return type matches the type of the vector we
insert into. Conceptually, this can be used to build a scalable vector out of
non-scalable vectors, however this intrinsic can also be used on purely fixed
types.

Scalable vectors can only be inserted into other scalable vectors.

Arguments:
""""""""""

The ``vec`` is the vector which ``subvec`` will be inserted into.
The ``subvec`` is the vector that will be inserted.

``idx`` represents the starting element number at which ``subvec`` will be
inserted. ``idx`` must be a constant multiple of ``subvec``'s known minimum
vector length. If ``subvec`` is a scalable vector, ``idx`` is first scaled by
the runtime scaling factor of ``subvec``. The elements of ``vec`` starting at
``idx`` are overwritten with ``subvec``. Elements ``idx`` through (``idx`` +
num_elements(``subvec``) - 1) must be valid ``vec`` indices. If this condition
cannot be determined statically but is false at runtime, then the result vector
is a :ref:`poison value <poisonvalues>`.


'``llvm.vector.extract``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      ; Extract fixed type from scalable type
      declare <4 x float> @llvm.vector.extract.v4f32.nxv4f32(<vscale x 4 x float> %vec, i64 <idx>)
      declare <2 x double> @llvm.vector.extract.v2f64.nxv2f64(<vscale x 2 x double> %vec, i64 <idx>)

      ; Extract scalable type from scalable type
      declare <vscale x 2 x float> @llvm.vector.extract.nxv2f32.nxv4f32(<vscale x 4 x float> %vec, i64 <idx>)

      ; Extract fixed type from fixed type
      declare <2 x double> @llvm.vector.extract.v2f64.v4f64(<4 x double> %vec, i64 <idx>)

Overview:
"""""""""

The '``llvm.vector.extract.*``' intrinsics extract a vector from within another
vector starting from a given index. The return type must be explicitly
specified. Conceptually, this can be used to decompose a scalable vector into
non-scalable parts, however this intrinsic can also be used on purely fixed
types.

Scalable vectors can only be extracted from other scalable vectors.

Arguments:
""""""""""

The ``vec`` is the vector from which we will extract a subvector.

The ``idx`` specifies the starting element number within ``vec`` from which a
subvector is extracted. ``idx`` must be a constant multiple of the known-minimum
vector length of the result type. If the result type is a scalable vector,
``idx`` is first scaled by the result type's runtime scaling factor. Elements
``idx`` through (``idx`` + num_elements(result_type) - 1) must be valid vector
indices. If this condition cannot be determined statically but is false at
runtime, then the result vector is a :ref:`poison value <poisonvalues>`. The
``idx`` parameter must be a vector index constant type (for most targets this
will be an integer pointer type).

'``llvm.vector.reverse``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <2 x i8> @llvm.vector.reverse.v2i8(<2 x i8> %a)
      declare <vscale x 4 x i32> @llvm.vector.reverse.nxv4i32(<vscale x 4 x i32> %a)

Overview:
"""""""""

The '``llvm.vector.reverse.*``' intrinsics reverse a vector.
The intrinsic takes a single vector and returns a vector of matching type but
with the original lane order reversed. These intrinsics work for both fixed
and scalable vectors. While this intrinsic supports all vector types
the recommended way to express this operation for fixed-width vectors is
still to use a shufflevector, as that may allow for more optimization
opportunities.

Arguments:
""""""""""

The argument to this intrinsic must be a vector.

'``llvm.vector.deinterleave2/3/5/7``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare {<2 x double>, <2 x double>} @llvm.vector.deinterleave2.v4f64(<4 x double> %vec1)
      declare {<vscale x 4 x i32>, <vscale x 4 x i32>}  @llvm.vector.deinterleave2.nxv8i32(<vscale x 8 x i32> %vec1)
      declare {<vscale x 2 x i8>, <vscale x 2 x i8>, <vscale x 2 x i8>} @llvm.vector.deinterleave3.nxv6i8(<vscale x 6 x i8> %vec1)
      declare {<2 x i32>, <2 x i32>, <2 x i32>, <2 x i32>, <2 x i32>} @llvm.vector.deinterleave5.v10i32(<10 x i32> %vec1)
      declare {<2 x i32>, <2 x i32>, <2 x i32>, <2 x i32>, <2 x i32>, <2 x i32>, <2 x i32>} @llvm.vector.deinterleave7.v14i32(<14 x i32> %vec1)

Overview:
"""""""""

The '``llvm.vector.deinterleave2/3/5/7``' intrinsics deinterleave adjacent lanes
into 2, 3, 5, and 7 separate vectors, respectively, and return them as the
result.

This intrinsic works for both fixed and scalable vectors. While this intrinsic
supports all vector types the recommended way to express this operation for
factor of 2 on fixed-width vectors is still to use a shufflevector, as that
may allow for more optimization opportunities.

For example:

.. code-block:: text

  {<2 x i64>, <2 x i64>} llvm.vector.deinterleave2.v4i64(<4 x i64> <i64 0, i64 1, i64 2, i64 3>); ==> {<2 x i64> <i64 0, i64 2>, <2 x i64> <i64 1, i64 3>}
  {<2 x i32>, <2 x i32>, <2 x i32>} llvm.vector.deinterleave3.v6i32(<6 x i32> <i32 0, i32 1, i32 2, i32 3, i32 4, i32 5>)
    ; ==> {<2 x i32> <i32 0, i32 3>, <2 x i32> <i32 1, i32 4>, <2 x i32> <i32 2, i32 5>}

Arguments:
""""""""""

The argument is a vector whose type corresponds to the logical concatenation of
the aggregated result types.

'``llvm.vector.interleave2/3/5/7``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <4 x double> @llvm.vector.interleave2.v4f64(<2 x double> %vec1, <2 x double> %vec2)
      declare <vscale x 8 x i32> @llvm.vector.interleave2.nxv8i32(<vscale x 4 x i32> %vec1, <vscale x 4 x i32> %vec2)
      declare <vscale x 6 x i8> @llvm.vector.interleave3.nxv6i8(<vscale x 2 x i8> %vec0, <vscale x 2 x i8> %vec1, <vscale x 2 x i8> %vec2)
      declare <10 x i32> @llvm.vector.interleave5.v10i32(<2 x i32> %vec0, <2 x i32> %vec1, <2 x i32> %vec2, <2 x i32> %vec3, <2 x i32> %vec4)
      declare <14 x i32> @llvm.vector.interleave7.v14i32(<2 x i32> %vec0, <2 x i32> %vec1, <2 x i32> %vec2, <2 x i32> %vec3, <2 x i32> %vec4, <2 x i32> %vec5, <2 x i32> %vec6)

Overview:
"""""""""

The '``llvm.vector.interleave2/3/5/7``' intrinsic constructs a vector
by interleaving all the input vectors.

This intrinsic works for both fixed and scalable vectors. While this intrinsic
supports all vector types the recommended way to express this operation for
factor of 2 on fixed-width vectors is still to use a shufflevector, as that
may allow for more optimization opportunities.

For example:

.. code-block:: text

   <4 x i64> llvm.vector.interleave2.v4i64(<2 x i64> <i64 0, i64 2>, <2 x i64> <i64 1, i64 3>); ==> <4 x i64> <i64 0, i64 1, i64 2, i64 3>
   <6 x i32> llvm.vector.interleave3.v6i32(<2 x i32> <i32 0, i32 3>, <2 x i32> <i32 1, i32 4>, <2 x i32> <i32 2, i32 5>)
    ; ==> <6 x i32> <i32 0, i32 1, i32 2, i32 3, i32 4, i32 5>

Arguments:
""""""""""
All arguments must be vectors of the same type whereby their logical
concatenation matches the result type.

'``llvm.experimental.cttz.elts``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ```llvm.experimental.cttz.elts```
on any vector of integer elements, both fixed width and scalable.

::

      declare i8 @llvm.experimental.cttz.elts.i8.v8i1(<8 x i1> <src>, i1 <is_zero_poison>)

Overview:
"""""""""

The '``llvm.experimental.cttz.elts``' intrinsic counts the number of trailing
zero elements of a vector.

Arguments:
""""""""""

The first argument is the vector to be counted. This argument must be a vector
with integer element type. The return type must also be an integer type which is
wide enough to hold the maximum number of elements of the source vector. The
behavior of this intrinsic is undefined if the return type is not wide enough
for the number of elements in the input vector.

The second argument is a constant flag that indicates whether the intrinsic
returns a valid result if the first argument is all zero. If the first argument
is all zero and the second argument is true, the result is poison.

Semantics:
""""""""""

The '``llvm.experimental.cttz.elts``' intrinsic counts the trailing (least
significant) zero elements in a vector. If ``src == 0`` the result is the
number of elements in the input vector.

'``llvm.vector.splice``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <2 x double> @llvm.vector.splice.v2f64(<2 x double> %vec1, <2 x double> %vec2, i32 %imm)
      declare <vscale x 4 x i32> @llvm.vector.splice.nxv4i32(<vscale x 4 x i32> %vec1, <vscale x 4 x i32> %vec2, i32 %imm)

Overview:
"""""""""

The '``llvm.vector.splice.*``' intrinsics construct a vector by
concatenating elements from the first input vector with elements of the second
input vector, returning a vector of the same type as the input vectors. The
signed immediate, modulo the number of elements in the vector, is the index
into the first vector from which to extract the result value. This means
conceptually that for a positive immediate, a vector is extracted from
``concat(%vec1, %vec2)`` starting at index ``imm``, whereas for a negative
immediate, it extracts ``-imm`` trailing elements from the first vector, and
the remaining elements from ``%vec2``.

These intrinsics work for both fixed and scalable vectors. While this intrinsic
supports all vector types the recommended way to express this operation for
fixed-width vectors is still to use a shufflevector, as that may allow for more
optimization opportunities.

For example:

.. code-block:: text

 llvm.vector.splice(<A,B,C,D>, <E,F,G,H>, 1);  ==> <B, C, D, E> index
 llvm.vector.splice(<A,B,C,D>, <E,F,G,H>, -3); ==> <B, C, D, E> trailing elements


Arguments:
""""""""""

The first two operands are vectors with the same type. The start index is imm
modulo the runtime number of elements in the source vector. For a fixed-width
vector <N x eltty>, imm is a signed integer constant in the range
-N <= imm < N. For a scalable vector <vscale x N x eltty>, imm is a signed
integer constant in the range -X <= imm < X where X=vscale_range_min * N.

'``llvm.stepvector``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This is an overloaded intrinsic. You can use ``llvm.stepvector``
to generate a vector whose lane values comprise the linear sequence
<0, 1, 2, ...>. It is primarily intended for scalable vectors.

::

      declare <vscale x 4 x i32> @llvm.stepvector.nxv4i32()
      declare <vscale x 8 x i16> @llvm.stepvector.nxv8i16()

The '``llvm.stepvector``' intrinsics are used to create vectors
of integers whose elements contain a linear sequence of values starting from 0
with a step of 1. This intrinsic can only be used for vectors with integer
elements that are at least 8 bits in size. If the sequence value exceeds
the allowed limit for the element type then the result for that lane is
a poison value.

These intrinsics work for both fixed and scalable vectors. While this intrinsic
supports all vector types, the recommended way to express this operation for
fixed-width vectors is still to generate a constant vector instead.


Arguments:
""""""""""

None.


'``llvm.experimental.get.vector.length``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare i32 @llvm.experimental.get.vector.length.i32(i32 %cnt, i32 immarg %vf, i1 immarg %scalable)
      declare i32 @llvm.experimental.get.vector.length.i64(i64 %cnt, i32 immarg %vf, i1 immarg %scalable)

Overview:
"""""""""

The '``llvm.experimental.get.vector.length.*``' intrinsics take a number of
elements to process and returns how many of the elements can be processed
with the requested vectorization factor.

Arguments:
""""""""""

The first argument is an unsigned value of any scalar integer type and specifies
the total number of elements to be processed. The second argument is an i32
immediate for the vectorization factor. The third argument indicates if the
vectorization factor should be multiplied by vscale.

Semantics:
""""""""""

Returns a non-negative i32 value (explicit vector length) that is unknown at compile
time and depends on the hardware specification.
If the result value does not fit in the result type, then the result is
a :ref:`poison value <poisonvalues>`.

This intrinsic is intended to be used by loop vectorization with VP intrinsics
in order to get the number of elements to process on each loop iteration. The
result should be used to decrease the count for the next iteration until the
count reaches zero.

Let ``%max_lanes`` be the number of lanes in the type described by ``%vf`` and
``%scalable``, here are the constraints on the returned value:

-  If ``%cnt`` equals to 0, returns 0.
-  The returned value is always less than or equal to ``%max_lanes``.
-  The returned value is always greater than or equal to ``ceil(%cnt / ceil(%cnt / %max_lanes))``,
   if ``%cnt`` is non-zero.
-  The returned values are monotonically non-increasing in each loop iteration. That is,
   the returned value of an iteration is at least as large as that of any later
   iteration.

Note that it has the following implications:

-  For a loop that uses this intrinsic, the number of iterations is equal to
   ``ceil(%C / %max_lanes)`` where ``%C`` is the initial ``%cnt`` value.
-  If ``%cnt`` is non-zero, the return value is non-zero as well.
-  If ``%cnt`` is less than or equal to ``%max_lanes``, the return value is equal to ``%cnt``.

'``llvm.experimental.vector.partial.reduce.add.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <4 x i32> @llvm.experimental.vector.partial.reduce.add.v4i32.v4i32.v8i32(<4 x i32> %a, <8 x i32> %b)
      declare <4 x i32> @llvm.experimental.vector.partial.reduce.add.v4i32.v4i32.v16i32(<4 x i32> %a, <16 x i32> %b)
      declare <vscale x 4 x i32> @llvm.experimental.vector.partial.reduce.add.nxv4i32.nxv4i32.nxv8i32(<vscale x 4 x i32> %a, <vscale x 8 x i32> %b)
      declare <vscale x 4 x i32> @llvm.experimental.vector.partial.reduce.add.nxv4i32.nxv4i32.nxv16i32(<vscale x 4 x i32> %a, <vscale x 16 x i32> %b)

Overview:
"""""""""

The '``llvm.vector.experimental.partial.reduce.add.*``' intrinsics reduce the
concatenation of the two vector arguments down to the number of elements of the
result vector type.

Arguments:
""""""""""

The first argument is an integer vector with the same type as the result.

The second argument is a vector with a length that is a known integer multiple
of the result's type, while maintaining the same element type.

Semantics:
""""""""""

Other than the reduction operator (e.g. add) the way in which the concatenated
arguments is reduced is entirely unspecified. By their nature these intrinsics
are not expected to be useful in isolation but instead implement the first phase
of an overall reduction operation.

The typical use case is loop vectorization where reductions are split into an
in-loop phase, where maintaining an unordered vector result is important for
performance, and an out-of-loop phase to calculate the final scalar result.

By avoiding the introduction of new ordering constraints, these intrinsics
enhance the ability to leverage a target's accumulation instructions.

'``llvm.experimental.vector.histogram.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

These intrinsics are overloaded.

These intrinsics represent histogram-like operations; that is, updating values
in memory that may not be contiguous, and where multiple elements within a
single vector may be updating the same value in memory.

The update operation must be specified as part of the intrinsic name. For a
simple histogram like the following the ``add`` operation would be used.

.. code-block:: c

    void simple_histogram(int *restrict buckets, unsigned *indices, int N, int inc) {
      for (int i = 0; i < N; ++i)
        buckets[indices[i]] += inc;
    }

More update operation types may be added in the future.

::

    declare void @llvm.experimental.vector.histogram.add.v8p0.i32(<8 x ptr> %ptrs, i32 %inc, <8 x i1> %mask)
    declare void @llvm.experimental.vector.histogram.add.nxv2p0.i64(<vscale x 2 x ptr> %ptrs, i64 %inc, <vscale x 2 x i1> %mask)

Arguments:
""""""""""

The first argument is a vector of pointers to the memory locations to be
updated. The second argument is a scalar used to update the value from
memory; it must match the type of value to be updated. The final argument
is a mask value to exclude locations from being modified.

Semantics:
""""""""""

The '``llvm.experimental.vector.histogram.*``' intrinsics are used to perform
updates on potentially overlapping values in memory. The intrinsics represent
the follow sequence of operations:

1. Gather load from the ``ptrs`` operand, with element type matching that of
   the ``inc`` operand.
2. Update of the values loaded from memory. In the case of the ``add``
   update operation, this means:

   1. Perform a cross-vector histogram operation on the ``ptrs`` operand.
   2. Multiply the result by the ``inc`` operand.
   3. Add the result to the values loaded from memory
3. Scatter the result of the update operation to the memory locations from
   the ``ptrs`` operand.

The ``mask`` operand will apply to at least the gather and scatter operations.

'``llvm.experimental.vector.extract.last.active``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This is an overloaded intrinsic.

::

    declare i32 @llvm.experimental.vector.extract.last.active.v4i32(<4 x i32> %data, <4 x i1> %mask, i32 %passthru)
    declare i16 @llvm.experimental.vector.extract.last.active.nxv8i16(<vscale x 8 x i16> %data, <vscale x 8 x i1> %mask, i16 %passthru)

Arguments:
""""""""""

The first argument is the data vector to extract a lane from. The second is a
mask vector controlling the extraction. The third argument is a passthru
value.

The two input vectors must have the same number of elements, and the type of
the passthru value must match that of the elements of the data vector.

Semantics:
""""""""""

The '``llvm.experimental.vector.extract.last.active``' intrinsic will extract an
element from the data vector at the index matching the highest active lane of
the mask vector. If no mask lanes are active then the passthru value is
returned instead.

.. _int_vector_compress:

'``llvm.experimental.vector.compress.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

LLVM provides an intrinsic for compressing data within a vector based on a selection mask.
Semantically, this is similar to :ref:`llvm.masked.compressstore <int_compressstore>` but with weaker assumptions
and without storing the results to memory, i.e., the data remains in the vector.

Syntax:
"""""""
This is an overloaded intrinsic. A number of scalar values of integer, floating point or pointer data type are collected
from an input vector and placed adjacently within the result vector. A mask defines which elements to collect from the vector.
The remaining lanes are filled with values from ``passthru``.

.. code-block:: llvm

      declare <8 x i32> @llvm.experimental.vector.compress.v8i32(<8 x i32> <value>, <8 x i1> <mask>, <8 x i32> <passthru>)
      declare <16 x float> @llvm.experimental.vector.compress.v16f32(<16 x float> <value>, <16 x i1> <mask>, <16 x float> undef)

Overview:
"""""""""

Selects elements from input vector ``value`` according to the ``mask``.
All selected elements are written into adjacent lanes in the result vector,
from lower to higher.
The mask holds an entry for each vector lane, and is used to select elements
to be kept.
If a ``passthru`` vector is given, all remaining lanes are filled with the
corresponding lane's value from ``passthru``.
The main difference to :ref:`llvm.masked.compressstore <int_compressstore>` is
that the we do not need to guard against memory access for unselected lanes.
This allows for branchless code and better optimization for all targets that
do not support or have inefficient
instructions of the explicit semantics of
:ref:`llvm.masked.compressstore <int_compressstore>` but still have some form
of compress operations.
The result vector can be written with a similar effect, as all the selected
values are at the lower positions of the vector, but without requiring
branches to avoid writes where the mask is ``false``.

Arguments:
""""""""""

The first operand is the input vector, from which elements are selected.
The second operand is the mask, a vector of boolean values.
The third operand is the passthru vector, from which elements are filled
into remaining lanes.
The mask and the input vector must have the same number of vector elements.
The input and passthru vectors must have the same type.

Semantics:
""""""""""

The ``llvm.experimental.vector.compress`` intrinsic compresses data within a vector.
It collects elements from possibly non-adjacent lanes of a vector and places
them contiguously in the result vector based on a selection mask, filling the
remaining lanes with values from ``passthru``.
This intrinsic performs the logic of the following C++ example.
All values in ``out`` after the last selected one are undefined if
``passthru`` is undefined.
If all entries in the ``mask`` are 0, the ``out`` vector is ``passthru``.
If any element of the mask is poison, all elements of the result are poison.
Otherwise, if any element of the mask is undef, all elements of the result are undef.
If ``passthru`` is undefined, the number of valid lanes is equal to the number
of ``true`` entries in the mask, i.e., all lanes >= number-of-selected-values
are undefined.

.. code-block:: cpp

    // Consecutively place selected values in a vector.
    using VecT __attribute__((vector_size(N))) = int;
    VecT compress(VecT vec, VecT mask, VecT passthru) {
      VecT out;
      int idx = 0;
      for (int i = 0; i < N / sizeof(int); ++i) {
        out[idx] = vec[i];
        idx += static_cast<bool>(mask[i]);
      }
      for (; idx < N / sizeof(int); ++idx) {
        out[idx] = passthru[idx];
      }
      return out;
    }


'``llvm.experimental.vector.match.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic.

::

    declare <<n> x i1> @llvm.experimental.vector.match(<<n> x <ty>> %op1, <<m> x <ty>> %op2, <<n> x i1> %mask)
    declare <vscale x <n> x i1> @llvm.experimental.vector.match(<vscale x <n> x <ty>> %op1, <<m> x <ty>> %op2, <vscale x <n> x i1> %mask)

Overview:
"""""""""

Find active elements of the first argument matching any elements of the second.

Arguments:
""""""""""

The first argument is the search vector, the second argument the vector of
elements we are searching for (i.e. for which we consider a match successful),
and the third argument is a mask that controls which elements of the first
argument are active. The first two arguments must be vectors of matching
integer element types. The first and third arguments and the result type must
have matching element counts (fixed or scalable). The second argument must be a
fixed vector, but its length may be different from the remaining arguments.

Semantics:
""""""""""

The '``llvm.experimental.vector.match``' intrinsic compares each active element
in the first argument against the elements of the second argument, placing
``1`` in the corresponding element of the output vector if any equality
comparison is successful, and ``0`` otherwise. Inactive elements in the mask
are set to ``0`` in the output.

Matrix Intrinsics
-----------------

Operations on matrixes requiring shape information (like number of rows/columns
or the memory layout) can be expressed using the matrix intrinsics. These
intrinsics require matrix dimensions to be passed as immediate arguments, and
matrixes are passed and returned as vectors. This means that for a ``R`` x
``C`` matrix, element ``i`` of column ``j`` is at index ``j * R + i`` in the
corresponding vector, with indices starting at 0. Currently column-major layout
is assumed.  The intrinsics support both integer and floating point matrixes.


'``llvm.matrix.transpose.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare vectorty @llvm.matrix.transpose.*(vectorty %In, i32 <Rows>, i32 <Cols>)

Overview:
"""""""""

The '``llvm.matrix.transpose.*``' intrinsics treat ``%In`` as a ``<Rows> x
<Cols>`` matrix and return the transposed matrix in the result vector.

Arguments:
""""""""""

The first argument ``%In`` is a vector that corresponds to a ``<Rows> x
<Cols>`` matrix. Thus, arguments ``<Rows>`` and ``<Cols>`` correspond to the
number of rows and columns, respectively, and must be positive, constant
integers. The returned vector must have ``<Rows> * <Cols>`` elements, and have
the same float or integer element type as ``%In``.

'``llvm.matrix.multiply.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare vectorty @llvm.matrix.multiply.*(vectorty %A, vectorty %B, i32 <OuterRows>, i32 <Inner>, i32 <OuterColumns>)

Overview:
"""""""""

The '``llvm.matrix.multiply.*``' intrinsics treat ``%A`` as a ``<OuterRows> x
<Inner>`` matrix, ``%B`` as a ``<Inner> x <OuterColumns>`` matrix, and
multiplies them. The result matrix is returned in the result vector.

Arguments:
""""""""""

The first vector argument ``%A`` corresponds to a matrix with ``<OuterRows> *
<Inner>`` elements, and the second argument ``%B`` to a matrix with
``<Inner> * <OuterColumns>`` elements. Arguments ``<OuterRows>``,
``<Inner>`` and ``<OuterColumns>`` must be positive, constant integers. The
returned vector must have ``<OuterRows> * <OuterColumns>`` elements.
Vectors ``%A``, ``%B``, and the returned vector all have the same float or
integer element type.


'``llvm.matrix.column.major.load.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare vectorty @llvm.matrix.column.major.load.*(
          ptrty %Ptr, i64 %Stride, i1 <IsVolatile>, i32 <Rows>, i32 <Cols>)

Overview:
"""""""""

The '``llvm.matrix.column.major.load.*``' intrinsics load a ``<Rows> x <Cols>``
matrix using a stride of ``%Stride`` to compute the start address of the
different columns.  The offset is computed using ``%Stride``'s bitwidth. This
allows for convenient loading of sub matrixes. If ``<IsVolatile>`` is true, the
intrinsic is considered a :ref:`volatile memory access <volatile>`. The result
matrix is returned in the result vector. If the ``%Ptr`` argument is known to
be aligned to some boundary, this can be specified as an attribute on the
argument.

Arguments:
""""""""""

The first argument ``%Ptr`` is a pointer type to the returned vector type, and
corresponds to the start address to load from. The second argument ``%Stride``
is a positive, constant integer with ``%Stride >= <Rows>``. ``%Stride`` is used
to compute the column memory addresses. I.e., for a column ``C``, its start
memory addresses is calculated with ``%Ptr + C * %Stride``. The third Argument
``<IsVolatile>`` is a boolean value.  The fourth and fifth arguments,
``<Rows>`` and ``<Cols>``, correspond to the number of rows and columns,
respectively, and must be positive, constant integers. The returned vector must
have ``<Rows> * <Cols>`` elements.

The :ref:`align <attr_align>` parameter attribute can be provided for the
``%Ptr`` arguments.


'``llvm.matrix.column.major.store.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.matrix.column.major.store.*(
          vectorty %In, ptrty %Ptr, i64 %Stride, i1 <IsVolatile>, i32 <Rows>, i32 <Cols>)

Overview:
"""""""""

The '``llvm.matrix.column.major.store.*``' intrinsics store the ``<Rows> x
<Cols>`` matrix in ``%In`` to memory using a stride of ``%Stride`` between
columns. The offset is computed using ``%Stride``'s bitwidth. If
``<IsVolatile>`` is true, the intrinsic is considered a
:ref:`volatile memory access <volatile>`.

If the ``%Ptr`` argument is known to be aligned to some boundary, this can be
specified as an attribute on the argument.

Arguments:
""""""""""

The first argument ``%In`` is a vector that corresponds to a ``<Rows> x
<Cols>`` matrix to be stored to memory. The second argument ``%Ptr`` is a
pointer to the vector type of ``%In``, and is the start address of the matrix
in memory. The third argument ``%Stride`` is a positive, constant integer with
``%Stride >= <Rows>``.  ``%Stride`` is used to compute the column memory
addresses. I.e., for a column ``C``, its start memory addresses is calculated
with ``%Ptr + C * %Stride``.  The fourth argument ``<IsVolatile>`` is a boolean
value. The arguments ``<Rows>`` and ``<Cols>`` correspond to the number of rows
and columns, respectively, and must be positive, constant integers.

The :ref:`align <attr_align>` parameter attribute can be provided
for the ``%Ptr`` arguments.


Half Precision Floating-Point Intrinsics
----------------------------------------

For most target platforms, half precision floating-point is a
storage-only format. This means that it is a dense encoding (in memory)
but does not support computation in the format.

This means that code must first load the half-precision floating-point
value as an i16, then convert it to float with
:ref:`llvm.convert.from.fp16 <int_convert_from_fp16>`. Computation can
then be performed on the float value (including extending to double
etc). To store the value back to memory, it is first converted to float
if needed, then converted to i16 with
:ref:`llvm.convert.to.fp16 <int_convert_to_fp16>`, then storing as an
i16 value.

.. _int_convert_to_fp16:

'``llvm.convert.to.fp16``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i16 @llvm.convert.to.fp16.f32(float %a)
      declare i16 @llvm.convert.to.fp16.f64(double %a)

Overview:
"""""""""

The '``llvm.convert.to.fp16``' intrinsic function performs a conversion from a
conventional floating-point type to half precision floating-point format.

Arguments:
""""""""""

The intrinsic function contains single argument - the value to be
converted.

Semantics:
""""""""""

The '``llvm.convert.to.fp16``' intrinsic function performs a conversion from a
conventional floating-point format to half precision floating-point format. The
return value is an ``i16`` which contains the converted number.

Examples:
"""""""""

.. code-block:: llvm

      %res = call i16 @llvm.convert.to.fp16.f32(float %a)
      store i16 %res, i16* @x, align 2

.. _int_convert_from_fp16:

'``llvm.convert.from.fp16``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare float @llvm.convert.from.fp16.f32(i16 %a)
      declare double @llvm.convert.from.fp16.f64(i16 %a)

Overview:
"""""""""

The '``llvm.convert.from.fp16``' intrinsic function performs a
conversion from half precision floating-point format to single precision
floating-point format.

Arguments:
""""""""""

The intrinsic function contains single argument - the value to be
converted.

Semantics:
""""""""""

The '``llvm.convert.from.fp16``' intrinsic function performs a
conversion from half single precision floating-point format to single
precision floating-point format. The input half-float value is
represented by an ``i16`` value.

Examples:
"""""""""

.. code-block:: llvm

      %a = load i16, ptr @x, align 2
      %res = call float @llvm.convert.from.fp16(i16 %a)

Saturating floating-point to integer conversions
------------------------------------------------

The ``fptoui`` and ``fptosi`` instructions return a
:ref:`poison value <poisonvalues>` if the rounded-towards-zero value is not
representable by the result type. These intrinsics provide an alternative
conversion, which will saturate towards the smallest and largest representable
integer values instead.

'``llvm.fptoui.sat.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.fptoui.sat`` on any
floating-point argument type and any integer result type, or vectors thereof.
Not all targets may support all types, however.

::

      declare i32 @llvm.fptoui.sat.i32.f32(float %f)
      declare i19 @llvm.fptoui.sat.i19.f64(double %f)
      declare <4 x i100> @llvm.fptoui.sat.v4i100.v4f128(<4 x fp128> %f)

Overview:
"""""""""

This intrinsic converts the argument into an unsigned integer using saturating
semantics.

Arguments:
""""""""""

The argument may be any floating-point or vector of floating-point type. The
return value may be any integer or vector of integer type. The number of vector
elements in argument and return must be the same.

Semantics:
""""""""""

The conversion to integer is performed subject to the following rules:

- If the argument is any NaN, zero is returned.
- If the argument is smaller than zero (this includes negative infinity),
  zero is returned.
- If the argument is larger than the largest representable unsigned integer of
  the result type (this includes positive infinity), the largest representable
  unsigned integer is returned.
- Otherwise, the result of rounding the argument towards zero is returned.

Example:
""""""""

.. code-block:: text

      %a = call i8 @llvm.fptoui.sat.i8.f32(float 123.875)            ; yields i8: 123
      %b = call i8 @llvm.fptoui.sat.i8.f32(float -5.75)              ; yields i8:   0
      %c = call i8 @llvm.fptoui.sat.i8.f32(float 377.0)              ; yields i8: 255
      %d = call i8 @llvm.fptoui.sat.i8.f32(float 0xFFF8000000000000) ; yields i8:   0

'``llvm.fptosi.sat.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.fptosi.sat`` on any
floating-point argument type and any integer result type, or vectors thereof.
Not all targets may support all types, however.

::

      declare i32 @llvm.fptosi.sat.i32.f32(float %f)
      declare i19 @llvm.fptosi.sat.i19.f64(double %f)
      declare <4 x i100> @llvm.fptosi.sat.v4i100.v4f128(<4 x fp128> %f)

Overview:
"""""""""

This intrinsic converts the argument into a signed integer using saturating
semantics.

Arguments:
""""""""""

The argument may be any floating-point or vector of floating-point type. The
return value may be any integer or vector of integer type. The number of vector
elements in argument and return must be the same.

Semantics:
""""""""""

The conversion to integer is performed subject to the following rules:

- If the argument is any NaN, zero is returned.
- If the argument is smaller than the smallest representable signed integer of
  the result type (this includes negative infinity), the smallest
  representable signed integer is returned.
- If the argument is larger than the largest representable signed integer of
  the result type (this includes positive infinity), the largest representable
  signed integer is returned.
- Otherwise, the result of rounding the argument towards zero is returned.

Example:
""""""""

.. code-block:: text

      %a = call i8 @llvm.fptosi.sat.i8.f32(float 23.875)             ; yields i8:   23
      %b = call i8 @llvm.fptosi.sat.i8.f32(float -130.75)            ; yields i8: -128
      %c = call i8 @llvm.fptosi.sat.i8.f32(float 999.0)              ; yields i8:  127
      %d = call i8 @llvm.fptosi.sat.i8.f32(float 0xFFF8000000000000) ; yields i8:    0

Convergence Intrinsics
----------------------

The LLVM convergence intrinsics for controlling the semantics of ``convergent``
operations, which all start with the ``llvm.experimental.convergence.``
prefix, are described in the :doc:`../ConvergentOperations` document.

.. _dbg_intrinsics:

Debugger Intrinsics
-------------------

The LLVM debugger intrinsics (which all start with ``llvm.dbg.``
prefix), are described in the `LLVM Source Level
Debugging <../SourceLevelDebugging.html#format-common-intrinsics>`_
document.

Exception Handling Intrinsics
-----------------------------

The LLVM exception handling intrinsics (which all start with
``llvm.eh.`` prefix), are described in the `LLVM Exception
Handling <../ExceptionHandling.html#format-common-intrinsics>`_ document.

Pointer Authentication Intrinsics
---------------------------------

The LLVM pointer authentication intrinsics (which all start with
``llvm.ptrauth.`` prefix), are described in the `Pointer Authentication
<../PointerAuth.html#intrinsics>`_ document.

.. _int_trampoline:

Trampoline Intrinsics
---------------------

These intrinsics make it possible to excise one parameter, marked with
the :ref:`nest <nest>` attribute, from a function. The result is a
callable function pointer lacking the nest parameter - the caller does
not need to provide a value for it. Instead, the value to use is stored
in advance in a "trampoline", a block of memory usually allocated on the
stack, which also contains code to splice the nest value into the
argument list. This is used to implement the GCC nested function address
extension.

For example, if the function is ``i32 f(ptr nest %c, i32 %x, i32 %y)``
then the resulting function pointer has signature ``i32 (i32, i32)``.
It can be created as follows:

.. code-block:: llvm

      %tramp = alloca [10 x i8], align 4 ; size and alignment only correct for X86
      call ptr @llvm.init.trampoline(ptr %tramp, ptr @f, ptr %nval)
      %fp = call ptr @llvm.adjust.trampoline(ptr %tramp)

The call ``%val = call i32 %fp(i32 %x, i32 %y)`` is then equivalent to
``%val = call i32 %f(ptr %nval, i32 %x, i32 %y)``.

.. _int_it:

'``llvm.init.trampoline``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.init.trampoline(ptr <tramp>, ptr <func>, ptr <nval>)

Overview:
"""""""""

This fills the memory pointed to by ``tramp`` with executable code,
turning it into a trampoline.

Arguments:
""""""""""

The ``llvm.init.trampoline`` intrinsic takes three arguments, all
pointers. The ``tramp`` argument must point to a sufficiently large and
sufficiently aligned block of memory; this memory is written to by the
intrinsic. Note that the size and the alignment are target-specific -
LLVM currently provides no portable way of determining them, so a
front-end that generates this intrinsic needs to have some
target-specific knowledge. The ``func`` argument must hold a function.

Semantics:
""""""""""

The block of memory pointed to by ``tramp`` is filled with target
dependent code, turning it into a function. Then ``tramp`` needs to be
passed to :ref:`llvm.adjust.trampoline <int_at>` to get a pointer which can
be :ref:`bitcast (to a new function) and called <int_trampoline>`. The new
function's signature is the same as that of ``func`` with any arguments
marked with the ``nest`` attribute removed. At most one such ``nest``
argument is allowed, and it must be of pointer type. Calling the new
function is equivalent to calling ``func`` with the same argument list,
but with ``nval`` used for the missing ``nest`` argument. If, after
calling ``llvm.init.trampoline``, the memory pointed to by ``tramp`` is
modified, then the effect of any later call to the returned function
pointer is undefined.

.. _int_at:

'``llvm.adjust.trampoline``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare ptr @llvm.adjust.trampoline(ptr <tramp>)

Overview:
"""""""""

This performs any required machine-specific adjustment to the address of
a trampoline (passed as ``tramp``).

Arguments:
""""""""""

``tramp`` must point to a block of memory which already has trampoline
code filled in by a previous call to
:ref:`llvm.init.trampoline <int_it>`.

Semantics:
""""""""""

On some architectures the address of the code to be executed needs to be
different than the address where the trampoline is actually stored. This
intrinsic returns the executable address corresponding to ``tramp``
after performing the required machine specific adjustments. The pointer
returned can then be :ref:`bitcast and executed <int_trampoline>`.


.. _int_vp:

Vector Predication Intrinsics
-----------------------------
VP intrinsics are intended for predicated SIMD/vector code.  A typical VP
operation takes a vector mask and an explicit vector length parameter as in:

::

      <W x T> llvm.vp.<opcode>.*(<W x T> %x, <W x T> %y, <W x i1> %mask, i32 %evl)

The vector mask parameter (%mask) always has a vector of `i1` type, for example
`<32 x i1>`.  The explicit vector length parameter always has the type `i32` and
is an unsigned integer value.  The explicit vector length parameter (%evl) is in
the range:

::

      0 <= %evl <= W,  where W is the number of vector elements

Note that for :ref:`scalable vector types <t_vector>` ``W`` is the runtime
length of the vector.

The VP intrinsic has undefined behavior if ``%evl > W``.  The explicit vector
length (%evl) creates a mask, %EVLmask, with all elements ``0 <= i < %evl`` set
to True, and all other lanes ``%evl <= i < W`` to False.  A new mask %M is
calculated with an element-wise AND from %mask and %EVLmask:

::

      M = %mask AND %EVLmask

A vector operation ``<opcode>`` on vectors ``A`` and ``B`` calculates:

::

       A <opcode> B =  {  A[i] <opcode> B[i]   M[i] = True, and
                       {  undef otherwise

Optimization Hint
^^^^^^^^^^^^^^^^^

Some targets, such as AVX512, do not support the %evl parameter in hardware.
The use of an effective %evl is discouraged for those targets.  The function
``TargetTransformInfo::hasActiveVectorLength()`` returns true when the target
has native support for %evl.

.. _int_vp_select:

'``llvm.vp.select.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.select.v16i32 (<16 x i1> <condition>, <16 x i32> <on_true>, <16 x i32> <on_false>, i32 <evl>)
      declare <vscale x 4 x i64>  @llvm.vp.select.nxv4i64 (<vscale x 4 x i1> <condition>, <vscale x 4 x i64> <on_true>, <vscale x 4 x i64> <on_false>, i32 <evl>)

Overview:
"""""""""

The '``llvm.vp.select``' intrinsic is used to choose one value based on a
condition vector, without IR-level branching.

Arguments:
""""""""""

The first argument is a vector of ``i1`` and indicates the condition.  The
second argument is the value that is selected where the condition vector is
true.  The third argument is the value that is selected where the condition
vector is false.  The vectors must be of the same size.  The fourth argument is
the explicit vector length.

#. The optional ``fast-math flags`` marker indicates that the select has one or
   more :ref:`fast-math flags <fastmath>`. These are optimization hints to
   enable otherwise unsafe floating-point optimizations. Fast-math flags are
   only valid for selects that return :ref:`supported floating-point types
   <fastmath_return_types>`.

Semantics:
""""""""""

The intrinsic selects lanes from the second and third argument depending on a
condition vector.

All result lanes at positions greater or equal than ``%evl`` are undefined.
For all lanes below ``%evl`` where the condition vector is true the lane is
taken from the second argument.  Otherwise, the lane is taken from the third
argument.

Example:
""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.select.v4i32(<4 x i1> %cond, <4 x i32> %on_true, <4 x i32> %on_false, i32 %evl)

      ;;; Expansion.
      ;; Any result is legal on lanes at and above %evl.
      %also.r = select <4 x i1> %cond, <4 x i32> %on_true, <4 x i32> %on_false


.. _int_vp_merge:

'``llvm.vp.merge.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.merge.v16i32 (<16 x i1> <condition>, <16 x i32> <on_true>, <16 x i32> <on_false>, i32 <pivot>)
      declare <vscale x 4 x i64>  @llvm.vp.merge.nxv4i64 (<vscale x 4 x i1> <condition>, <vscale x 4 x i64> <on_true>, <vscale x 4 x i64> <on_false>, i32 <pivot>)

Overview:
"""""""""

The '``llvm.vp.merge``' intrinsic is used to choose one value based on a
condition vector and an index argument, without IR-level branching.

Arguments:
""""""""""

The first argument is a vector of ``i1`` and indicates the condition.  The
second argument is the value that is merged where the condition vector is true.
The third argument is the value that is selected where the condition vector is
false or the lane position is greater equal than the pivot. The fourth argument
is the pivot.

#. The optional ``fast-math flags`` marker indicates that the merge has one or
   more :ref:`fast-math flags <fastmath>`. These are optimization hints to
   enable otherwise unsafe floating-point optimizations. Fast-math flags are
   only valid for merges that return :ref:`supported floating-point types
   <fastmath_return_types>`.

Semantics:
""""""""""

The intrinsic selects lanes from the second and third argument depending on a
condition vector and pivot value.

For all lanes where the condition vector is true and the lane position is less
than ``%pivot`` the lane is taken from the second argument.  Otherwise, the lane
is taken from the third argument.

Example:
""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.merge.v4i32(<4 x i1> %cond, <4 x i32> %on_true, <4 x i32> %on_false, i32 %pivot)

      ;;; Expansion.
      ;; Lanes at and above %pivot are taken from %on_false
      %atfirst = insertelement <4 x i32> poison, i32 %pivot, i32 0
      %splat = shufflevector <4 x i32> %atfirst, <4 x i32> poison, <4 x i32> zeroinitializer
      %pivotmask = icmp ult <4 x i32> <i32 0, i32 1, i32 2, i32 3>, <4 x i32> %splat
      %mergemask = and <4 x i1> %cond, <4 x i1> %pivotmask
      %also.r = select <4 x i1> %mergemask, <4 x i32> %on_true, <4 x i32> %on_false



.. _int_vp_add:

'``llvm.vp.add.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.add.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.add.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.add.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated integer addition of two vectors of integers.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.add``' intrinsic performs integer addition (:ref:`add <i_add>`)
of the first and second vector arguments on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.add.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = add <4 x i32> %a, %b
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison

.. _int_vp_sub:

'``llvm.vp.sub.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.sub.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.sub.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.sub.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated integer subtraction of two vectors of integers.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.sub``' intrinsic performs integer subtraction
(:ref:`sub <i_sub>`)  of the first and second vector arguments on each enabled
lane. The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.sub.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = sub <4 x i32> %a, %b
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison



.. _int_vp_mul:

'``llvm.vp.mul.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.mul.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.mul.nxv46i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.mul.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated integer multiplication of two vectors of integers.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""
The '``llvm.vp.mul``' intrinsic performs integer multiplication
(:ref:`mul <i_mul>`) of the first and second vector arguments on each enabled
lane. The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.mul.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = mul <4 x i32> %a, %b
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_sdiv:

'``llvm.vp.sdiv.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.sdiv.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.sdiv.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.sdiv.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated, signed division of two vectors of integers.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.sdiv``' intrinsic performs signed division (:ref:`sdiv <i_sdiv>`)
of the first and second vector arguments on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.sdiv.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = sdiv <4 x i32> %a, %b
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_udiv:

'``llvm.vp.udiv.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.udiv.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.udiv.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.udiv.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated, unsigned division of two vectors of integers.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.udiv``' intrinsic performs unsigned division
(:ref:`udiv <i_udiv>`) of the first and second vector arguments on each enabled
lane. The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.udiv.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = udiv <4 x i32> %a, %b
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison



.. _int_vp_srem:

'``llvm.vp.srem.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.srem.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.srem.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.srem.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated computations of the signed remainder of two integer vectors.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.srem``' intrinsic computes the remainder of the signed division
(:ref:`srem <i_srem>`) of the first and second vector arguments on each enabled
lane.  The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.srem.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = srem <4 x i32> %a, %b
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison



.. _int_vp_urem:

'``llvm.vp.urem.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.urem.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.urem.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.urem.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated computation of the unsigned remainder of two integer vectors.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.urem``' intrinsic computes the remainder of the unsigned division
(:ref:`urem <i_urem>`) of the first and second vector arguments on each enabled
lane.  The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.urem.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = urem <4 x i32> %a, %b
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_ashr:

'``llvm.vp.ashr.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.ashr.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.ashr.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.ashr.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Vector-predicated arithmetic right-shift.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.ashr``' intrinsic computes the arithmetic right shift
(:ref:`ashr <i_ashr>`) of the first argument by the second argument on each
enabled lane. The result on disabled lanes is a
:ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.ashr.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = ashr <4 x i32> %a, %b
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_lshr:


'``llvm.vp.lshr.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.lshr.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.lshr.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.lshr.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Vector-predicated logical right-shift.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.lshr``' intrinsic computes the logical right shift
(:ref:`lshr <i_lshr>`) of the first argument by the second argument on each
enabled lane. The result on disabled lanes is a
:ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.lshr.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = lshr <4 x i32> %a, %b
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_shl:

'``llvm.vp.shl.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.shl.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.shl.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.shl.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Vector-predicated left shift.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.shl``' intrinsic computes the left shift (:ref:`shl <i_shl>`) of
the first argument by the second argument on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.shl.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = shl <4 x i32> %a, %b
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_or:

'``llvm.vp.or.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.or.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.or.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.or.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Vector-predicated or.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.or``' intrinsic performs a bitwise or (:ref:`or <i_or>`) of the
first two arguments on each enabled lane.  The result on disabled lanes is
a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.or.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = or <4 x i32> %a, %b
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_and:

'``llvm.vp.and.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.and.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.and.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.and.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Vector-predicated and.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.and``' intrinsic performs a bitwise and (:ref:`and <i_or>`) of
the first two arguments on each enabled lane.  The result on disabled lanes is
a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.and.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = and <4 x i32> %a, %b
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_xor:

'``llvm.vp.xor.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.xor.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.xor.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.xor.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Vector-predicated, bitwise xor.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.xor``' intrinsic performs a bitwise xor (:ref:`xor <i_xor>`) of
the first two arguments on each enabled lane.
The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.xor.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = xor <4 x i32> %a, %b
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison

.. _int_vp_abs:

'``llvm.vp.abs.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.abs.v16i32 (<16 x i32> <op>, i1 <is_int_min_poison>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.abs.nxv4i32 (<vscale x 4 x i32> <op>, i1 <is_int_min_poison>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.abs.v256i64 (<256 x i64> <op>, i1 <is_int_min_poison>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated abs of a vector of integers.


Arguments:
""""""""""

The first argument and the result have the same vector of integer type. The
second argument must be a constant and is a flag to indicate whether the result
value of the '``llvm.vp.abs``' intrinsic is a :ref:`poison value <poisonvalues>`
if the first argument is statically or dynamically an ``INT_MIN`` value. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.abs``' intrinsic performs abs (:ref:`abs <int_abs>`) of the first argument on each
enabled lane.  The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.abs.v4i32(<4 x i32> %a, i1 false, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.abs.v4i32(<4 x i32> %a, i1 false)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison



.. _int_vp_smax:

'``llvm.vp.smax.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.smax.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.smax.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.smax.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated integer signed maximum of two vectors of integers.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.smax``' intrinsic performs integer signed maximum (:ref:`smax <int_smax>`)
of the first and second vector arguments on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.smax.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.smax.v4i32(<4 x i32> %a, <4 x i32> %b)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_smin:

'``llvm.vp.smin.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.smin.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.smin.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.smin.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated integer signed minimum of two vectors of integers.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.smin``' intrinsic performs integer signed minimum (:ref:`smin <int_smin>`)
of the first and second vector arguments on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.smin.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.smin.v4i32(<4 x i32> %a, <4 x i32> %b)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_umax:

'``llvm.vp.umax.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.umax.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.umax.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.umax.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated integer unsigned maximum of two vectors of integers.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.umax``' intrinsic performs integer unsigned maximum (:ref:`umax <int_umax>`)
of the first and second vector arguments on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.umax.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.umax.v4i32(<4 x i32> %a, <4 x i32> %b)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_umin:

'``llvm.vp.umin.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.umin.v16i32 (<16 x i32> <left_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.umin.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.umin.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated integer unsigned minimum of two vectors of integers.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.umin``' intrinsic performs integer unsigned minimum (:ref:`umin <int_umin>`)
of the first and second vector arguments on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.umin.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.umin.v4i32(<4 x i32> %a, <4 x i32> %b)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_copysign:

'``llvm.vp.copysign.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.copysign.v16f32 (<16 x float> <mag_op>, <16 x float> <sign_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.copysign.nxv4f32 (<vscale x 4 x float> <mag_op>, <vscale x 4 x float> <sign_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.copysign.v256f64 (<256 x double> <mag_op>, <256 x double> <sign_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point copysign of two vectors of floating-point values.


Arguments:
""""""""""

The first two arguments and the result have the same vector of floating-point type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.copysign``' intrinsic performs floating-point copysign (:ref:`copysign <int_copysign>`)
of the first and second vector arguments on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.  The operation is
performed in the default floating-point environment.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.copysign.v4f32(<4 x float> %mag, <4 x float> %sign, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.copysign.v4f32(<4 x float> %mag, <4 x float> %sign)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_minnum:

'``llvm.vp.minnum.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.minnum.v16f32 (<16 x float> <left_op>, <16 x float> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.minnum.nxv4f32 (<vscale x 4 x float> <left_op>, <vscale x 4 x float> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.minnum.v256f64 (<256 x double> <left_op>, <256 x double> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point IEEE-754-2008 minNum of two vectors of floating-point values.


Arguments:
""""""""""

The first two arguments and the result have the same vector of floating-point type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.minnum``' intrinsic performs floating-point minimum (:ref:`minnum <i_minnum>`)
of the first and second vector arguments on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.  The operation is
performed in the default floating-point environment.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.minnum.v4f32(<4 x float> %a, <4 x float> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.minnum.v4f32(<4 x float> %a, <4 x float> %b)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_maxnum:

'``llvm.vp.maxnum.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.maxnum.v16f32 (<16 x float> <left_op>, <16 x float> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.maxnum.nxv4f32 (<vscale x 4 x float> <left_op>, <vscale x 4 x float> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.maxnum.v256f64 (<256 x double> <left_op>, <256 x double> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point IEEE-754-2008 maxNum of two vectors of floating-point values.


Arguments:
""""""""""

The first two arguments and the result have the same vector of floating-point type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.maxnum``' intrinsic performs floating-point maximum (:ref:`maxnum <i_maxnum>`)
of the first and second vector arguments on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.  The operation is
performed in the default floating-point environment.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.maxnum.v4f32(<4 x float> %a, <4 x float> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.maxnum.v4f32(<4 x float> %a, <4 x float> %b, <4 x i1> %mask, i32 %evl)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_minimum:

'``llvm.vp.minimum.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.minimum.v16f32 (<16 x float> <left_op>, <16 x float> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.minimum.nxv4f32 (<vscale x 4 x float> <left_op>, <vscale x 4 x float> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.minimum.v256f64 (<256 x double> <left_op>, <256 x double> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point minimum of two vectors of floating-point values,
propagating NaNs and treating -0.0 as less than +0.0.

Arguments:
""""""""""

The first two arguments and the result have the same vector of floating-point type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.minimum``' intrinsic performs floating-point minimum (:ref:`minimum <i_minimum>`)
of the first and second vector arguments on each enabled lane, the result being
NaN if either argument is a NaN. -0.0 is considered to be less than +0.0 for this
intrinsic. The result on disabled lanes is a :ref:`poison value <poisonvalues>`.
The operation is performed in the default floating-point environment.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.minimum.v4f32(<4 x float> %a, <4 x float> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.minimum.v4f32(<4 x float> %a, <4 x float> %b)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_maximum:

'``llvm.vp.maximum.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.maximum.v16f32 (<16 x float> <left_op>, <16 x float> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.maximum.nxv4f32 (<vscale x 4 x float> <left_op>, <vscale x 4 x float> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.maximum.v256f64 (<256 x double> <left_op>, <256 x double> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point maximum of two vectors of floating-point values,
propagating NaNs and treating -0.0 as less than +0.0.

Arguments:
""""""""""

The first two arguments and the result have the same vector of floating-point type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.maximum``' intrinsic performs floating-point maximum (:ref:`maximum <i_maximum>`)
of the first and second vector arguments on each enabled lane, the result being
NaN if either argument is a NaN. -0.0 is considered to be less than +0.0 for this
intrinsic. The result on disabled lanes is a :ref:`poison value <poisonvalues>`.
The operation is performed in the default floating-point environment.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.maximum.v4f32(<4 x float> %a, <4 x float> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.maximum.v4f32(<4 x float> %a, <4 x float> %b, <4 x i1> %mask, i32 %evl)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_fadd:

'``llvm.vp.fadd.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.fadd.v16f32 (<16 x float> <left_op>, <16 x float> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.fadd.nxv4f32 (<vscale x 4 x float> <left_op>, <vscale x 4 x float> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.fadd.v256f64 (<256 x double> <left_op>, <256 x double> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point addition of two vectors of floating-point values.


Arguments:
""""""""""

The first two arguments and the result have the same vector of floating-point type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.fadd``' intrinsic performs floating-point addition (:ref:`fadd <i_fadd>`)
of the first and second vector arguments on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.  The operation is
performed in the default floating-point environment.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.fadd.v4f32(<4 x float> %a, <4 x float> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = fadd <4 x float> %a, %b
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_fsub:

'``llvm.vp.fsub.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.fsub.v16f32 (<16 x float> <left_op>, <16 x float> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.fsub.nxv4f32 (<vscale x 4 x float> <left_op>, <vscale x 4 x float> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.fsub.v256f64 (<256 x double> <left_op>, <256 x double> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point subtraction of two vectors of floating-point values.


Arguments:
""""""""""

The first two arguments and the result have the same vector of floating-point type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.fsub``' intrinsic performs floating-point subtraction (:ref:`fsub <i_fsub>`)
of the first and second vector arguments on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.  The operation is
performed in the default floating-point environment.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.fsub.v4f32(<4 x float> %a, <4 x float> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = fsub <4 x float> %a, %b
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_fmul:

'``llvm.vp.fmul.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.fmul.v16f32 (<16 x float> <left_op>, <16 x float> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.fmul.nxv4f32 (<vscale x 4 x float> <left_op>, <vscale x 4 x float> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.fmul.v256f64 (<256 x double> <left_op>, <256 x double> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point multiplication of two vectors of floating-point values.


Arguments:
""""""""""

The first two arguments and the result have the same vector of floating-point type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.fmul``' intrinsic performs floating-point multiplication (:ref:`fmul <i_fmul>`)
of the first and second vector arguments on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.  The operation is
performed in the default floating-point environment.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.fmul.v4f32(<4 x float> %a, <4 x float> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = fmul <4 x float> %a, %b
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_fdiv:

'``llvm.vp.fdiv.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.fdiv.v16f32 (<16 x float> <left_op>, <16 x float> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.fdiv.nxv4f32 (<vscale x 4 x float> <left_op>, <vscale x 4 x float> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.fdiv.v256f64 (<256 x double> <left_op>, <256 x double> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point division of two vectors of floating-point values.


Arguments:
""""""""""

The first two arguments and the result have the same vector of floating-point type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.fdiv``' intrinsic performs floating-point division (:ref:`fdiv <i_fdiv>`)
of the first and second vector arguments on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.  The operation is
performed in the default floating-point environment.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.fdiv.v4f32(<4 x float> %a, <4 x float> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = fdiv <4 x float> %a, %b
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_frem:

'``llvm.vp.frem.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.frem.v16f32 (<16 x float> <left_op>, <16 x float> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.frem.nxv4f32 (<vscale x 4 x float> <left_op>, <vscale x 4 x float> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.frem.v256f64 (<256 x double> <left_op>, <256 x double> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point remainder of two vectors of floating-point values.


Arguments:
""""""""""

The first two arguments and the result have the same vector of floating-point type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.frem``' intrinsic performs floating-point remainder (:ref:`frem <i_frem>`)
of the first and second vector arguments on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.  The operation is
performed in the default floating-point environment.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.frem.v4f32(<4 x float> %a, <4 x float> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = frem <4 x float> %a, %b
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_fneg:

'``llvm.vp.fneg.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.fneg.v16f32 (<16 x float> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.fneg.nxv4f32 (<vscale x 4 x float> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.fneg.v256f64 (<256 x double> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point negation of a vector of floating-point values.


Arguments:
""""""""""

The first argument and the result have the same vector of floating-point type.
The second argument is the vector mask and has the same number of elements as the
result vector type. The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.fneg``' intrinsic performs floating-point negation (:ref:`fneg <i_fneg>`)
of the first vector argument on each enabled lane.  The result on disabled lanes
is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.fneg.v4f32(<4 x float> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = fneg <4 x float> %a
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_fabs:

'``llvm.vp.fabs.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.fabs.v16f32 (<16 x float> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.fabs.nxv4f32 (<vscale x 4 x float> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.fabs.v256f64 (<256 x double> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point absolute value of a vector of floating-point values.


Arguments:
""""""""""

The first argument and the result have the same vector of floating-point type.
The second argument is the vector mask and has the same number of elements as the
result vector type. The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.fabs``' intrinsic performs floating-point absolute value
(:ref:`fabs <int_fabs>`) of the first vector argument on each enabled lane.  The
result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.fabs.v4f32(<4 x float> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.fabs.v4f32(<4 x float> %a)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_sqrt:

'``llvm.vp.sqrt.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.sqrt.v16f32 (<16 x float> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.sqrt.nxv4f32 (<vscale x 4 x float> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.sqrt.v256f64 (<256 x double> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point square root of a vector of floating-point values.


Arguments:
""""""""""

The first argument and the result have the same vector of floating-point type.
The second argument is the vector mask and has the same number of elements as the
result vector type. The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.sqrt``' intrinsic performs floating-point square root (:ref:`sqrt <int_sqrt>`) of
the first vector argument on each enabled lane.  The result on disabled lanes is
a :ref:`poison value <poisonvalues>`. The operation is performed in the default
floating-point environment.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.sqrt.v4f32(<4 x float> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.sqrt.v4f32(<4 x float> %a)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_fma:

'``llvm.vp.fma.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.fma.v16f32 (<16 x float> <left_op>, <16 x float> <middle_op>, <16 x float> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.fma.nxv4f32 (<vscale x 4 x float> <left_op>, <vscale x 4 x float> <middle_op>, <vscale x 4 x float> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.fma.v256f64 (<256 x double> <left_op>, <256 x double> <middle_op>, <256 x double> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point fused multiply-add of two vectors of floating-point values.


Arguments:
""""""""""

The first three arguments and the result have the same vector of floating-point type. The
fourth argument is the vector mask and has the same number of elements as the
result vector type. The fifth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.fma``' intrinsic performs floating-point fused multiply-add (:ref:`llvm.fma <int_fma>`)
of the first, second, and third vector argument on each enabled lane.  The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.  The operation is
performed in the default floating-point environment.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.fma.v4f32(<4 x float> %a, <4 x float> %b, <4 x float> %c, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.fma(<4 x float> %a, <4 x float> %b, <4 x float> %c)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_fmuladd:

'``llvm.vp.fmuladd.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.fmuladd.v16f32 (<16 x float> <left_op>, <16 x float> <middle_op>, <16 x float> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.fmuladd.nxv4f32 (<vscale x 4 x float> <left_op>, <vscale x 4 x float> <middle_op>, <vscale x 4 x float> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.fmuladd.v256f64 (<256 x double> <left_op>, <256 x double> <middle_op>, <256 x double> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point multiply-add of two vectors of floating-point values
that can be fused if code generator determines that (a) the target instruction
set has support for a fused operation, and (b) that the fused operation is more
efficient than the equivalent, separate pair of mul and add instructions.

Arguments:
""""""""""

The first three arguments and the result have the same vector of floating-point
type. The fourth argument is the vector mask and has the same number of elements
as the result vector type. The fifth argument is the explicit vector length of
the operation.

Semantics:
""""""""""

The '``llvm.vp.fmuladd``' intrinsic performs floating-point multiply-add (:ref:`llvm.fuladd <int_fmuladd>`)
of the first, second, and third vector argument on each enabled lane.  The result
on disabled lanes is a :ref:`poison value <poisonvalues>`.  The operation is
performed in the default floating-point environment.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.fmuladd.v4f32(<4 x float> %a, <4 x float> %b, <4 x float> %c, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.fmuladd(<4 x float> %a, <4 x float> %b, <4 x float> %c)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_reduce_add:

'``llvm.vp.reduce.add.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare i32 @llvm.vp.reduce.add.v4i32(i32 <start_value>, <4 x i32> <val>, <4 x i1> <mask>, i32 <vector_length>)
      declare i16 @llvm.vp.reduce.add.nxv8i16(i16 <start_value>, <vscale x 8 x i16> <val>, <vscale x 8 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated integer ``ADD`` reduction of a vector and a scalar starting value,
returning the result as a scalar.

Arguments:
""""""""""

The first argument is the start value of the reduction, which must be a scalar
integer type equal to the result type. The second argument is the vector on
which the reduction is performed and must be a vector of integer values whose
element type is the result/start type. The third argument is the vector mask and
is a vector of boolean values with the same number of elements as the vector
argument. The fourth argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.reduce.add``' intrinsic performs the integer ``ADD`` reduction
(:ref:`llvm.vector.reduce.add <int_vector_reduce_add>`) of the vector argument
``val`` on each enabled lane, adding it to the scalar ``start_value``. Disabled
lanes are treated as containing the neutral value ``0`` (i.e. having no effect
on the reduction operation). If the vector length is zero, the result is equal
to ``start_value``.

To ignore the start value, the neutral value can be used.

Examples:
"""""""""

.. code-block:: llvm

      %r = call i32 @llvm.vp.reduce.add.v4i32(i32 %start, <4 x i32> %a, <4 x i1> %mask, i32 %evl)
      ; %r is equivalent to %also.r, where lanes greater than or equal to %evl
      ; are treated as though %mask were false for those lanes.

      %masked.a = select <4 x i1> %mask, <4 x i32> %a, <4 x i32> zeroinitializer
      %reduction = call i32 @llvm.vector.reduce.add.v4i32(<4 x i32> %masked.a)
      %also.r = add i32 %reduction, %start


.. _int_vp_reduce_fadd:

'``llvm.vp.reduce.fadd.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare float @llvm.vp.reduce.fadd.v4f32(float <start_value>, <4 x float> <val>, <4 x i1> <mask>, i32 <vector_length>)
      declare double @llvm.vp.reduce.fadd.nxv8f64(double <start_value>, <vscale x 8 x double> <val>, <vscale x 8 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point ``ADD`` reduction of a vector and a scalar starting
value, returning the result as a scalar.

Arguments:
""""""""""

The first argument is the start value of the reduction, which must be a scalar
floating-point type equal to the result type. The second argument is the vector
on which the reduction is performed and must be a vector of floating-point
values whose element type is the result/start type. The third argument is the
vector mask and is a vector of boolean values with the same number of elements
as the vector argument. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.reduce.fadd``' intrinsic performs the floating-point ``ADD``
reduction (:ref:`llvm.vector.reduce.fadd <int_vector_reduce_fadd>`) of the
vector argument ``val`` on each enabled lane, adding it to the scalar
``start_value``. Disabled lanes are treated as containing the neutral value
``-0.0`` (i.e. having no effect on the reduction operation). If no lanes are
enabled, the resulting value will be equal to ``start_value``.

To ignore the start value, the neutral value can be used.

See the unpredicated version (:ref:`llvm.vector.reduce.fadd
<int_vector_reduce_fadd>`) for more detail on the semantics of the reduction.

Examples:
"""""""""

.. code-block:: llvm

      %r = call float @llvm.vp.reduce.fadd.v4f32(float %start, <4 x float> %a, <4 x i1> %mask, i32 %evl)
      ; %r is equivalent to %also.r, where lanes greater than or equal to %evl
      ; are treated as though %mask were false for those lanes.

      %masked.a = select <4 x i1> %mask, <4 x float> %a, <4 x float> <float -0.0, float -0.0, float -0.0, float -0.0>
      %also.r = call float @llvm.vector.reduce.fadd.v4f32(float %start, <4 x float> %masked.a)


.. _int_vp_reduce_mul:

'``llvm.vp.reduce.mul.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare i32 @llvm.vp.reduce.mul.v4i32(i32 <start_value>, <4 x i32> <val>, <4 x i1> <mask>, i32 <vector_length>)
      declare i16 @llvm.vp.reduce.mul.nxv8i16(i16 <start_value>, <vscale x 8 x i16> <val>, <vscale x 8 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated integer ``MUL`` reduction of a vector and a scalar starting value,
returning the result as a scalar.


Arguments:
""""""""""

The first argument is the start value of the reduction, which must be a scalar
integer type equal to the result type. The second argument is the vector on
which the reduction is performed and must be a vector of integer values whose
element type is the result/start type. The third argument is the vector mask and
is a vector of boolean values with the same number of elements as the vector
argument. The fourth argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.reduce.mul``' intrinsic performs the integer ``MUL`` reduction
(:ref:`llvm.vector.reduce.mul <int_vector_reduce_mul>`) of the vector argument ``val``
on each enabled lane, multiplying it by the scalar ``start_value``. Disabled
lanes are treated as containing the neutral value ``1`` (i.e. having no effect
on the reduction operation). If the vector length is zero, the result is the
start value.

To ignore the start value, the neutral value can be used.

Examples:
"""""""""

.. code-block:: llvm

      %r = call i32 @llvm.vp.reduce.mul.v4i32(i32 %start, <4 x i32> %a, <4 x i1> %mask, i32 %evl)
      ; %r is equivalent to %also.r, where lanes greater than or equal to %evl
      ; are treated as though %mask were false for those lanes.

      %masked.a = select <4 x i1> %mask, <4 x i32> %a, <4 x i32> <i32 1, i32 1, i32 1, i32 1>
      %reduction = call i32 @llvm.vector.reduce.mul.v4i32(<4 x i32> %masked.a)
      %also.r = mul i32 %reduction, %start

.. _int_vp_reduce_fmul:

'``llvm.vp.reduce.fmul.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare float @llvm.vp.reduce.fmul.v4f32(float <start_value>, <4 x float> <val>, <4 x i1> <mask>, i32 <vector_length>)
      declare double @llvm.vp.reduce.fmul.nxv8f64(double <start_value>, <vscale x 8 x double> <val>, <vscale x 8 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point ``MUL`` reduction of a vector and a scalar starting
value, returning the result as a scalar.


Arguments:
""""""""""

The first argument is the start value of the reduction, which must be a scalar
floating-point type equal to the result type. The second argument is the vector
on which the reduction is performed and must be a vector of floating-point
values whose element type is the result/start type. The third argument is the
vector mask and is a vector of boolean values with the same number of elements
as the vector argument. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.reduce.fmul``' intrinsic performs the floating-point ``MUL``
reduction (:ref:`llvm.vector.reduce.fmul <int_vector_reduce_fmul>`) of the
vector argument ``val`` on each enabled lane, multiplying it by the scalar
`start_value``. Disabled lanes are treated as containing the neutral value
``1.0`` (i.e. having no effect on the reduction operation). If no lanes are
enabled, the resulting value will be equal to the starting value.

To ignore the start value, the neutral value can be used.

See the unpredicated version (:ref:`llvm.vector.reduce.fmul
<int_vector_reduce_fmul>`) for more detail on the semantics.

Examples:
"""""""""

.. code-block:: llvm

      %r = call float @llvm.vp.reduce.fmul.v4f32(float %start, <4 x float> %a, <4 x i1> %mask, i32 %evl)
      ; %r is equivalent to %also.r, where lanes greater than or equal to %evl
      ; are treated as though %mask were false for those lanes.

      %masked.a = select <4 x i1> %mask, <4 x float> %a, <4 x float> <float 1.0, float 1.0, float 1.0, float 1.0>
      %also.r = call float @llvm.vector.reduce.fmul.v4f32(float %start, <4 x float> %masked.a)


.. _int_vp_reduce_and:

'``llvm.vp.reduce.and.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare i32 @llvm.vp.reduce.and.v4i32(i32 <start_value>, <4 x i32> <val>, <4 x i1> <mask>, i32 <vector_length>)
      declare i16 @llvm.vp.reduce.and.nxv8i16(i16 <start_value>, <vscale x 8 x i16> <val>, <vscale x 8 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated integer ``AND`` reduction of a vector and a scalar starting value,
returning the result as a scalar.


Arguments:
""""""""""

The first argument is the start value of the reduction, which must be a scalar
integer type equal to the result type. The second argument is the vector on
which the reduction is performed and must be a vector of integer values whose
element type is the result/start type. The third argument is the vector mask and
is a vector of boolean values with the same number of elements as the vector
argument. The fourth argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.reduce.and``' intrinsic performs the integer ``AND`` reduction
(:ref:`llvm.vector.reduce.and <int_vector_reduce_and>`) of the vector argument
``val`` on each enabled lane, performing an '``and``' of that with with the
scalar ``start_value``. Disabled lanes are treated as containing the neutral
value ``UINT_MAX``, or ``-1`` (i.e. having no effect on the reduction
operation). If the vector length is zero, the result is the start value.

To ignore the start value, the neutral value can be used.

Examples:
"""""""""

.. code-block:: llvm

      %r = call i32 @llvm.vp.reduce.and.v4i32(i32 %start, <4 x i32> %a, <4 x i1> %mask, i32 %evl)
      ; %r is equivalent to %also.r, where lanes greater than or equal to %evl
      ; are treated as though %mask were false for those lanes.

      %masked.a = select <4 x i1> %mask, <4 x i32> %a, <4 x i32> <i32 -1, i32 -1, i32 -1, i32 -1>
      %reduction = call i32 @llvm.vector.reduce.and.v4i32(<4 x i32> %masked.a)
      %also.r = and i32 %reduction, %start


.. _int_vp_reduce_or:

'``llvm.vp.reduce.or.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare i32 @llvm.vp.reduce.or.v4i32(i32 <start_value>, <4 x i32> <val>, <4 x i1> <mask>, i32 <vector_length>)
      declare i16 @llvm.vp.reduce.or.nxv8i16(i16 <start_value>, <vscale x 8 x i16> <val>, <vscale x 8 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated integer ``OR`` reduction of a vector and a scalar starting value,
returning the result as a scalar.


Arguments:
""""""""""

The first argument is the start value of the reduction, which must be a scalar
integer type equal to the result type. The second argument is the vector on
which the reduction is performed and must be a vector of integer values whose
element type is the result/start type. The third argument is the vector mask and
is a vector of boolean values with the same number of elements as the vector
argument. The fourth argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.reduce.or``' intrinsic performs the integer ``OR`` reduction
(:ref:`llvm.vector.reduce.or <int_vector_reduce_or>`) of the vector argument
``val`` on each enabled lane, performing an '``or``' of that with the scalar
``start_value``. Disabled lanes are treated as containing the neutral value
``0`` (i.e. having no effect on the reduction operation). If the vector length
is zero, the result is the start value.

To ignore the start value, the neutral value can be used.

Examples:
"""""""""

.. code-block:: llvm

      %r = call i32 @llvm.vp.reduce.or.v4i32(i32 %start, <4 x i32> %a, <4 x i1> %mask, i32 %evl)
      ; %r is equivalent to %also.r, where lanes greater than or equal to %evl
      ; are treated as though %mask were false for those lanes.

      %masked.a = select <4 x i1> %mask, <4 x i32> %a, <4 x i32> <i32 0, i32 0, i32 0, i32 0>
      %reduction = call i32 @llvm.vector.reduce.or.v4i32(<4 x i32> %masked.a)
      %also.r = or i32 %reduction, %start

.. _int_vp_reduce_xor:

'``llvm.vp.reduce.xor.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare i32 @llvm.vp.reduce.xor.v4i32(i32 <start_value>, <4 x i32> <val>, <4 x i1> <mask>, i32 <vector_length>)
      declare i16 @llvm.vp.reduce.xor.nxv8i16(i16 <start_value>, <vscale x 8 x i16> <val>, <vscale x 8 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated integer ``XOR`` reduction of a vector and a scalar starting value,
returning the result as a scalar.


Arguments:
""""""""""

The first argument is the start value of the reduction, which must be a scalar
integer type equal to the result type. The second argument is the vector on
which the reduction is performed and must be a vector of integer values whose
element type is the result/start type. The third argument is the vector mask and
is a vector of boolean values with the same number of elements as the vector
argument. The fourth argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.reduce.xor``' intrinsic performs the integer ``XOR`` reduction
(:ref:`llvm.vector.reduce.xor <int_vector_reduce_xor>`) of the vector argument
``val`` on each enabled lane, performing an '``xor``' of that with the scalar
``start_value``. Disabled lanes are treated as containing the neutral value
``0`` (i.e. having no effect on the reduction operation). If the vector length
is zero, the result is the start value.

To ignore the start value, the neutral value can be used.

Examples:
"""""""""

.. code-block:: llvm

      %r = call i32 @llvm.vp.reduce.xor.v4i32(i32 %start, <4 x i32> %a, <4 x i1> %mask, i32 %evl)
      ; %r is equivalent to %also.r, where lanes greater than or equal to %evl
      ; are treated as though %mask were false for those lanes.

      %masked.a = select <4 x i1> %mask, <4 x i32> %a, <4 x i32> <i32 0, i32 0, i32 0, i32 0>
      %reduction = call i32 @llvm.vector.reduce.xor.v4i32(<4 x i32> %masked.a)
      %also.r = xor i32 %reduction, %start


.. _int_vp_reduce_smax:

'``llvm.vp.reduce.smax.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare i32 @llvm.vp.reduce.smax.v4i32(i32 <start_value>, <4 x i32> <val>, <4 x i1> <mask>, i32 <vector_length>)
      declare i16 @llvm.vp.reduce.smax.nxv8i16(i16 <start_value>, <vscale x 8 x i16> <val>, <vscale x 8 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated signed-integer ``MAX`` reduction of a vector and a scalar starting
value, returning the result as a scalar.


Arguments:
""""""""""

The first argument is the start value of the reduction, which must be a scalar
integer type equal to the result type. The second argument is the vector on
which the reduction is performed and must be a vector of integer values whose
element type is the result/start type. The third argument is the vector mask and
is a vector of boolean values with the same number of elements as the vector
argument. The fourth argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.reduce.smax``' intrinsic performs the signed-integer ``MAX``
reduction (:ref:`llvm.vector.reduce.smax <int_vector_reduce_smax>`) of the
vector argument ``val`` on each enabled lane, and taking the maximum of that and
the scalar ``start_value``. Disabled lanes are treated as containing the
neutral value ``INT_MIN`` (i.e. having no effect on the reduction operation).
If the vector length is zero, the result is the start value.

To ignore the start value, the neutral value can be used.

Examples:
"""""""""

.. code-block:: llvm

      %r = call i8 @llvm.vp.reduce.smax.v4i8(i8 %start, <4 x i8> %a, <4 x i1> %mask, i32 %evl)
      ; %r is equivalent to %also.r, where lanes greater than or equal to %evl
      ; are treated as though %mask were false for those lanes.

      %masked.a = select <4 x i1> %mask, <4 x i8> %a, <4 x i8> <i8 -128, i8 -128, i8 -128, i8 -128>
      %reduction = call i8 @llvm.vector.reduce.smax.v4i8(<4 x i8> %masked.a)
      %also.r = call i8 @llvm.smax.i8(i8 %reduction, i8 %start)


.. _int_vp_reduce_smin:

'``llvm.vp.reduce.smin.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare i32 @llvm.vp.reduce.smin.v4i32(i32 <start_value>, <4 x i32> <val>, <4 x i1> <mask>, i32 <vector_length>)
      declare i16 @llvm.vp.reduce.smin.nxv8i16(i16 <start_value>, <vscale x 8 x i16> <val>, <vscale x 8 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated signed-integer ``MIN`` reduction of a vector and a scalar starting
value, returning the result as a scalar.


Arguments:
""""""""""

The first argument is the start value of the reduction, which must be a scalar
integer type equal to the result type. The second argument is the vector on
which the reduction is performed and must be a vector of integer values whose
element type is the result/start type. The third argument is the vector mask and
is a vector of boolean values with the same number of elements as the vector
argument. The fourth argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.reduce.smin``' intrinsic performs the signed-integer ``MIN``
reduction (:ref:`llvm.vector.reduce.smin <int_vector_reduce_smin>`) of the
vector argument ``val`` on each enabled lane, and taking the minimum of that and
the scalar ``start_value``. Disabled lanes are treated as containing the
neutral value ``INT_MAX`` (i.e. having no effect on the reduction operation).
If the vector length is zero, the result is the start value.

To ignore the start value, the neutral value can be used.

Examples:
"""""""""

.. code-block:: llvm

      %r = call i8 @llvm.vp.reduce.smin.v4i8(i8 %start, <4 x i8> %a, <4 x i1> %mask, i32 %evl)
      ; %r is equivalent to %also.r, where lanes greater than or equal to %evl
      ; are treated as though %mask were false for those lanes.

      %masked.a = select <4 x i1> %mask, <4 x i8> %a, <4 x i8> <i8 127, i8 127, i8 127, i8 127>
      %reduction = call i8 @llvm.vector.reduce.smin.v4i8(<4 x i8> %masked.a)
      %also.r = call i8 @llvm.smin.i8(i8 %reduction, i8 %start)


.. _int_vp_reduce_umax:

'``llvm.vp.reduce.umax.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare i32 @llvm.vp.reduce.umax.v4i32(i32 <start_value>, <4 x i32> <val>, <4 x i1> <mask>, i32 <vector_length>)
      declare i16 @llvm.vp.reduce.umax.nxv8i16(i16 <start_value>, <vscale x 8 x i16> <val>, <vscale x 8 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated unsigned-integer ``MAX`` reduction of a vector and a scalar starting
value, returning the result as a scalar.


Arguments:
""""""""""

The first argument is the start value of the reduction, which must be a scalar
integer type equal to the result type. The second argument is the vector on
which the reduction is performed and must be a vector of integer values whose
element type is the result/start type. The third argument is the vector mask and
is a vector of boolean values with the same number of elements as the vector
argument. The fourth argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.reduce.umax``' intrinsic performs the unsigned-integer ``MAX``
reduction (:ref:`llvm.vector.reduce.umax <int_vector_reduce_umax>`) of the
vector argument ``val`` on each enabled lane, and taking the maximum of that and
the scalar ``start_value``. Disabled lanes are treated as containing the
neutral value ``0`` (i.e. having no effect on the reduction operation). If the
vector length is zero, the result is the start value.

To ignore the start value, the neutral value can be used.

Examples:
"""""""""

.. code-block:: llvm

      %r = call i32 @llvm.vp.reduce.umax.v4i32(i32 %start, <4 x i32> %a, <4 x i1> %mask, i32 %evl)
      ; %r is equivalent to %also.r, where lanes greater than or equal to %evl
      ; are treated as though %mask were false for those lanes.

      %masked.a = select <4 x i1> %mask, <4 x i32> %a, <4 x i32> <i32 0, i32 0, i32 0, i32 0>
      %reduction = call i32 @llvm.vector.reduce.umax.v4i32(<4 x i32> %masked.a)
      %also.r = call i32 @llvm.umax.i32(i32 %reduction, i32 %start)


.. _int_vp_reduce_umin:

'``llvm.vp.reduce.umin.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare i32 @llvm.vp.reduce.umin.v4i32(i32 <start_value>, <4 x i32> <val>, <4 x i1> <mask>, i32 <vector_length>)
      declare i16 @llvm.vp.reduce.umin.nxv8i16(i16 <start_value>, <vscale x 8 x i16> <val>, <vscale x 8 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated unsigned-integer ``MIN`` reduction of a vector and a scalar starting
value, returning the result as a scalar.


Arguments:
""""""""""

The first argument is the start value of the reduction, which must be a scalar
integer type equal to the result type. The second argument is the vector on
which the reduction is performed and must be a vector of integer values whose
element type is the result/start type. The third argument is the vector mask and
is a vector of boolean values with the same number of elements as the vector
argument. The fourth argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.reduce.umin``' intrinsic performs the unsigned-integer ``MIN``
reduction (:ref:`llvm.vector.reduce.umin <int_vector_reduce_umin>`) of the
vector argument ``val`` on each enabled lane, taking the minimum of that and the
scalar ``start_value``. Disabled lanes are treated as containing the neutral
value ``UINT_MAX``, or ``-1`` (i.e. having no effect on the reduction
operation). If the vector length is zero, the result is the start value.

To ignore the start value, the neutral value can be used.

Examples:
"""""""""

.. code-block:: llvm

      %r = call i32 @llvm.vp.reduce.umin.v4i32(i32 %start, <4 x i32> %a, <4 x i1> %mask, i32 %evl)
      ; %r is equivalent to %also.r, where lanes greater than or equal to %evl
      ; are treated as though %mask were false for those lanes.

      %masked.a = select <4 x i1> %mask, <4 x i32> %a, <4 x i32> <i32 -1, i32 -1, i32 -1, i32 -1>
      %reduction = call i32 @llvm.vector.reduce.umin.v4i32(<4 x i32> %masked.a)
      %also.r = call i32 @llvm.umin.i32(i32 %reduction, i32 %start)


.. _int_vp_reduce_fmax:

'``llvm.vp.reduce.fmax.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare float @llvm.vp.reduce.fmax.v4f32(float <start_value>, <4 x float> <val>, <4 x i1> <mask>, i32 <vector_length>)
      declare double @llvm.vp.reduce.fmax.nxv8f64(double <start_value>, <vscale x 8 x double> <val>, <vscale x 8 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point ``MAX`` reduction of a vector and a scalar starting
value, returning the result as a scalar.


Arguments:
""""""""""

The first argument is the start value of the reduction, which must be a scalar
floating-point type equal to the result type. The second argument is the vector
on which the reduction is performed and must be a vector of floating-point
values whose element type is the result/start type. The third argument is the
vector mask and is a vector of boolean values with the same number of elements
as the vector argument. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.reduce.fmax``' intrinsic performs the floating-point ``MAX``
reduction (:ref:`llvm.vector.reduce.fmax <int_vector_reduce_fmax>`) of the
vector argument ``val`` on each enabled lane, taking the maximum of that and the
scalar ``start_value``. Disabled lanes are treated as containing the neutral
value (i.e. having no effect on the reduction operation). If the vector length
is zero, the result is the start value.

The neutral value is dependent on the :ref:`fast-math flags <fastmath>`. If no
flags are set, the neutral value is ``-QNAN``. If ``nnan``  and ``ninf`` are
both set, then the neutral value is the smallest floating-point value for the
result type. If only ``nnan`` is set then the neutral value is ``-Infinity``.

This instruction has the same comparison semantics as the
:ref:`llvm.vector.reduce.fmax <int_vector_reduce_fmax>` intrinsic (and thus the
'``llvm.maxnum.*``' intrinsic).

To ignore the start value, the neutral value can be used.

Examples:
"""""""""

.. code-block:: llvm

      %r = call float @llvm.vp.reduce.fmax.v4f32(float %float, <4 x float> %a, <4 x i1> %mask, i32 %evl)
      ; %r is equivalent to %also.r, where lanes greater than or equal to %evl
      ; are treated as though %mask were false for those lanes.

      %masked.a = select <4 x i1> %mask, <4 x float> %a, <4 x float> <float QNAN, float QNAN, float QNAN, float QNAN>
      %reduction = call float @llvm.vector.reduce.fmax.v4f32(<4 x float> %masked.a)
      %also.r = call float @llvm.maxnum.f32(float %reduction, float %start)


.. _int_vp_reduce_fmin:

'``llvm.vp.reduce.fmin.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare float @llvm.vp.reduce.fmin.v4f32(float <start_value>, <4 x float> <val>, <4 x i1> <mask>, i32 <vector_length>)
      declare double @llvm.vp.reduce.fmin.nxv8f64(double <start_value>, <vscale x 8 x double> <val>, <vscale x 8 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point ``MIN`` reduction of a vector and a scalar starting
value, returning the result as a scalar.


Arguments:
""""""""""

The first argument is the start value of the reduction, which must be a scalar
floating-point type equal to the result type. The second argument is the vector
on which the reduction is performed and must be a vector of floating-point
values whose element type is the result/start type. The third argument is the
vector mask and is a vector of boolean values with the same number of elements
as the vector argument. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.reduce.fmin``' intrinsic performs the floating-point ``MIN``
reduction (:ref:`llvm.vector.reduce.fmin <int_vector_reduce_fmin>`) of the
vector argument ``val`` on each enabled lane, taking the minimum of that and the
scalar ``start_value``. Disabled lanes are treated as containing the neutral
value (i.e. having no effect on the reduction operation). If the vector length
is zero, the result is the start value.

The neutral value is dependent on the :ref:`fast-math flags <fastmath>`. If no
flags are set, the neutral value is ``+QNAN``. If ``nnan``  and ``ninf`` are
both set, then the neutral value is the largest floating-point value for the
result type. If only ``nnan`` is set then the neutral value is ``+Infinity``.

This instruction has the same comparison semantics as the
:ref:`llvm.vector.reduce.fmin <int_vector_reduce_fmin>` intrinsic (and thus the
'``llvm.minnum.*``' intrinsic).

To ignore the start value, the neutral value can be used.

Examples:
"""""""""

.. code-block:: llvm

      %r = call float @llvm.vp.reduce.fmin.v4f32(float %start, <4 x float> %a, <4 x i1> %mask, i32 %evl)
      ; %r is equivalent to %also.r, where lanes greater than or equal to %evl
      ; are treated as though %mask were false for those lanes.

      %masked.a = select <4 x i1> %mask, <4 x float> %a, <4 x float> <float QNAN, float QNAN, float QNAN, float QNAN>
      %reduction = call float @llvm.vector.reduce.fmin.v4f32(<4 x float> %masked.a)
      %also.r = call float @llvm.minnum.f32(float %reduction, float %start)


.. _int_vp_reduce_fmaximum:

'``llvm.vp.reduce.fmaximum.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare float @llvm.vp.reduce.fmaximum.v4f32(float <start_value>, <4 x float> <val>, <4 x i1> <mask>, i32 <vector_length>)
      declare double @llvm.vp.reduce.fmaximum.nxv8f64(double <start_value>, <vscale x 8 x double> <val>, <vscale x 8 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point ``MAX`` reduction of a vector and a scalar starting
value, returning the result as a scalar.


Arguments:
""""""""""

The first argument is the start value of the reduction, which must be a scalar
floating-point type equal to the result type. The second argument is the vector
on which the reduction is performed and must be a vector of floating-point
values whose element type is the result/start type. The third argument is the
vector mask and is a vector of boolean values with the same number of elements
as the vector argument. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.reduce.fmaximum``' intrinsic performs the floating-point ``MAX``
reduction (:ref:`llvm.vector.reduce.fmaximum <int_vector_reduce_fmaximum>`) of
the vector argument ``val`` on each enabled lane, taking the maximum of that and
the scalar ``start_value``. Disabled lanes are treated as containing the
neutral value (i.e. having no effect on the reduction operation). If the vector
length is zero, the result is the start value.

The neutral value is dependent on the :ref:`fast-math flags <fastmath>`. If no
flags are set or only the ``nnan`` is set, the neutral value is ``-Infinity``.
If ``ninf`` is set, then the neutral value is the smallest floating-point value
for the result type.

This instruction has the same comparison semantics as the
:ref:`llvm.vector.reduce.fmaximum <int_vector_reduce_fmaximum>` intrinsic (and
thus the '``llvm.maximum.*``' intrinsic). That is, the result will always be a
number unless any of the elements in the vector or the starting value is
``NaN``. Namely, this intrinsic propagates ``NaN``. Also, -0.0 is considered
less than +0.0.

To ignore the start value, the neutral value can be used.

Examples:
"""""""""

.. code-block:: llvm

      %r = call float @llvm.vp.reduce.fmaximum.v4f32(float %float, <4 x float> %a, <4 x i1> %mask, i32 %evl)
      ; %r is equivalent to %also.r, where lanes greater than or equal to %evl
      ; are treated as though %mask were false for those lanes.

      %masked.a = select <4 x i1> %mask, <4 x float> %a, <4 x float> <float -infinity, float -infinity, float -infinity, float -infinity>
      %reduction = call float @llvm.vector.reduce.fmaximum.v4f32(<4 x float> %masked.a)
      %also.r = call float @llvm.maximum.f32(float %reduction, float %start)


.. _int_vp_reduce_fminimum:

'``llvm.vp.reduce.fminimum.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare float @llvm.vp.reduce.fminimum.v4f32(float <start_value>, <4 x float> <val>, <4 x i1> <mask>, i32 <vector_length>)
      declare double @llvm.vp.reduce.fminimum.nxv8f64(double <start_value>, <vscale x 8 x double> <val>, <vscale x 8 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point ``MIN`` reduction of a vector and a scalar starting
value, returning the result as a scalar.


Arguments:
""""""""""

The first argument is the start value of the reduction, which must be a scalar
floating-point type equal to the result type. The second argument is the vector
on which the reduction is performed and must be a vector of floating-point
values whose element type is the result/start type. The third argument is the
vector mask and is a vector of boolean values with the same number of elements
as the vector argument. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.reduce.fminimum``' intrinsic performs the floating-point ``MIN``
reduction (:ref:`llvm.vector.reduce.fminimum <int_vector_reduce_fminimum>`) of
the vector argument ``val`` on each enabled lane, taking the minimum of that and
the scalar ``start_value``. Disabled lanes are treated as containing the neutral
value (i.e. having no effect on the reduction operation). If the vector length
is zero, the result is the start value.

The neutral value is dependent on the :ref:`fast-math flags <fastmath>`. If no
flags are set or only the ``nnan`` is set, the neutral value is ``+Infinity``.
If ``ninf`` is set, then the neutral value is the largest floating-point value
for the result type.

This instruction has the same comparison semantics as the
:ref:`llvm.vector.reduce.fminimum <int_vector_reduce_fminimum>` intrinsic (and
thus the '``llvm.minimum.*``' intrinsic). That is, the result will always be a
number unless any of the elements in the vector or the starting value is
``NaN``. Namely, this intrinsic propagates ``NaN``. Also, -0.0 is considered
less than +0.0.

To ignore the start value, the neutral value can be used.

Examples:
"""""""""

.. code-block:: llvm

      %r = call float @llvm.vp.reduce.fminimum.v4f32(float %start, <4 x float> %a, <4 x i1> %mask, i32 %evl)
      ; %r is equivalent to %also.r, where lanes greater than or equal to %evl
      ; are treated as though %mask were false for those lanes.

      %masked.a = select <4 x i1> %mask, <4 x float> %a, <4 x float> <float infinity, float infinity, float infinity, float infinity>
      %reduction = call float @llvm.vector.reduce.fminimum.v4f32(<4 x float> %masked.a)
      %also.r = call float @llvm.minimum.f32(float %reduction, float %start)


.. _int_get_active_lane_mask:

'``llvm.get.active.lane.mask.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <4 x i1> @llvm.get.active.lane.mask.v4i1.i32(i32 %base, i32 %n)
      declare <8 x i1> @llvm.get.active.lane.mask.v8i1.i64(i64 %base, i64 %n)
      declare <16 x i1> @llvm.get.active.lane.mask.v16i1.i64(i64 %base, i64 %n)
      declare <vscale x 16 x i1> @llvm.get.active.lane.mask.nxv16i1.i64(i64 %base, i64 %n)


Overview:
"""""""""

Create a mask representing active and inactive vector lanes.


Arguments:
""""""""""

Both arguments have the same scalar integer type. The result is a vector with
the i1 element type.

Semantics:
""""""""""

The '``llvm.get.active.lane.mask.*``' intrinsics are semantically equivalent
to:

::

      %m[i] = icmp ult (%base + i), %n

where ``%m`` is a vector (mask) of active/inactive lanes with its elements
indexed by ``i``,  and ``%base``, ``%n`` are the two arguments to
``llvm.get.active.lane.mask.*``, ``%icmp`` is an integer compare and ``ult``
the unsigned less-than comparison operator.  Overflow cannot occur in
``(%base + i)`` and its comparison against ``%n`` as it is performed in integer
numbers and not in machine numbers.  If ``%n`` is ``0``, then the result is a
poison value. The above is equivalent to:

::

      %m = @llvm.get.active.lane.mask(%base, %n)

This can, for example, be emitted by the loop vectorizer in which case
``%base`` is the first element of the vector induction variable (VIV) and
``%n`` is the loop tripcount. Thus, these intrinsics perform an element-wise
less than comparison of VIV with the loop tripcount, producing a mask of
true/false values representing active/inactive vector lanes, except if the VIV
overflows in which case they return false in the lanes where the VIV overflows.
The arguments are scalar types to accommodate scalable vector types, for which
it is unknown what the type of the step vector needs to be that enumerate its
lanes without overflow.

This mask ``%m`` can e.g. be used in masked load/store instructions. These
intrinsics provide a hint to the backend. I.e., for a vector loop, the
back-edge taken count of the original scalar loop is explicit as the second
argument.


Examples:
"""""""""

.. code-block:: llvm

      %active.lane.mask = call <4 x i1> @llvm.get.active.lane.mask.v4i1.i64(i64 %elem0, i64 429)
      %wide.masked.load = call <4 x i32> @llvm.masked.load.v4i32.p0v4i32(<4 x i32>* %3, i32 4, <4 x i1> %active.lane.mask, <4 x i32> poison)


.. _int_experimental_vp_splice:

'``llvm.experimental.vp.splice``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <2 x double> @llvm.experimental.vp.splice.v2f64(<2 x double> %vec1, <2 x double> %vec2, i32 %imm, <2 x i1> %mask, i32 %evl1, i32 %evl2)
      declare <vscale x 4 x i32> @llvm.experimental.vp.splice.nxv4i32(<vscale x 4 x i32> %vec1, <vscale x 4 x i32> %vec2, i32 %imm, <vscale x 4 x i1> %mask, i32 %evl1, i32 %evl2)

Overview:
"""""""""

The '``llvm.experimental.vp.splice.*``' intrinsic is the vector length
predicated version of the '``llvm.vector.splice.*``' intrinsic.

Arguments:
""""""""""

The result and the first two arguments ``vec1`` and ``vec2`` are vectors with
the same type.  The third argument ``imm`` is an immediate signed integer that
indicates the offset index.  The fourth argument ``mask`` is a vector mask and
has the same number of elements as the result.  The last two arguments ``evl1``
and ``evl2`` are unsigned integers indicating the explicit vector lengths of
``vec1`` and ``vec2`` respectively.  ``imm``, ``evl1`` and ``evl2`` should
respect the following constraints: ``-evl1 <= imm < evl1``, ``0 <= evl1 <= VL``
and ``0 <= evl2 <= VL``, where ``VL`` is the runtime vector factor. If these
constraints are not satisfied the intrinsic has undefined behavior.

Semantics:
""""""""""

Effectively, this intrinsic concatenates ``vec1[0..evl1-1]`` and
``vec2[0..evl2-1]`` and creates the result vector by selecting the elements in a
window of size ``evl2``, starting at index ``imm`` (for a positive immediate) of
the concatenated vector. Elements in the result vector beyond ``evl2`` are
``undef``.  If ``imm`` is negative the starting index is ``evl1 + imm``.  The result
vector of active vector length ``evl2`` contains ``evl1 - imm`` (``-imm`` for
negative ``imm``) elements from indices ``[imm..evl1 - 1]``
(``[evl1 + imm..evl1 -1]`` for negative ``imm``) of ``vec1`` followed by the
first ``evl2 - (evl1 - imm)`` (``evl2 + imm`` for negative ``imm``) elements of
``vec2``. If ``evl1 - imm`` (``-imm``) >= ``evl2``, only the first ``evl2``
elements are considered and the remaining are ``undef``.  The lanes in the result
vector disabled by ``mask`` are ``poison``.

Examples:
"""""""""

.. code-block:: text

 llvm.experimental.vp.splice(<A,B,C,D>, <E,F,G,H>, 1, 2, 3);  ==> <B, E, F, poison> index
 llvm.experimental.vp.splice(<A,B,C,D>, <E,F,G,H>, -2, 3, 2); ==> <B, C, poison, poison> trailing elements


.. _int_experimental_vp_splat:


'``llvm.experimental.vp.splat``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <2 x double> @llvm.experimental.vp.splat.v2f64(double %scalar, <2 x i1> %mask, i32 %evl)
      declare <vscale x 4 x i32> @llvm.experimental.vp.splat.nxv4i32(i32 %scalar, <vscale x 4 x i1> %mask, i32 %evl)

Overview:
"""""""""

The '``llvm.experimental.vp.splat.*``' intrinsic is to create a predicated splat
with specific effective vector length.

Arguments:
""""""""""

The result is a vector and it is a splat of the first scalar argument. The
second argument ``mask`` is a vector mask and has the same number of elements as
the result. The third argument is the explicit vector length of the operation.

Semantics:
""""""""""

This intrinsic splats a vector with ``evl`` elements of a scalar argument.
The lanes in the result vector disabled by ``mask`` are ``poison``. The
elements past ``evl`` are poison.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.splat.v4f32(float %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r
      %e = insertelement <4 x float> poison, float %a, i32 0
      %s = shufflevector <4 x float> %e, <4 x float> poison, <4 x i32> zeroinitializer
      %also.r = select <4 x i1> %mask, <4 x float> %s, <4 x float> poison


.. _int_experimental_vp_reverse:


'``llvm.experimental.vp.reverse``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <2 x double> @llvm.experimental.vp.reverse.v2f64(<2 x double> %vec, <2 x i1> %mask, i32 %evl)
      declare <vscale x 4 x i32> @llvm.experimental.vp.reverse.nxv4i32(<vscale x 4 x i32> %vec, <vscale x 4 x i1> %mask, i32 %evl)

Overview:
"""""""""

The '``llvm.experimental.vp.reverse.*``' intrinsic is the vector length
predicated version of the '``llvm.vector.reverse.*``' intrinsic.

Arguments:
""""""""""

The result and the first argument ``vec`` are vectors with the same type.
The second argument ``mask`` is a vector mask and has the same number of
elements as the result. The third argument is the explicit vector length of
the operation.

Semantics:
""""""""""

This intrinsic reverses the order of the first ``evl`` elements in a vector.
The lanes in the result vector disabled by ``mask`` are ``poison``. The
elements past ``evl`` are poison.

.. _int_vp_load:

'``llvm.vp.load``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

    declare <4 x float> @llvm.vp.load.v4f32.p0(ptr %ptr, <4 x i1> %mask, i32 %evl)
    declare <vscale x 2 x i16> @llvm.vp.load.nxv2i16.p0(ptr %ptr, <vscale x 2 x i1> %mask, i32 %evl)
    declare <8 x float> @llvm.vp.load.v8f32.p1(ptr addrspace(1) %ptr, <8 x i1> %mask, i32 %evl)
    declare <vscale x 1 x i64> @llvm.vp.load.nxv1i64.p6(ptr addrspace(6) %ptr, <vscale x 1 x i1> %mask, i32 %evl)

Overview:
"""""""""

The '``llvm.vp.load.*``' intrinsic is the vector length predicated version of
the :ref:`llvm.masked.load <int_mload>` intrinsic.

Arguments:
""""""""""

The first argument is the base pointer for the load. The second argument is a
vector of boolean values with the same number of elements as the return type.
The third is the explicit vector length of the operation. The return type and
underlying type of the base pointer are the same vector types.

The :ref:`align <attr_align>` parameter attribute can be provided for the first
argument.

Semantics:
""""""""""

The '``llvm.vp.load``' intrinsic reads a vector from memory in the same way as
the '``llvm.masked.load``' intrinsic, where the mask is taken from the
combination of the '``mask``' and '``evl``' arguments in the usual VP way.
Certain '``llvm.masked.load``' arguments do not have corresponding arguments in
'``llvm.vp.load``': the '``passthru``' argument is implicitly ``poison``; the
'``alignment``' argument is taken as the ``align`` parameter attribute, if
provided. The default alignment is taken as the ABI alignment of the return
type as specified by the :ref:`datalayout string<langref_datalayout>`.

Examples:
"""""""""

.. code-block:: text

     %r = call <8 x i8> @llvm.vp.load.v8i8.p0(ptr align 2 %ptr, <8 x i1> %mask, i32 %evl)
     ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

     %also.r = call <8 x i8> @llvm.masked.load.v8i8.p0(ptr %ptr, i32 2, <8 x i1> %mask, <8 x i8> poison)


.. _int_vp_store:

'``llvm.vp.store``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

    declare void @llvm.vp.store.v4f32.p0(<4 x float> %val, ptr %ptr, <4 x i1> %mask, i32 %evl)
    declare void @llvm.vp.store.nxv2i16.p0(<vscale x 2 x i16> %val, ptr %ptr, <vscale x 2 x i1> %mask, i32 %evl)
    declare void @llvm.vp.store.v8f32.p1(<8 x float> %val, ptr addrspace(1) %ptr, <8 x i1> %mask, i32 %evl)
    declare void @llvm.vp.store.nxv1i64.p6(<vscale x 1 x i64> %val, ptr addrspace(6) %ptr, <vscale x 1 x i1> %mask, i32 %evl)

Overview:
"""""""""

The '``llvm.vp.store.*``' intrinsic is the vector length predicated version of
the :ref:`llvm.masked.store <int_mstore>` intrinsic.

Arguments:
""""""""""

The first argument is the vector value to be written to memory. The second
argument is the base pointer for the store. It has the same underlying type as
the value argument. The third argument is a vector of boolean values with the
same number of elements as the return type. The fourth is the explicit vector
length of the operation.

The :ref:`align <attr_align>` parameter attribute can be provided for the
second argument.

Semantics:
""""""""""

The '``llvm.vp.store``' intrinsic reads a vector from memory in the same way as
the '``llvm.masked.store``' intrinsic, where the mask is taken from the
combination of the '``mask``' and '``evl``' arguments in the usual VP way. The
alignment of the operation (corresponding to the '``alignment``' argument of
'``llvm.masked.store``') is specified by the ``align`` parameter attribute (see
above). If it is not provided then the ABI alignment of the type of the
'``value``' argument as specified by the :ref:`datalayout
string<langref_datalayout>` is used instead.

Examples:
"""""""""

.. code-block:: text

     call void @llvm.vp.store.v8i8.p0(<8 x i8> %val, ptr align 4 %ptr, <8 x i1> %mask, i32 %evl)
     ;; For all lanes below %evl, the call above is lane-wise equivalent to the call below.

     call void @llvm.masked.store.v8i8.p0(<8 x i8> %val, ptr %ptr, i32 4, <8 x i1> %mask)


.. _int_experimental_vp_strided_load:

'``llvm.experimental.vp.strided.load``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

    declare <4 x float> @llvm.experimental.vp.strided.load.v4f32.i64(ptr %ptr, i64 %stride, <4 x i1> %mask, i32 %evl)
    declare <vscale x 2 x i16> @llvm.experimental.vp.strided.load.nxv2i16.i64(ptr %ptr, i64 %stride, <vscale x 2 x i1> %mask, i32 %evl)

Overview:
"""""""""

The '``llvm.experimental.vp.strided.load``' intrinsic loads, into a vector, scalar values from
memory locations evenly spaced apart by '``stride``' number of bytes, starting from '``ptr``'.

Arguments:
""""""""""

The first argument is the base pointer for the load. The second argument is the stride
value expressed in bytes. The third argument is a vector of boolean values
with the same number of elements as the return type. The fourth is the explicit
vector length of the operation. The base pointer underlying type matches the type of the scalar
elements of the return argument.

The :ref:`align <attr_align>` parameter attribute can be provided for the first
argument.

Semantics:
""""""""""

The '``llvm.experimental.vp.strided.load``' intrinsic loads, into a vector, multiple scalar
values from memory in the same way as the :ref:`llvm.vp.gather <int_vp_gather>` intrinsic,
where the vector of pointers is in the form:

   ``%ptrs = <%ptr, %ptr + %stride, %ptr + 2 * %stride, ... >``,

with '``ptr``' previously casted to a pointer '``i8``', '``stride``' always interpreted as a signed
integer and all arithmetic occurring in the pointer type.

Examples:
"""""""""

.. code-block:: text

	 %r = call <8 x i64> @llvm.experimental.vp.strided.load.v8i64.i64(i64* %ptr, i64 %stride, <8 x i64> %mask, i32 %evl)
	 ;; The operation can also be expressed like this:

	 %addr = bitcast i64* %ptr to i8*
	 ;; Create a vector of pointers %addrs in the form:
	 ;; %addrs = <%addr, %addr + %stride, %addr + 2 * %stride, ...>
	 %ptrs = bitcast <8 x i8* > %addrs to <8 x i64* >
	 %also.r = call <8 x i64> @llvm.vp.gather.v8i64.v8p0i64(<8 x i64* > %ptrs, <8 x i64> %mask, i32 %evl)


.. _int_experimental_vp_strided_store:

'``llvm.experimental.vp.strided.store``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

    declare void @llvm.experimental.vp.strided.store.v4f32.i64(<4 x float> %val, ptr %ptr, i64 %stride, <4 x i1> %mask, i32 %evl)
    declare void @llvm.experimental.vp.strided.store.nxv2i16.i64(<vscale x 2 x i16> %val, ptr %ptr, i64 %stride, <vscale x 2 x i1> %mask, i32 %evl)

Overview:
"""""""""

The '``@llvm.experimental.vp.strided.store``' intrinsic stores the elements of
'``val``' into memory locations evenly spaced apart by '``stride``' number of
bytes, starting from '``ptr``'.

Arguments:
""""""""""

The first argument is the vector value to be written to memory. The second
argument is the base pointer for the store. Its underlying type matches the
scalar element type of the value argument. The third argument is the stride value
expressed in bytes. The fourth argument is a vector of boolean values with the
same number of elements as the return type. The fifth is the explicit vector
length of the operation.

The :ref:`align <attr_align>` parameter attribute can be provided for the
second argument.

Semantics:
""""""""""

The '``llvm.experimental.vp.strided.store``' intrinsic stores the elements of
'``val``' in the same way as the :ref:`llvm.vp.scatter <int_vp_scatter>` intrinsic,
where the vector of pointers is in the form:

	``%ptrs = <%ptr, %ptr + %stride, %ptr + 2 * %stride, ... >``,

with '``ptr``' previously casted to a pointer '``i8``', '``stride``' always interpreted as a signed
integer and all arithmetic occurring in the pointer type.

Examples:
"""""""""

.. code-block:: text

	 call void @llvm.experimental.vp.strided.store.v8i64.i64(<8 x i64> %val, i64* %ptr, i64 %stride, <8 x i1> %mask, i32 %evl)
	 ;; The operation can also be expressed like this:

	 %addr = bitcast i64* %ptr to i8*
	 ;; Create a vector of pointers %addrs in the form:
	 ;; %addrs = <%addr, %addr + %stride, %addr + 2 * %stride, ...>
	 %ptrs = bitcast <8 x i8* > %addrs to <8 x i64* >
	 call void @llvm.vp.scatter.v8i64.v8p0i64(<8 x i64> %val, <8 x i64*> %ptrs, <8 x i1> %mask, i32 %evl)


.. _int_vp_gather:

'``llvm.vp.gather``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

    declare <4 x double> @llvm.vp.gather.v4f64.v4p0(<4 x ptr> %ptrs, <4 x i1> %mask, i32 %evl)
    declare <vscale x 2 x i8> @llvm.vp.gather.nxv2i8.nxv2p0(<vscale x 2 x ptr> %ptrs, <vscale x 2 x i1> %mask, i32 %evl)
    declare <2 x float> @llvm.vp.gather.v2f32.v2p2(<2 x ptr addrspace(2)> %ptrs, <2 x i1> %mask, i32 %evl)
    declare <vscale x 4 x i32> @llvm.vp.gather.nxv4i32.nxv4p4(<vscale x 4 x ptr addrspace(4)> %ptrs, <vscale x 4 x i1> %mask, i32 %evl)

Overview:
"""""""""

The '``llvm.vp.gather.*``' intrinsic is the vector length predicated version of
the :ref:`llvm.masked.gather <int_mgather>` intrinsic.

Arguments:
""""""""""

The first argument is a vector of pointers which holds all memory addresses to
read. The second argument is a vector of boolean values with the same number of
elements as the return type. The third is the explicit vector length of the
operation. The return type and underlying type of the vector of pointers are
the same vector types.

The :ref:`align <attr_align>` parameter attribute can be provided for the first
argument.

Semantics:
""""""""""

The '``llvm.vp.gather``' intrinsic reads multiple scalar values from memory in
the same way as the '``llvm.masked.gather``' intrinsic, where the mask is taken
from the combination of the '``mask``' and '``evl``' arguments in the usual VP
way. Certain '``llvm.masked.gather``' arguments do not have corresponding
arguments in '``llvm.vp.gather``': the '``passthru``' argument is implicitly
``poison``; the '``alignment``' argument is taken as the ``align`` parameter, if
provided. The default alignment is taken as the ABI alignment of the source
addresses as specified by the :ref:`datalayout string<langref_datalayout>`.

Examples:
"""""""""

.. code-block:: text

     %r = call <8 x i8> @llvm.vp.gather.v8i8.v8p0(<8 x ptr>  align 8 %ptrs, <8 x i1> %mask, i32 %evl)
     ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

     %also.r = call <8 x i8> @llvm.masked.gather.v8i8.v8p0(<8 x ptr> %ptrs, i32 8, <8 x i1> %mask, <8 x i8> poison)


.. _int_vp_scatter:

'``llvm.vp.scatter``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

    declare void @llvm.vp.scatter.v4f64.v4p0(<4 x double> %val, <4 x ptr> %ptrs, <4 x i1> %mask, i32 %evl)
    declare void @llvm.vp.scatter.nxv2i8.nxv2p0(<vscale x 2 x i8> %val, <vscale x 2 x ptr> %ptrs, <vscale x 2 x i1> %mask, i32 %evl)
    declare void @llvm.vp.scatter.v2f32.v2p2(<2 x float> %val, <2 x ptr addrspace(2)> %ptrs, <2 x i1> %mask, i32 %evl)
    declare void @llvm.vp.scatter.nxv4i32.nxv4p4(<vscale x 4 x i32> %val, <vscale x 4 x ptr addrspace(4)> %ptrs, <vscale x 4 x i1> %mask, i32 %evl)

Overview:
"""""""""

The '``llvm.vp.scatter.*``' intrinsic is the vector length predicated version of
the :ref:`llvm.masked.scatter <int_mscatter>` intrinsic.

Arguments:
""""""""""

The first argument is a vector value to be written to memory. The second argument
is a vector of pointers, pointing to where the value elements should be stored.
The third argument is a vector of boolean values with the same number of
elements as the return type. The fourth is the explicit vector length of the
operation.

The :ref:`align <attr_align>` parameter attribute can be provided for the
second argument.

Semantics:
""""""""""

The '``llvm.vp.scatter``' intrinsic writes multiple scalar values to memory in
the same way as the '``llvm.masked.scatter``' intrinsic, where the mask is
taken from the combination of the '``mask``' and '``evl``' arguments in the
usual VP way. The '``alignment``' argument of the '``llvm.masked.scatter``' does
not have a corresponding argument in '``llvm.vp.scatter``': it is instead
provided via the optional ``align`` parameter attribute on the
vector-of-pointers argument. Otherwise it is taken as the ABI alignment of the
destination addresses as specified by the :ref:`datalayout
string<langref_datalayout>`.

Examples:
"""""""""

.. code-block:: text

     call void @llvm.vp.scatter.v8i8.v8p0(<8 x i8> %val, <8 x ptr> align 1 %ptrs, <8 x i1> %mask, i32 %evl)
     ;; For all lanes below %evl, the call above is lane-wise equivalent to the call below.

     call void @llvm.masked.scatter.v8i8.v8p0(<8 x i8> %val, <8 x ptr> %ptrs, i32 1, <8 x i1> %mask)


.. _int_vp_trunc:

'``llvm.vp.trunc.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i16>  @llvm.vp.trunc.v16i16.v16i32 (<16 x i32> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i16>  @llvm.vp.trunc.nxv4i16.nxv4i32 (<vscale x 4 x i32> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

The '``llvm.vp.trunc``' intrinsic truncates its first argument to the return
type. The operation has a mask and an explicit vector length parameter.


Arguments:
""""""""""

The '``llvm.vp.trunc``' intrinsic takes a value to cast as its first argument.
The return type is the type to cast the value to. Both types must be vector of
:ref:`integer <t_integer>` type. The bit size of the value must be larger than
the bit size of the return type. The second argument is the vector mask. The
return type, the value to cast, and the vector mask have the same number of
elements.  The third argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.trunc``' intrinsic truncates the high order bits in value and
converts the remaining bits to return type. Since the source size must be larger
than the destination size, '``llvm.vp.trunc``' cannot be a *no-op cast*. It will
always truncate bits. The conversion is performed on lane positions below the
explicit vector length and where the vector mask is true.  Masked-off lanes are
``poison``.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i16> @llvm.vp.trunc.v4i16.v4i32(<4 x i32> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = trunc <4 x i32> %a to <4 x i16>
      %also.r = select <4 x i1> %mask, <4 x i16> %t, <4 x i16> poison


.. _int_vp_zext:

'``llvm.vp.zext.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.zext.v16i32.v16i16 (<16 x i16> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.zext.nxv4i32.nxv4i16 (<vscale x 4 x i16> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

The '``llvm.vp.zext``' intrinsic zero extends its first argument to the return
type. The operation has a mask and an explicit vector length parameter.


Arguments:
""""""""""

The '``llvm.vp.zext``' intrinsic takes a value to cast as its first argument.
The return type is the type to cast the value to. Both types must be vectors of
:ref:`integer <t_integer>` type. The bit size of the value must be smaller than
the bit size of the return type. The second argument is the vector mask. The
return type, the value to cast, and the vector mask have the same number of
elements.  The third argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.zext``' intrinsic fill the high order bits of the value with zero
bits until it reaches the size of the return type. When zero extending from i1,
the result will always be either 0 or 1. The conversion is performed on lane
positions below the explicit vector length and where the vector mask is true.
Masked-off lanes are ``poison``.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.zext.v4i32.v4i16(<4 x i16> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = zext <4 x i16> %a to <4 x i32>
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_sext:

'``llvm.vp.sext.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.sext.v16i32.v16i16 (<16 x i16> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.sext.nxv4i32.nxv4i16 (<vscale x 4 x i16> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

The '``llvm.vp.sext``' intrinsic sign extends its first argument to the return
type. The operation has a mask and an explicit vector length parameter.


Arguments:
""""""""""

The '``llvm.vp.sext``' intrinsic takes a value to cast as its first argument.
The return type is the type to cast the value to. Both types must be vectors of
:ref:`integer <t_integer>` type. The bit size of the value must be smaller than
the bit size of the return type. The second argument is the vector mask. The
return type, the value to cast, and the vector mask have the same number of
elements.  The third argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.sext``' intrinsic performs a sign extension by copying the sign
bit (highest order bit) of the value until it reaches the size of the return
type. When sign extending from i1, the result will always be either -1 or 0.
The conversion is performed on lane positions below the explicit vector length
and where the vector mask is true. Masked-off lanes are ``poison``.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.sext.v4i32.v4i16(<4 x i16> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = sext <4 x i16> %a to <4 x i32>
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_fptrunc:

'``llvm.vp.fptrunc.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.fptrunc.v16f32.v16f64 (<16 x double> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.trunc.nxv4f32.nxv4f64 (<vscale x 4 x double> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

The '``llvm.vp.fptrunc``' intrinsic truncates its first argument to the return
type. The operation has a mask and an explicit vector length parameter.


Arguments:
""""""""""

The '``llvm.vp.fptrunc``' intrinsic takes a value to cast as its first argument.
The return type is the type to cast the value to. Both types must be vector of
:ref:`floating-point <t_floating>` type. The bit size of the value must be
larger than the bit size of the return type. This implies that
'``llvm.vp.fptrunc``' cannot be used to make a *no-op cast*. The second argument
is the vector mask. The return type, the value to cast, and the vector mask have
the same number of elements.  The third argument is the explicit vector length of
the operation.

Semantics:
""""""""""

The '``llvm.vp.fptrunc``' intrinsic casts a ``value`` from a larger
:ref:`floating-point <t_floating>` type to a smaller :ref:`floating-point
<t_floating>` type.
This instruction is assumed to execute in the default :ref:`floating-point
environment <floatenv>`. The conversion is performed on lane positions below the
explicit vector length and where the vector mask is true.  Masked-off lanes are
``poison``.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.fptrunc.v4f32.v4f64(<4 x double> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = fptrunc <4 x double> %a to <4 x float>
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_fpext:

'``llvm.vp.fpext.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x double>  @llvm.vp.fpext.v16f64.v16f32 (<16 x float> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x double>  @llvm.vp.fpext.nxv4f64.nxv4f32 (<vscale x 4 x float> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

The '``llvm.vp.fpext``' intrinsic extends its first argument to the return
type. The operation has a mask and an explicit vector length parameter.


Arguments:
""""""""""

The '``llvm.vp.fpext``' intrinsic takes a value to cast as its first argument.
The return type is the type to cast the value to. Both types must be vector of
:ref:`floating-point <t_floating>` type. The bit size of the value must be
smaller than the bit size of the return type. This implies that
'``llvm.vp.fpext``' cannot be used to make a *no-op cast*. The second argument
is the vector mask. The return type, the value to cast, and the vector mask have
the same number of elements.  The third argument is the explicit vector length of
the operation.

Semantics:
""""""""""

The '``llvm.vp.fpext``' intrinsic extends the ``value`` from a smaller
:ref:`floating-point <t_floating>` type to a larger :ref:`floating-point
<t_floating>` type. The '``llvm.vp.fpext``' cannot be used to make a
*no-op cast* because it always changes bits. Use ``bitcast`` to make a
*no-op cast* for a floating-point cast.
The conversion is performed on lane positions below the explicit vector length
and where the vector mask is true.  Masked-off lanes are ``poison``.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x double> @llvm.vp.fpext.v4f64.v4f32(<4 x float> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = fpext <4 x float> %a to <4 x double>
      %also.r = select <4 x i1> %mask, <4 x double> %t, <4 x double> poison


.. _int_vp_fptoui:

'``llvm.vp.fptoui.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.fptoui.v16i32.v16f32 (<16 x float> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.fptoui.nxv4i32.nxv4f32 (<vscale x 4 x float> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.fptoui.v256i64.v256f64 (<256 x double> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

The '``llvm.vp.fptoui``' intrinsic converts the :ref:`floating-point
<t_floating>` argument to the unsigned integer return type.
The operation has a mask and an explicit vector length parameter.


Arguments:
""""""""""

The '``llvm.vp.fptoui``' intrinsic takes a value to cast as its first argument.
The value to cast must be a vector of :ref:`floating-point <t_floating>` type.
The return type is the type to cast the value to. The return type must be
vector of :ref:`integer <t_integer>` type.  The second argument is the vector
mask. The return type, the value to cast, and the vector mask have the same
number of elements.  The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.fptoui``' intrinsic converts its :ref:`floating-point
<t_floating>` argument into the nearest (rounding towards zero) unsigned integer
value where the lane position is below the explicit vector length and the
vector mask is true.  Masked-off lanes are ``poison``. On enabled lanes where
conversion takes place and the value cannot fit in the return type, the result
on that lane is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.fptoui.v4i32.v4f32(<4 x float> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = fptoui <4 x float> %a to <4 x i32>
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_fptosi:

'``llvm.vp.fptosi.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.fptosi.v16i32.v16f32 (<16 x float> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.fptosi.nxv4i32.nxv4f32 (<vscale x 4 x float> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.fptosi.v256i64.v256f64 (<256 x double> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

The '``llvm.vp.fptosi``' intrinsic converts the :ref:`floating-point
<t_floating>` argument to the signed integer return type.
The operation has a mask and an explicit vector length parameter.


Arguments:
""""""""""

The '``llvm.vp.fptosi``' intrinsic takes a value to cast as its first argument.
The value to cast must be a vector of :ref:`floating-point <t_floating>` type.
The return type is the type to cast the value to. The return type must be
vector of :ref:`integer <t_integer>` type.  The second argument is the vector
mask. The return type, the value to cast, and the vector mask have the same
number of elements.  The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.fptosi``' intrinsic converts its :ref:`floating-point
<t_floating>` argument into the nearest (rounding towards zero) signed integer
value where the lane position is below the explicit vector length and the
vector mask is true.  Masked-off lanes are ``poison``. On enabled lanes where
conversion takes place and the value cannot fit in the return type, the result
on that lane is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.fptosi.v4i32.v4f32(<4 x float> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = fptosi <4 x float> %a to <4 x i32>
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_uitofp:

'``llvm.vp.uitofp.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.uitofp.v16f32.v16i32 (<16 x i32> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.uitofp.nxv4f32.nxv4i32 (<vscale x 4 x i32> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.uitofp.v256f64.v256i64 (<256 x i64> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

The '``llvm.vp.uitofp``' intrinsic converts its unsigned integer argument to the
:ref:`floating-point <t_floating>` return type.  The operation has a mask and
an explicit vector length parameter.


Arguments:
""""""""""

The '``llvm.vp.uitofp``' intrinsic takes a value to cast as its first argument.
The value to cast must be vector of :ref:`integer <t_integer>` type.  The
return type is the type to cast the value to.  The return type must be a vector
of :ref:`floating-point <t_floating>` type.  The second argument is the vector
mask. The return type, the value to cast, and the vector mask have the same
number of elements.  The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.uitofp``' intrinsic interprets its first argument as an unsigned
integer quantity and converts it to the corresponding floating-point value. If
the value cannot be exactly represented, it is rounded using the default
rounding mode.  The conversion is performed on lane positions below the
explicit vector length and where the vector mask is true.  Masked-off lanes are
``poison``.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.uitofp.v4f32.v4i32(<4 x i32> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = uitofp <4 x i32> %a to <4 x float>
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_sitofp:

'``llvm.vp.sitofp.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.sitofp.v16f32.v16i32 (<16 x i32> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.sitofp.nxv4f32.nxv4i32 (<vscale x 4 x i32> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.sitofp.v256f64.v256i64 (<256 x i64> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

The '``llvm.vp.sitofp``' intrinsic converts its signed integer argument to the
:ref:`floating-point <t_floating>` return type.  The operation has a mask and
an explicit vector length parameter.


Arguments:
""""""""""

The '``llvm.vp.sitofp``' intrinsic takes a value to cast as its first argument.
The value to cast must be vector of :ref:`integer <t_integer>` type.  The
return type is the type to cast the value to.  The return type must be a vector
of :ref:`floating-point <t_floating>` type.  The second argument is the vector
mask. The return type, the value to cast, and the vector mask have the same
number of elements.  The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.sitofp``' intrinsic interprets its first argument as a signed
integer quantity and converts it to the corresponding floating-point value. If
the value cannot be exactly represented, it is rounded using the default
rounding mode.  The conversion is performed on lane positions below the
explicit vector length and where the vector mask is true.  Masked-off lanes are
``poison``.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.sitofp.v4f32.v4i32(<4 x i32> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = sitofp <4 x i32> %a to <4 x float>
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison


.. _int_vp_ptrtoint:

'``llvm.vp.ptrtoint.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i8>  @llvm.vp.ptrtoint.v16i8.v16p0(<16 x ptr> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i8>  @llvm.vp.ptrtoint.nxv4i8.nxv4p0(<vscale x 4 x ptr> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.ptrtoint.v16i64.v16p0(<256 x ptr> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

The '``llvm.vp.ptrtoint``' intrinsic converts its pointer to the integer return
type.  The operation has a mask and an explicit vector length parameter.


Arguments:
""""""""""

The '``llvm.vp.ptrtoint``' intrinsic takes a value to cast as its first argument
, which must be a vector of pointers, and a type to cast it to return type,
which must be a vector of :ref:`integer <t_integer>` type.
The second argument is the vector mask. The return type, the value to cast, and
the vector mask have the same number of elements.
The third argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.ptrtoint``' intrinsic converts value to return type by
interpreting the pointer value as an integer and either truncating or zero
extending that value to the size of the integer type.
If ``value`` is smaller than return type, then a zero extension is done. If
``value`` is larger than return type, then a truncation is done. If they are
the same size, then nothing is done (*no-op cast*) other than a type
change.
The conversion is performed on lane positions below the explicit vector length
and where the vector mask is true.  Masked-off lanes are ``poison``.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i8> @llvm.vp.ptrtoint.v4i8.v4p0i32(<4 x ptr> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = ptrtoint <4 x ptr> %a to <4 x i8>
      %also.r = select <4 x i1> %mask, <4 x i8> %t, <4 x i8> poison


.. _int_vp_inttoptr:

'``llvm.vp.inttoptr.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x ptr>  @llvm.vp.inttoptr.v16p0.v16i32 (<16 x i32> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x ptr>  @llvm.vp.inttoptr.nxv4p0.nxv4i32 (<vscale x 4 x i32> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x ptr>  @llvm.vp.inttoptr.v256p0.v256i32 (<256 x i32> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

The '``llvm.vp.inttoptr``' intrinsic converts its integer value to the point
return type. The operation has a mask and an explicit vector length parameter.


Arguments:
""""""""""

The '``llvm.vp.inttoptr``' intrinsic takes a value to cast as its first argument
, which must be a vector of :ref:`integer <t_integer>` type, and a type to cast
it to return type, which must be a vector of pointers type.
The second argument is the vector mask. The return type, the value to cast, and
the vector mask have the same number of elements.
The third argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.inttoptr``' intrinsic converts ``value`` to return type by
applying either a zero extension or a truncation depending on the size of the
integer ``value``. If ``value`` is larger than the size of a pointer, then a
truncation is done. If ``value`` is smaller than the size of a pointer, then a
zero extension is done. If they are the same size, nothing is done (*no-op cast*).
The conversion is performed on lane positions below the explicit vector length
and where the vector mask is true.  Masked-off lanes are ``poison``.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x ptr> @llvm.vp.inttoptr.v4p0i32.v4i32(<4 x i32> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = inttoptr <4 x i32> %a to <4 x ptr>
      %also.r = select <4 x i1> %mask, <4 x ptr> %t, <4 x ptr> poison


.. _int_vp_fcmp:

'``llvm.vp.fcmp.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i1> @llvm.vp.fcmp.v16f32(<16 x float> <left_op>, <16 x float> <right_op>, metadata <condition code>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i1> @llvm.vp.fcmp.nxv4f32(<vscale x 4 x float> <left_op>, <vscale x 4 x float> <right_op>, metadata <condition code>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i1> @llvm.vp.fcmp.v256f64(<256 x double> <left_op>, <256 x double> <right_op>, metadata <condition code>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

The '``llvm.vp.fcmp``' intrinsic returns a vector of boolean values based on
the comparison of its arguments. The operation has a mask and an explicit vector
length parameter.


Arguments:
""""""""""

The '``llvm.vp.fcmp``' intrinsic takes the two values to compare as its first
and second arguments. These two values must be vectors of :ref:`floating-point
<t_floating>` types.
The return type is the result of the comparison. The return type must be a
vector of :ref:`i1 <t_integer>` type. The fourth argument is the vector mask.
The return type, the values to compare, and the vector mask have the same
number of elements. The third argument is the condition code indicating the kind
of comparison to perform. It must be a metadata string with :ref:`one of the
supported floating-point condition code values <fcmp_md_cc>`. The fifth argument
is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.fcmp``' compares its first two arguments according to the
condition code given as the third argument. The arguments are compared element by
element on each enabled lane, where the semantics of the comparison are
defined :ref:`according to the condition code <fcmp_md_cc_sem>`. Masked-off
lanes are ``poison``.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i1> @llvm.vp.fcmp.v4f32(<4 x float> %a, <4 x float> %b, metadata !"oeq", <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = fcmp oeq <4 x float> %a, %b
      %also.r = select <4 x i1> %mask, <4 x i1> %t, <4 x i1> poison


.. _int_vp_icmp:

'``llvm.vp.icmp.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <32 x i1> @llvm.vp.icmp.v32i32(<32 x i32> <left_op>, <32 x i32> <right_op>, metadata <condition code>, <32 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 2 x i1> @llvm.vp.icmp.nxv2i32(<vscale x 2 x i32> <left_op>, <vscale x 2 x i32> <right_op>, metadata <condition code>, <vscale x 2 x i1> <mask>, i32 <vector_length>)
      declare <128 x i1> @llvm.vp.icmp.v128i8(<128 x i8> <left_op>, <128 x i8> <right_op>, metadata <condition code>, <128 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

The '``llvm.vp.icmp``' intrinsic returns a vector of boolean values based on
the comparison of its arguments. The operation has a mask and an explicit vector
length parameter.


Arguments:
""""""""""

The '``llvm.vp.icmp``' intrinsic takes the two values to compare as its first
and second arguments. These two values must be vectors of :ref:`integer
<t_integer>` types.
The return type is the result of the comparison. The return type must be a
vector of :ref:`i1 <t_integer>` type. The fourth argument is the vector mask.
The return type, the values to compare, and the vector mask have the same
number of elements. The third argument is the condition code indicating the kind
of comparison to perform. It must be a metadata string with :ref:`one of the
supported integer condition code values <icmp_md_cc>`. The fifth argument is the
explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.icmp``' compares its first two arguments according to the
condition code given as the third argument. The arguments are compared element by
element on each enabled lane, where the semantics of the comparison are
defined :ref:`according to the condition code <icmp_md_cc_sem>`. Masked-off
lanes are ``poison``.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i1> @llvm.vp.icmp.v4i32(<4 x i32> %a, <4 x i32> %b, metadata !"ne", <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = icmp ne <4 x i32> %a, %b
      %also.r = select <4 x i1> %mask, <4 x i1> %t, <4 x i1> poison

.. _int_vp_ceil:

'``llvm.vp.ceil.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.ceil.v16f32 (<16 x float> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.ceil.nxv4f32 (<vscale x 4 x float> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.ceil.v256f64 (<256 x double> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point ceiling of a vector of floating-point values.


Arguments:
""""""""""

The first argument and the result have the same vector of floating-point type.
The second argument is the vector mask and has the same number of elements as the
result vector type. The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.ceil``' intrinsic performs floating-point ceiling
(:ref:`ceil <int_ceil>`) of the first vector argument on each enabled lane. The
result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.ceil.v4f32(<4 x float> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.ceil.v4f32(<4 x float> %a)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison

.. _int_vp_floor:

'``llvm.vp.floor.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.floor.v16f32 (<16 x float> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.floor.nxv4f32 (<vscale x 4 x float> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.floor.v256f64 (<256 x double> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point floor of a vector of floating-point values.


Arguments:
""""""""""

The first argument and the result have the same vector of floating-point type.
The second argument is the vector mask and has the same number of elements as the
result vector type. The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.floor``' intrinsic performs floating-point floor
(:ref:`floor <int_floor>`) of the first vector argument on each enabled lane.
The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.floor.v4f32(<4 x float> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.floor.v4f32(<4 x float> %a)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison

.. _int_vp_rint:

'``llvm.vp.rint.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.rint.v16f32 (<16 x float> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.rint.nxv4f32 (<vscale x 4 x float> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.rint.v256f64 (<256 x double> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point rint of a vector of floating-point values.


Arguments:
""""""""""

The first argument and the result have the same vector of floating-point type.
The second argument is the vector mask and has the same number of elements as the
result vector type. The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.rint``' intrinsic performs floating-point rint
(:ref:`rint <int_rint>`) of the first vector argument on each enabled lane.
The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.rint.v4f32(<4 x float> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.rint.v4f32(<4 x float> %a)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison

.. _int_vp_nearbyint:

'``llvm.vp.nearbyint.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.nearbyint.v16f32 (<16 x float> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.nearbyint.nxv4f32 (<vscale x 4 x float> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.nearbyint.v256f64 (<256 x double> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point nearbyint of a vector of floating-point values.


Arguments:
""""""""""

The first argument and the result have the same vector of floating-point type.
The second argument is the vector mask and has the same number of elements as the
result vector type. The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.nearbyint``' intrinsic performs floating-point nearbyint
(:ref:`nearbyint <int_nearbyint>`) of the first vector argument on each enabled lane.
The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.nearbyint.v4f32(<4 x float> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.nearbyint.v4f32(<4 x float> %a)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison

.. _int_vp_round:

'``llvm.vp.round.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.round.v16f32 (<16 x float> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.round.nxv4f32 (<vscale x 4 x float> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.round.v256f64 (<256 x double> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point round of a vector of floating-point values.


Arguments:
""""""""""

The first argument and the result have the same vector of floating-point type.
The second argument is the vector mask and has the same number of elements as the
result vector type. The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.round``' intrinsic performs floating-point round
(:ref:`round <int_round>`) of the first vector argument on each enabled lane.
The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.round.v4f32(<4 x float> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.round.v4f32(<4 x float> %a)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison

.. _int_vp_roundeven:

'``llvm.vp.roundeven.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.roundeven.v16f32 (<16 x float> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.roundeven.nxv4f32 (<vscale x 4 x float> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.roundeven.v256f64 (<256 x double> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point roundeven of a vector of floating-point values.


Arguments:
""""""""""

The first argument and the result have the same vector of floating-point type.
The second argument is the vector mask and has the same number of elements as the
result vector type. The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.roundeven``' intrinsic performs floating-point roundeven
(:ref:`roundeven <int_roundeven>`) of the first vector argument on each enabled
lane. The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.roundeven.v4f32(<4 x float> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.roundeven.v4f32(<4 x float> %a)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison

.. _int_vp_roundtozero:

'``llvm.vp.roundtozero.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x float>  @llvm.vp.roundtozero.v16f32 (<16 x float> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x float>  @llvm.vp.roundtozero.nxv4f32 (<vscale x 4 x float> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x double>  @llvm.vp.roundtozero.v256f64 (<256 x double> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated floating-point round-to-zero of a vector of floating-point values.


Arguments:
""""""""""

The first argument and the result have the same vector of floating-point type.
The second argument is the vector mask and has the same number of elements as the
result vector type. The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.roundtozero``' intrinsic performs floating-point roundeven
(:ref:`llvm.trunc <int_llvm_trunc>`) of the first vector argument on each enabled lane.  The
result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x float> @llvm.vp.roundtozero.v4f32(<4 x float> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x float> @llvm.trunc.v4f32(<4 x float> %a)
      %also.r = select <4 x i1> %mask, <4 x float> %t, <4 x float> poison

.. _int_vp_lrint:

'``llvm.vp.lrint.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32> @llvm.vp.lrint.v16i32.v16f32(<16 x float> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32> @llvm.vp.lrint.nxv4i32.nxv4f32(<vscale x 4 x float> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64> @llvm.vp.lrint.v256i64.v256f64(<256 x double> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated lrint of a vector of floating-point values.


Arguments:
""""""""""

The result is an integer vector and the first argument is a vector of :ref:`floating-point <t_floating>`
type with the same number of elements as the result vector type. The second
argument is the vector mask and has the same number of elements as the result
vector type. The third argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.lrint``' intrinsic performs lrint (:ref:`lrint <int_lrint>`) of
the first vector argument on each enabled lane. The result on disabled lanes is a
:ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.lrint.v4i32.v4f32(<4 x float> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.lrint.v4f32(<4 x float> %a)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison

.. _int_vp_llrint:

'``llvm.vp.llrint.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32> @llvm.vp.llrint.v16i32.v16f32(<16 x float> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32> @llvm.vp.llrint.nxv4i32.nxv4f32(<vscale x 4 x float> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64> @llvm.vp.llrint.v256i64.v256f64(<256 x double> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated llrint of a vector of floating-point values.


Arguments:
""""""""""
The result is an integer vector and the first argument is a vector of :ref:`floating-point <t_floating>`
type with the same number of elements as the result vector type. The second
argument is the vector mask and has the same number of elements as the result
vector type. The third argument is the explicit vector length of the operation.

Semantics:
""""""""""

The '``llvm.vp.llrint``' intrinsic performs lrint (:ref:`llrint <int_llrint>`) of
the first vector argument on each enabled lane. The result on disabled lanes is a
:ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.llrint.v4i32.v4f32(<4 x float> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.llrint.v4f32(<4 x float> %a)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_bitreverse:

'``llvm.vp.bitreverse.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.bitreverse.v16i32 (<16 x i32> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.bitreverse.nxv4i32 (<vscale x 4 x i32> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.bitreverse.v256i64 (<256 x i64> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated bitreverse of a vector of integers.


Arguments:
""""""""""

The first argument and the result have the same vector of integer type. The
second argument is the vector mask and has the same number of elements as the
result vector type. The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.bitreverse``' intrinsic performs bitreverse (:ref:`bitreverse <int_bitreverse>`) of the first argument on each
enabled lane.  The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.bitreverse.v4i32(<4 x i32> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.bitreverse.v4i32(<4 x i32> %a)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_bswap:

'``llvm.vp.bswap.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.bswap.v16i32 (<16 x i32> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.bswap.nxv4i32 (<vscale x 4 x i32> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.bswap.v256i64 (<256 x i64> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated bswap of a vector of integers.


Arguments:
""""""""""

The first argument and the result have the same vector of integer type. The
second argument is the vector mask and has the same number of elements as the
result vector type. The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.bswap``' intrinsic performs bswap (:ref:`bswap <int_bswap>`) of the first argument on each
enabled lane.  The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.bswap.v4i32(<4 x i32> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.bswap.v4i32(<4 x i32> %a)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_ctpop:

'``llvm.vp.ctpop.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.ctpop.v16i32 (<16 x i32> <op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.ctpop.nxv4i32 (<vscale x 4 x i32> <op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.ctpop.v256i64 (<256 x i64> <op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated ctpop of a vector of integers.


Arguments:
""""""""""

The first argument and the result have the same vector of integer type. The
second argument is the vector mask and has the same number of elements as the
result vector type. The third argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.ctpop``' intrinsic performs ctpop (:ref:`ctpop <int_ctpop>`) of the first argument on each
enabled lane.  The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.ctpop.v4i32(<4 x i32> %a, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.ctpop.v4i32(<4 x i32> %a)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_ctlz:

'``llvm.vp.ctlz.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.ctlz.v16i32 (<16 x i32> <op>, i1 <is_zero_poison>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.ctlz.nxv4i32 (<vscale x 4 x i32> <op>, i1 <is_zero_poison>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.ctlz.v256i64 (<256 x i64> <op>, i1 <is_zero_poison>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated ctlz of a vector of integers.


Arguments:
""""""""""

The first argument and the result have the same vector of integer type. The
second argument is a constant flag that indicates whether the intrinsic returns
a valid result if the first argument is zero. The third argument is the vector
mask and has the same number of elements as the result vector type. the fourth
argument is the explicit vector length of the operation. If the first argument
is zero and the second argument is true, the result is poison.

Semantics:
""""""""""

The '``llvm.vp.ctlz``' intrinsic performs ctlz (:ref:`ctlz <int_ctlz>`) of the first argument on each
enabled lane.  The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.ctlz.v4i32(<4 x i32> %a, i1 false, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.ctlz.v4i32(<4 x i32> %a, i1 false)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_cttz:

'``llvm.vp.cttz.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.cttz.v16i32 (<16 x i32> <op>, i1 <is_zero_poison>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.cttz.nxv4i32 (<vscale x 4 x i32> <op>, i1 <is_zero_poison>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.cttz.v256i64 (<256 x i64> <op>, i1 <is_zero_poison>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated cttz of a vector of integers.


Arguments:
""""""""""

The first argument and the result have the same vector of integer type. The
second argument is a constant flag that indicates whether the intrinsic
returns a valid result if the first argument is zero. The third argument is
the vector mask and has the same number of elements as the result vector type.
The fourth argument is the explicit vector length of the operation. If the
first argument is zero and the second argument is true, the result is poison.

Semantics:
""""""""""

The '``llvm.vp.cttz``' intrinsic performs cttz (:ref:`cttz <int_cttz>`) of the first argument on each
enabled lane.  The result on disabled lanes is a :ref:`poison value <poisonvalues>`.

Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.cttz.v4i32(<4 x i32> %a, i1 false, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.cttz.v4i32(<4 x i32> %a, i1 false)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_cttz_elts:

'``llvm.vp.cttz.elts.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic. You can use ```llvm.vp.cttz.elts``` on any
vector of integer elements, both fixed width and scalable.

::

      declare i32  @llvm.vp.cttz.elts.i32.v16i32 (<16 x i32> <op>, i1 <is_zero_poison>, <16 x i1> <mask>, i32 <vector_length>)
      declare i64  @llvm.vp.cttz.elts.i64.nxv4i32 (<vscale x 4 x i32> <op>, i1 <is_zero_poison>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare i64  @llvm.vp.cttz.elts.i64.v256i1 (<256 x i1> <op>, i1 <is_zero_poison>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

This '```llvm.vp.cttz.elts```' intrinsic counts the number of trailing zero
elements of a vector. This is basically the vector-predicated version of
'```llvm.experimental.cttz.elts```'.

Arguments:
""""""""""

The first argument is the vector to be counted. This argument must be a vector
with integer element type. The return type must also be an integer type which is
wide enough to hold the maximum number of elements of the source vector. The
behavior of this intrinsic is undefined if the return type is not wide enough
for the number of elements in the input vector.

The second argument is a constant flag that indicates whether the intrinsic
returns a valid result if the first argument is all zero.

The third argument is the vector mask and has the same number of elements as the
input vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.cttz.elts``' intrinsic counts the trailing (least
significant / lowest-numbered) zero elements in the first argument on each
enabled lane. If the first argument is all zero and the second argument is true,
the result is poison. Otherwise, it returns the explicit vector length (i.e. the
fourth argument).

.. _int_vp_sadd_sat:

'``llvm.vp.sadd.sat.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.sadd.sat.v16i32 (<16 x i32> <left_op> <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.sadd.sat.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.sadd.sat.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated signed saturating addition of two vectors of integers.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.sadd.sat``' intrinsic performs sadd.sat (:ref:`sadd.sat <int_sadd_sat>`)
of the first and second vector arguments on each enabled lane. The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.


Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.sadd.sat.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.sadd.sat.v4i32(<4 x i32> %a, <4 x i32> %b)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_uadd_sat:

'``llvm.vp.uadd.sat.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.uadd.sat.v16i32 (<16 x i32> <left_op> <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.uadd.sat.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.uadd.sat.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated unsigned saturating addition of two vectors of integers.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.uadd.sat``' intrinsic performs uadd.sat (:ref:`uadd.sat <int_uadd_sat>`)
of the first and second vector arguments on each enabled lane. The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.


Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.uadd.sat.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.uadd.sat.v4i32(<4 x i32> %a, <4 x i32> %b)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_ssub_sat:

'``llvm.vp.ssub.sat.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.ssub.sat.v16i32 (<16 x i32> <left_op> <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.ssub.sat.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.ssub.sat.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated signed saturating subtraction of two vectors of integers.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.ssub.sat``' intrinsic performs ssub.sat (:ref:`ssub.sat <int_ssub_sat>`)
of the first and second vector arguments on each enabled lane. The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.


Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.ssub.sat.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.ssub.sat.v4i32(<4 x i32> %a, <4 x i32> %b)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_usub_sat:

'``llvm.vp.usub.sat.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.usub.sat.v16i32 (<16 x i32> <left_op> <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.usub.sat.nxv4i32 (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.usub.sat.v256i64 (<256 x i64> <left_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated unsigned saturating subtraction of two vectors of integers.


Arguments:
""""""""""

The first two arguments and the result have the same vector of integer type. The
third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.usub.sat``' intrinsic performs usub.sat (:ref:`usub.sat <int_usub_sat>`)
of the first and second vector arguments on each enabled lane. The result on
disabled lanes is a :ref:`poison value <poisonvalues>`.


Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.usub.sat.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.usub.sat.v4i32(<4 x i32> %a, <4 x i32> %b)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


.. _int_vp_fshl:

'``llvm.vp.fshl.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.fshl.v16i32 (<16 x i32> <left_op>, <16 x i32> <middle_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.fshl.nxv4i32  (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <middle_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.fshl.v256i64 (<256 x i64> <left_op>, <256 x i64> <middle_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated fshl of three vectors of integers.


Arguments:
""""""""""

The first three arguments and the result have the same vector of integer type. The
fourth argument is the vector mask and has the same number of elements as the
result vector type. The fifth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.fshl``' intrinsic performs fshl (:ref:`fshl <int_fshl>`) of the first, second, and third
vector argument on each enabled lane. The result on disabled lanes is a :ref:`poison value <poisonvalues>`.


Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.fshl.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i32> %c, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.fshl.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i32> %c)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison


'``llvm.vp.fshr.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <16 x i32>  @llvm.vp.fshr.v16i32 (<16 x i32> <left_op>, <16 x i32> <middle_op>, <16 x i32> <right_op>, <16 x i1> <mask>, i32 <vector_length>)
      declare <vscale x 4 x i32>  @llvm.vp.fshr.nxv4i32  (<vscale x 4 x i32> <left_op>, <vscale x 4 x i32> <middle_op>, <vscale x 4 x i32> <right_op>, <vscale x 4 x i1> <mask>, i32 <vector_length>)
      declare <256 x i64>  @llvm.vp.fshr.v256i64 (<256 x i64> <left_op>, <256 x i64> <middle_op>, <256 x i64> <right_op>, <256 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated fshr of three vectors of integers.


Arguments:
""""""""""

The first three arguments and the result have the same vector of integer type. The
fourth argument is the vector mask and has the same number of elements as the
result vector type. The fifth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.fshr``' intrinsic performs fshr (:ref:`fshr <int_fshr>`) of the first, second, and third
vector argument on each enabled lane. The result on disabled lanes is a :ref:`poison value <poisonvalues>`.


Examples:
"""""""""

.. code-block:: llvm

      %r = call <4 x i32> @llvm.vp.fshr.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i32> %c, <4 x i1> %mask, i32 %evl)
      ;; For all lanes below %evl, %r is lane-wise equivalent to %also.r

      %t = call <4 x i32> @llvm.fshr.v4i32(<4 x i32> %a, <4 x i32> %b, <4 x i32> %c)
      %also.r = select <4 x i1> %mask, <4 x i32> %t, <4 x i32> poison

'``llvm.vp.is.fpclass.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic.

::

      declare <vscale x 2 x i1> @llvm.vp.is.fpclass.nxv2f32(<vscale x 2 x float> <op>, i32 <test>, <vscale x 2 x i1> <mask>, i32 <vector_length>)
      declare <2 x i1> @llvm.vp.is.fpclass.v2f16(<2 x half> <op>, i32 <test>, <2 x i1> <mask>, i32 <vector_length>)

Overview:
"""""""""

Predicated llvm.is.fpclass :ref:`llvm.is.fpclass <llvm.is.fpclass>`

Arguments:
""""""""""

The first argument is a floating-point vector, the result type is a vector of
boolean with the same number of elements as the first argument.  The second
argument specifies, which tests to perform :ref:`llvm.is.fpclass <llvm.is.fpclass>`.
The third argument is the vector mask and has the same number of elements as the
result vector type. The fourth argument is the explicit vector length of the
operation.

Semantics:
""""""""""

The '``llvm.vp.is.fpclass``' intrinsic performs llvm.is.fpclass (:ref:`llvm.is.fpclass <llvm.is.fpclass>`).


Examples:
"""""""""

.. code-block:: llvm

      %r = call <2 x i1> @llvm.vp.is.fpclass.v2f16(<2 x half> %x, i32 3, <2 x i1> %m, i32 %evl)
      %t = call <vscale x 2 x i1> @llvm.vp.is.fpclass.nxv2f16(<vscale x 2 x half> %x, i32 3, <vscale x 2 x i1> %m, i32 %evl)

.. _int_mload_mstore:

Masked Vector Load and Store Intrinsics
---------------------------------------

LLVM provides intrinsics for predicated vector load and store operations. The predicate is specified by a mask argument, which holds one bit per vector element, switching the associated vector lane on or off. The memory addresses corresponding to the "off" lanes are not accessed. When all bits of the mask are on, the intrinsic is identical to a regular vector load or store. When all bits are off, no memory is accessed.

.. _int_mload:

'``llvm.masked.load.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic. The loaded data is a vector of any integer, floating-point or pointer data type.

::

      declare <16 x float>  @llvm.masked.load.v16f32.p0(ptr <ptr>, i32 <alignment>, <16 x i1> <mask>, <16 x float> <passthru>)
      declare <2 x double>  @llvm.masked.load.v2f64.p0(ptr <ptr>, i32 <alignment>, <2 x i1>  <mask>, <2 x double> <passthru>)
      ;; The data is a vector of pointers
      declare <8 x ptr> @llvm.masked.load.v8p0.p0(ptr <ptr>, i32 <alignment>, <8 x i1> <mask>, <8 x ptr> <passthru>)

Overview:
"""""""""

Reads a vector from memory according to the provided mask. The mask holds a bit for each vector lane, and is used to prevent memory accesses to the masked-off lanes. The masked-off lanes in the result vector are taken from the corresponding lanes of the '``passthru``' argument.


Arguments:
""""""""""

The first argument is the base pointer for the load. The second argument is the alignment of the source location. It must be a power of two constant integer value. The third argument, mask, is a vector of boolean values with the same number of elements as the return type. The fourth is a pass-through value that is used to fill the masked-off lanes of the result. The return type, underlying type of the base pointer and the type of the '``passthru``' argument are the same vector types.

Semantics:
""""""""""

The '``llvm.masked.load``' intrinsic is designed for conditional reading of selected vector elements in a single IR operation. It is useful for targets that support vector masked loads and allows vectorizing predicated basic blocks on these targets. Other targets may support this intrinsic differently, for example by lowering it into a sequence of branches that guard scalar load operations.
The result of this operation is equivalent to a regular vector load instruction followed by a 'select' between the loaded and the passthru values, predicated on the same mask, except that the masked-off lanes are not accessed.
Only the masked-on lanes of the vector need to be inbounds of an allocation (but all these lanes need to be inbounds of the same allocation).
In particular, using this intrinsic prevents exceptions on memory accesses to masked-off lanes.
Masked-off lanes are also not considered accessed for the purpose of data races or ``noalias`` constraints.


::

       %res = call <16 x float> @llvm.masked.load.v16f32.p0(ptr %ptr, i32 4, <16 x i1>%mask, <16 x float> %passthru)

       ;; The result of the two following instructions is identical aside from potential memory access exception
       %loadlal = load <16 x float>, ptr %ptr, align 4
       %res = select <16 x i1> %mask, <16 x float> %loadlal, <16 x float> %passthru

.. _int_mstore:

'``llvm.masked.store.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic. The data stored in memory is a vector of any integer, floating-point or pointer data type.

::

       declare void @llvm.masked.store.v8i32.p0 (<8  x i32>   <value>, ptr <ptr>, i32 <alignment>, <8  x i1> <mask>)
       declare void @llvm.masked.store.v16f32.p0(<16 x float> <value>, ptr <ptr>, i32 <alignment>, <16 x i1> <mask>)
       ;; The data is a vector of pointers
       declare void @llvm.masked.store.v8p0.p0  (<8 x ptr>    <value>, ptr <ptr>, i32 <alignment>, <8 x i1> <mask>)

Overview:
"""""""""

Writes a vector to memory according to the provided mask. The mask holds a bit for each vector lane, and is used to prevent memory accesses to the masked-off lanes.

Arguments:
""""""""""

The first argument is the vector value to be written to memory. The second argument is the base pointer for the store, it has the same underlying type as the value argument. The third argument is the alignment of the destination location. It must be a power of two constant integer value. The fourth argument, mask, is a vector of boolean values. The types of the mask and the value argument must have the same number of vector elements.


Semantics:
""""""""""

The '``llvm.masked.store``' intrinsics is designed for conditional writing of selected vector elements in a single IR operation. It is useful for targets that support vector masked store and allows vectorizing predicated basic blocks on these targets. Other targets may support this intrinsic differently, for example by lowering it into a sequence of branches that guard scalar store operations.
The result of this operation is equivalent to a load-modify-store sequence, except that the masked-off lanes are not accessed.
Only the masked-on lanes of the vector need to be inbounds of an allocation (but all these lanes need to be inbounds of the same allocation).
In particular, using this intrinsic prevents exceptions on memory accesses to masked-off lanes.
Masked-off lanes are also not considered accessed for the purpose of data races or ``noalias`` constraints.

::

       call void @llvm.masked.store.v16f32.p0(<16 x float> %value, ptr %ptr, i32 4,  <16 x i1> %mask)

       ;; The result of the following instructions is identical aside from potential data races and memory access exceptions
       %oldval = load <16 x float>, ptr %ptr, align 4
       %res = select <16 x i1> %mask, <16 x float> %value, <16 x float> %oldval
       store <16 x float> %res, ptr %ptr, align 4


Masked Vector Gather and Scatter Intrinsics
-------------------------------------------

LLVM provides intrinsics for vector gather and scatter operations. They are similar to :ref:`Masked Vector Load and Store <int_mload_mstore>`, except they are designed for arbitrary memory accesses, rather than sequential memory accesses. Gather and scatter also employ a mask argument, which holds one bit per vector element, switching the associated vector lane on or off. The memory addresses corresponding to the "off" lanes are not accessed. When all bits are off, no memory is accessed.

.. _int_mgather:

'``llvm.masked.gather.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic. The loaded data are multiple scalar values of any integer, floating-point or pointer data type gathered together into one vector.

::

      declare <16 x float> @llvm.masked.gather.v16f32.v16p0(<16 x ptr> <ptrs>, i32 <alignment>, <16 x i1> <mask>, <16 x float> <passthru>)
      declare <2 x double> @llvm.masked.gather.v2f64.v2p1(<2 x ptr addrspace(1)> <ptrs>, i32 <alignment>, <2 x i1>  <mask>, <2 x double> <passthru>)
      declare <8 x ptr> @llvm.masked.gather.v8p0.v8p0(<8 x ptr> <ptrs>, i32 <alignment>, <8 x i1>  <mask>, <8 x ptr> <passthru>)

Overview:
"""""""""

Reads scalar values from arbitrary memory locations and gathers them into one vector. The memory locations are provided in the vector of pointers '``ptrs``'. The memory is accessed according to the provided mask. The mask holds a bit for each vector lane, and is used to prevent memory accesses to the masked-off lanes. The masked-off lanes in the result vector are taken from the corresponding lanes of the '``passthru``' argument.


Arguments:
""""""""""

The first argument is a vector of pointers which holds all memory addresses to read. The second argument is an alignment of the source addresses. It must be 0 or a power of two constant integer value. The third argument, mask, is a vector of boolean values with the same number of elements as the return type. The fourth is a pass-through value that is used to fill the masked-off lanes of the result. The return type, underlying type of the vector of pointers and the type of the '``passthru``' argument are the same vector types.

Semantics:
""""""""""

The '``llvm.masked.gather``' intrinsic is designed for conditional reading of multiple scalar values from arbitrary memory locations in a single IR operation. It is useful for targets that support vector masked gathers and allows vectorizing basic blocks with data and control divergence. Other targets may support this intrinsic differently, for example by lowering it into a sequence of scalar load operations.
The semantics of this operation are equivalent to a sequence of conditional scalar loads with subsequent gathering all loaded values into a single vector. The mask restricts memory access to certain lanes and facilitates vectorization of predicated basic blocks.


::

       %res = call <4 x double> @llvm.masked.gather.v4f64.v4p0(<4 x ptr> %ptrs, i32 8, <4 x i1> <i1 true, i1 true, i1 true, i1 true>, <4 x double> poison)

       ;; The gather with all-true mask is equivalent to the following instruction sequence
       %ptr0 = extractelement <4 x ptr> %ptrs, i32 0
       %ptr1 = extractelement <4 x ptr> %ptrs, i32 1
       %ptr2 = extractelement <4 x ptr> %ptrs, i32 2
       %ptr3 = extractelement <4 x ptr> %ptrs, i32 3

       %val0 = load double, ptr %ptr0, align 8
       %val1 = load double, ptr %ptr1, align 8
       %val2 = load double, ptr %ptr2, align 8
       %val3 = load double, ptr %ptr3, align 8

       %vec0    = insertelement <4 x double> poison, %val0, 0
       %vec01   = insertelement <4 x double> %vec0, %val1, 1
       %vec012  = insertelement <4 x double> %vec01, %val2, 2
       %vec0123 = insertelement <4 x double> %vec012, %val3, 3

.. _int_mscatter:

'``llvm.masked.scatter.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic. The data stored in memory is a vector of any integer, floating-point or pointer data type. Each vector element is stored in an arbitrary memory address. Scatter with overlapping addresses is guaranteed to be ordered from least-significant to most-significant element.

::

       declare void @llvm.masked.scatter.v8i32.v8p0  (<8 x i32>    <value>, <8 x ptr>               <ptrs>, i32 <alignment>, <8 x i1>  <mask>)
       declare void @llvm.masked.scatter.v16f32.v16p1(<16 x float> <value>, <16 x ptr addrspace(1)> <ptrs>, i32 <alignment>, <16 x i1> <mask>)
       declare void @llvm.masked.scatter.v4p0.v4p0   (<4 x ptr>    <value>, <4 x ptr>               <ptrs>, i32 <alignment>, <4 x i1>  <mask>)

Overview:
"""""""""

Writes each element from the value vector to the corresponding memory address. The memory addresses are represented as a vector of pointers. Writing is done according to the provided mask. The mask holds a bit for each vector lane, and is used to prevent memory accesses to the masked-off lanes.

Arguments:
""""""""""

The first argument is a vector value to be written to memory. The second argument is a vector of pointers, pointing to where the value elements should be stored. It has the same underlying type as the value argument. The third argument is an alignment of the destination addresses. It must be 0 or a power of two constant integer value. The fourth argument, mask, is a vector of boolean values. The types of the mask and the value argument must have the same number of vector elements.

Semantics:
""""""""""

The '``llvm.masked.scatter``' intrinsics is designed for writing selected vector elements to arbitrary memory addresses in a single IR operation. The operation may be conditional, when not all bits in the mask are switched on. It is useful for targets that support vector masked scatter and allows vectorizing basic blocks with data and control divergence. Other targets may support this intrinsic differently, for example by lowering it into a sequence of branches that guard scalar store operations.

::

       ;; This instruction unconditionally stores data vector in multiple addresses
       call @llvm.masked.scatter.v8i32.v8p0(<8 x i32> %value, <8 x ptr> %ptrs, i32 4,  <8 x i1>  <true, true, .. true>)

       ;; It is equivalent to a list of scalar stores
       %val0 = extractelement <8 x i32> %value, i32 0
       %val1 = extractelement <8 x i32> %value, i32 1
       ..
       %val7 = extractelement <8 x i32> %value, i32 7
       %ptr0 = extractelement <8 x ptr> %ptrs, i32 0
       %ptr1 = extractelement <8 x ptr> %ptrs, i32 1
       ..
       %ptr7 = extractelement <8 x ptr> %ptrs, i32 7
       ;; Note: the order of the following stores is important when they overlap:
       store i32 %val0, ptr %ptr0, align 4
       store i32 %val1, ptr %ptr1, align 4
       ..
       store i32 %val7, ptr %ptr7, align 4


Masked Vector Expanding Load and Compressing Store Intrinsics
-------------------------------------------------------------

LLVM provides intrinsics for expanding load and compressing store operations. Data selected from a vector according to a mask is stored in consecutive memory addresses (compressed store), and vice-versa (expanding load). These operations effective map to "if (cond.i) a[j++] = v.i" and "if (cond.i) v.i = a[j++]" patterns, respectively. Note that when the mask starts with '1' bits followed by '0' bits, these operations are identical to :ref:`llvm.masked.store <int_mstore>` and :ref:`llvm.masked.load <int_mload>`.

.. _int_expandload:

'``llvm.masked.expandload.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic. Several values of integer, floating point or pointer data type are loaded from consecutive memory addresses and stored into the elements of a vector according to the mask.

::

      declare <16 x float>  @llvm.masked.expandload.v16f32 (ptr <ptr>, <16 x i1> <mask>, <16 x float> <passthru>)
      declare <2 x i64>     @llvm.masked.expandload.v2i64 (ptr <ptr>, <2 x i1>  <mask>, <2 x i64> <passthru>)

Overview:
"""""""""

Reads a number of scalar values sequentially from memory location provided in '``ptr``' and spreads them in a vector. The '``mask``' holds a bit for each vector lane. The number of elements read from memory is equal to the number of '1' bits in the mask. The loaded elements are positioned in the destination vector according to the sequence of '1' and '0' bits in the mask. E.g., if the mask vector is '10010001', "expandload" reads 3 values from memory addresses ptr, ptr+1, ptr+2 and places them in lanes 0, 3 and 7 accordingly. The masked-off lanes are filled by elements from the corresponding lanes of the '``passthru``' argument.


Arguments:
""""""""""

The first argument is the base pointer for the load. It has the same underlying type as the element of the returned vector. The second argument, mask, is a vector of boolean values with the same number of elements as the return type. The third is a pass-through value that is used to fill the masked-off lanes of the result. The return type and the type of the '``passthru``' argument have the same vector type.

The :ref:`align <attr_align>` parameter attribute can be provided for the first
argument. The pointer alignment defaults to 1.

Semantics:
""""""""""

The '``llvm.masked.expandload``' intrinsic is designed for reading multiple scalar values from adjacent memory addresses into possibly non-adjacent vector lanes. It is useful for targets that support vector expanding loads and allows vectorizing loop with cross-iteration dependency like in the following example:

.. code-block:: c

    // In this loop we load from B and spread the elements into array A.
    double *A, B; int *C;
    for (int i = 0; i < size; ++i) {
      if (C[i] != 0)
        A[i] = B[j++];
    }


.. code-block:: llvm

    ; Load several elements from array B and expand them in a vector.
    ; The number of loaded elements is equal to the number of '1' elements in the Mask.
    %Tmp = call <8 x double> @llvm.masked.expandload.v8f64(ptr %Bptr, <8 x i1> %Mask, <8 x double> poison)
    ; Store the result in A
    call void @llvm.masked.store.v8f64.p0(<8 x double> %Tmp, ptr %Aptr, i32 8, <8 x i1> %Mask)

    ; %Bptr should be increased on each iteration according to the number of '1' elements in the Mask.
    %MaskI = bitcast <8 x i1> %Mask to i8
    %MaskIPopcnt = call i8 @llvm.ctpop.i8(i8 %MaskI)
    %MaskI64 = zext i8 %MaskIPopcnt to i64
    %BNextInd = add i64 %BInd, %MaskI64


Other targets may support this intrinsic differently, for example, by lowering it into a sequence of conditional scalar load operations and shuffles.
If all mask elements are '1', the intrinsic behavior is equivalent to the regular unmasked vector load.

.. _int_compressstore:

'``llvm.masked.compressstore.*``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic. A number of scalar values of integer, floating point or pointer data type are collected from an input vector and stored into adjacent memory addresses. A mask defines which elements to collect from the vector.

::

      declare void @llvm.masked.compressstore.v8i32  (<8  x i32>   <value>, ptr <ptr>, <8  x i1> <mask>)
      declare void @llvm.masked.compressstore.v16f32 (<16 x float> <value>, ptr <ptr>, <16 x i1> <mask>)

Overview:
"""""""""

Selects elements from input vector '``value``' according to the '``mask``'. All selected elements are written into adjacent memory addresses starting at address '`ptr`', from lower to higher. The mask holds a bit for each vector lane, and is used to select elements to be stored. The number of elements to be stored is equal to the number of active bits in the mask.

Arguments:
""""""""""

The first argument is the input vector, from which elements are collected and written to memory. The second argument is the base pointer for the store, it has the same underlying type as the element of the input vector argument. The third argument is the mask, a vector of boolean values. The mask and the input vector must have the same number of vector elements.

The :ref:`align <attr_align>` parameter attribute can be provided for the second
argument. The pointer alignment defaults to 1.

Semantics:
""""""""""

The '``llvm.masked.compressstore``' intrinsic is designed for compressing data in memory. It allows to collect elements from possibly non-adjacent lanes of a vector and store them contiguously in memory in one IR operation. It is useful for targets that support compressing store operations and allows vectorizing loops with cross-iteration dependencies like in the following example:

.. code-block:: c

    // In this loop we load elements from A and store them consecutively in B
    double *A, B; int *C;
    for (int i = 0; i < size; ++i) {
      if (C[i] != 0)
        B[j++] = A[i]
    }


.. code-block:: llvm

    ; Load elements from A.
    %Tmp = call <8 x double> @llvm.masked.load.v8f64.p0(ptr %Aptr, i32 8, <8 x i1> %Mask, <8 x double> poison)
    ; Store all selected elements consecutively in array B
    call <void> @llvm.masked.compressstore.v8f64(<8 x double> %Tmp, ptr %Bptr, <8 x i1> %Mask)

    ; %Bptr should be increased on each iteration according to the number of '1' elements in the Mask.
    %MaskI = bitcast <8 x i1> %Mask to i8
    %MaskIPopcnt = call i8 @llvm.ctpop.i8(i8 %MaskI)
    %MaskI64 = zext i8 %MaskIPopcnt to i64
    %BNextInd = add i64 %BInd, %MaskI64


Other targets may support this intrinsic differently, for example, by lowering it into a sequence of branches that guard scalar store operations.


Memory Use Markers
------------------

This class of intrinsics provides information about the
:ref:`lifetime of allocated objects <objectlifetime>` and ranges where variables
are immutable.

.. _int_lifestart:

'``llvm.lifetime.start``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.lifetime.start(i64 <size>, ptr captures(none) <ptr>)

Overview:
"""""""""

The '``llvm.lifetime.start``' intrinsic specifies the start of a memory
object's lifetime.

Arguments:
""""""""""

The first argument is a constant integer representing the size of the
object, or -1 if it is variable sized. The second argument is a pointer
to the object.

Semantics:
""""""""""

If ``ptr`` is a stack-allocated object and it points to the first byte of
the object, the object is initially marked as dead.
``ptr`` is conservatively considered as a non-stack-allocated object if
the stack coloring algorithm that is used in the optimization pipeline cannot
conclude that ``ptr`` is a stack-allocated object.

After '``llvm.lifetime.start``', the stack object that ``ptr`` points is marked
as alive and has an uninitialized value.
The stack object is marked as dead when either
:ref:`llvm.lifetime.end <int_lifeend>` to the alloca is executed or the
function returns.

After :ref:`llvm.lifetime.end <int_lifeend>` is called,
'``llvm.lifetime.start``' on the stack object can be called again.
The second '``llvm.lifetime.start``' call marks the object as alive, but it
does not change the address of the object.

If ``ptr`` is a non-stack-allocated object, it does not point to the first
byte of the object or it is a stack object that is already alive, it simply
fills all bytes of the object with ``poison``.


.. _int_lifeend:

'``llvm.lifetime.end``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.lifetime.end(i64 <size>, ptr captures(none) <ptr>)

Overview:
"""""""""

The '``llvm.lifetime.end``' intrinsic specifies the end of a
:ref:`allocated object's lifetime<objectlifetime>`.

Arguments:
""""""""""

The first argument is a constant integer representing the size of the
object, or -1 if it is variable sized. The second argument is a pointer
to the object.

Semantics:
""""""""""

If ``ptr`` is a stack-allocated object and it points to the first byte of the
object, the object is dead.
``ptr`` is conservatively considered as a non-stack-allocated object if
the stack coloring algorithm that is used in the optimization pipeline cannot
conclude that ``ptr`` is a stack-allocated object.

Calling ``llvm.lifetime.end`` on an already dead alloca is no-op.

If ``ptr`` is a non-stack-allocated object or it does not point to the first
byte of the object, it is equivalent to simply filling all bytes of the object
with ``poison``.


'``llvm.invariant.start``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic. The :ref:`allocated object<allocatedobjects>`
can belong to any address space.

::

      declare ptr @llvm.invariant.start.p0(i64 <size>, ptr captures(none) <ptr>)

Overview:
"""""""""

The '``llvm.invariant.start``' intrinsic specifies that the contents of
an :ref:`allocated object<allocatedobjects>` will not change.

Arguments:
""""""""""

The first argument is a constant integer representing the size of the
object, or -1 if it is variable sized. The second argument is a pointer
to the object.

Semantics:
""""""""""

This intrinsic indicates that until an ``llvm.invariant.end`` that uses
the return value, the referenced memory location is constant and
unchanging.

'``llvm.invariant.end``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic. The :ref:`allocated object<allocatedobjects>`
can belong to any address space.

::

      declare void @llvm.invariant.end.p0(ptr <start>, i64 <size>, ptr captures(none) <ptr>)

Overview:
"""""""""

The '``llvm.invariant.end``' intrinsic specifies that the contents of an
:ref:`allocated object<allocatedobjects>` are mutable.

Arguments:
""""""""""

The first argument is the matching ``llvm.invariant.start`` intrinsic.
The second argument is a constant integer representing the size of the
object, or -1 if it is variable sized and the third argument is a
pointer to the object.

Semantics:
""""""""""

This intrinsic indicates that the memory is mutable again.

'``llvm.launder.invariant.group``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic. The :ref:`allocated object<allocatedobjects>`
can belong to any address space. The returned pointer must belong to the same
address space as the argument.

::

      declare ptr @llvm.launder.invariant.group.p0(ptr <ptr>)

Overview:
"""""""""

The '``llvm.launder.invariant.group``' intrinsic can be used when an invariant
established by ``invariant.group`` metadata no longer holds, to obtain a new
pointer value that carries fresh invariant group information. It is an
experimental intrinsic, which means that its semantics might change in the
future.


Arguments:
""""""""""

The ``llvm.launder.invariant.group`` takes only one argument, which is a pointer
to the memory.

Semantics:
""""""""""

Returns another pointer that aliases its argument but which is considered different
for the purposes of ``load``/``store`` ``invariant.group`` metadata.
It does not read any accessible memory and the execution can be speculated.

'``llvm.strip.invariant.group``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
This is an overloaded intrinsic. The :ref:`allocated object<allocatedobjects>`
can belong to any address space. The returned pointer must belong to the same
address space as the argument.

::

      declare ptr @llvm.strip.invariant.group.p0(ptr <ptr>)

Overview:
"""""""""

The '``llvm.strip.invariant.group``' intrinsic can be used when an invariant
established by ``invariant.group`` metadata no longer holds, to obtain a new pointer
value that does not carry the invariant information. It is an experimental
intrinsic, which means that its semantics might change in the future.


Arguments:
""""""""""

The ``llvm.strip.invariant.group`` takes only one argument, which is a pointer
to the memory.

Semantics:
""""""""""

Returns another pointer that aliases its argument but which has no associated
``invariant.group`` metadata.
It does not read any memory and can be speculated.



.. _constrainedfp:

Constrained Floating-Point Intrinsics
-------------------------------------

These intrinsics are used to provide special handling of floating-point
operations when specific rounding mode or floating-point exception behavior is
required.  By default, LLVM optimization passes assume that the rounding mode is
round-to-nearest and that floating-point exceptions will not be monitored.
Constrained FP intrinsics are used to support non-default rounding modes and
accurately preserve exception behavior without compromising LLVM's ability to
optimize FP code when the default behavior is used.

If any FP operation in a function is constrained then they all must be
constrained. This is required for correct LLVM IR. Optimizations that
move code around can create miscompiles if mixing of constrained and normal
operations is done. The correct way to mix constrained and less constrained
operations is to use the rounding mode and exception handling metadata to
mark constrained intrinsics as having LLVM's default behavior.

Each of these intrinsics corresponds to a normal floating-point operation. The
data arguments and the return value are the same as the corresponding FP
operation.

The rounding mode argument is a metadata string specifying what
assumptions, if any, the optimizer can make when transforming constant
values. Some constrained FP intrinsics omit this argument. If required
by the intrinsic, this argument must be one of the following strings:

::

      "round.dynamic"
      "round.tonearest"
      "round.downward"
      "round.upward"
      "round.towardzero"
      "round.tonearestaway"

If this argument is "round.dynamic" optimization passes must assume that the
rounding mode is unknown and may change at runtime.  No transformations that
depend on rounding mode may be performed in this case.

The other possible values for the rounding mode argument correspond to the
similarly named IEEE rounding modes.  If the argument is any of these values
optimization passes may perform transformations as long as they are consistent
with the specified rounding mode.

For example, 'x-0'->'x' is not a valid transformation if the rounding mode is
"round.downward" or "round.dynamic" because if the value of 'x' is +0 then
'x-0' should evaluate to '-0' when rounding downward.  However, this
transformation is legal for all other rounding modes.

For values other than "round.dynamic" optimization passes may assume that the
actual runtime rounding mode (as defined in a target-specific manner) matches
the specified rounding mode, but this is not guaranteed.  Using a specific
non-dynamic rounding mode which does not match the actual rounding mode at
runtime results in undefined behavior.

The exception behavior argument is a metadata string describing the floating
point exception semantics that required for the intrinsic. This argument
must be one of the following strings:

::

      "fpexcept.ignore"
      "fpexcept.maytrap"
      "fpexcept.strict"

If this argument is "fpexcept.ignore" optimization passes may assume that the
exception status flags will not be read and that floating-point exceptions will
be masked.  This allows transformations to be performed that may change the
exception semantics of the original code.  For example, FP operations may be
speculatively executed in this case whereas they must not be for either of the
other possible values of this argument.

If the exception behavior argument is "fpexcept.maytrap" optimization passes
must avoid transformations that may raise exceptions that would not have been
raised by the original code (such as speculatively executing FP operations), but
passes are not required to preserve all exceptions that are implied by the
original code.  For example, exceptions may be potentially hidden by constant
folding.

If the exception behavior argument is "fpexcept.strict" all transformations must
strictly preserve the floating-point exception semantics of the original code.
Any FP exception that would have been raised by the original code must be raised
by the transformed code, and the transformed code must not raise any FP
exceptions that would not have been raised by the original code.  This is the
exception behavior argument that will be used if the code being compiled reads
the FP exception status flags, but this mode can also be used with code that
unmasks FP exceptions.

The number and order of floating-point exceptions is NOT guaranteed.  For
example, a series of FP operations that each may raise exceptions may be
vectorized into a single instruction that raises each unique exception a single
time.

Proper :ref:`function attributes <fnattrs>` usage is required for the
constrained intrinsics to function correctly.

All function *calls* done in a function that uses constrained floating
point intrinsics must have the ``strictfp`` attribute either on the
calling instruction or on the declaration or definition of the function
being called.

All function *definitions* that use constrained floating point intrinsics
must have the ``strictfp`` attribute.

'``llvm.experimental.constrained.fadd``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.fadd(<type> <op1>, <type> <op2>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.fadd``' intrinsic returns the sum of its
two arguments.


Arguments:
""""""""""

The first two arguments to the '``llvm.experimental.constrained.fadd``'
intrinsic must be :ref:`floating-point <t_floating>` or :ref:`vector <t_vector>`
of floating-point values. Both arguments must have identical types.

The third and fourth arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

The value produced is the floating-point sum of the two value arguments and has
the same type as the arguments.


'``llvm.experimental.constrained.fsub``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.fsub(<type> <op1>, <type> <op2>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.fsub``' intrinsic returns the difference
of its two arguments.


Arguments:
""""""""""

The first two arguments to the '``llvm.experimental.constrained.fsub``'
intrinsic must be :ref:`floating-point <t_floating>` or :ref:`vector <t_vector>`
of floating-point values. Both arguments must have identical types.

The third and fourth arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

The value produced is the floating-point difference of the two value arguments
and has the same type as the arguments.


'``llvm.experimental.constrained.fmul``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.fmul(<type> <op1>, <type> <op2>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.fmul``' intrinsic returns the product of
its two arguments.


Arguments:
""""""""""

The first two arguments to the '``llvm.experimental.constrained.fmul``'
intrinsic must be :ref:`floating-point <t_floating>` or :ref:`vector <t_vector>`
of floating-point values. Both arguments must have identical types.

The third and fourth arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

The value produced is the floating-point product of the two value arguments and
has the same type as the arguments.


'``llvm.experimental.constrained.fdiv``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.fdiv(<type> <op1>, <type> <op2>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.fdiv``' intrinsic returns the quotient of
its two arguments.


Arguments:
""""""""""

The first two arguments to the '``llvm.experimental.constrained.fdiv``'
intrinsic must be :ref:`floating-point <t_floating>` or :ref:`vector <t_vector>`
of floating-point values. Both arguments must have identical types.

The third and fourth arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

The value produced is the floating-point quotient of the two value arguments and
has the same type as the arguments.


'``llvm.experimental.constrained.frem``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.frem(<type> <op1>, <type> <op2>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.frem``' intrinsic returns the remainder
from the division of its two arguments.


Arguments:
""""""""""

The first two arguments to the '``llvm.experimental.constrained.frem``'
intrinsic must be :ref:`floating-point <t_floating>` or :ref:`vector <t_vector>`
of floating-point values. Both arguments must have identical types.

The third and fourth arguments specify the rounding mode and exception
behavior as described above.  The rounding mode argument has no effect, since
the result of frem is never rounded, but the argument is included for
consistency with the other constrained floating-point intrinsics.

Semantics:
""""""""""

The value produced is the floating-point remainder from the division of the two
value arguments and has the same type as the arguments.  The remainder has the
same sign as the dividend.

'``llvm.experimental.constrained.fma``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.fma(<type> <op1>, <type> <op2>, <type> <op3>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.fma``' intrinsic returns the result of a
fused-multiply-add operation on its arguments.

Arguments:
""""""""""

The first three arguments to the '``llvm.experimental.constrained.fma``'
intrinsic must be :ref:`floating-point <t_floating>` or :ref:`vector
<t_vector>` of floating-point values. All arguments must have identical types.

The fourth and fifth arguments specify the rounding mode and exception behavior
as described above.

Semantics:
""""""""""

The result produced is the product of the first two arguments added to the third
argument computed with infinite precision, and then rounded to the target
precision.

'``llvm.experimental.constrained.fptoui``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <ty2>
      @llvm.experimental.constrained.fptoui(<type> <value>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.fptoui``' intrinsic converts a
floating-point ``value`` to its unsigned integer equivalent of type ``ty2``.

Arguments:
""""""""""

The first argument to the '``llvm.experimental.constrained.fptoui``'
intrinsic must be :ref:`floating point <t_floating>` or :ref:`vector
<t_vector>` of floating point values.

The second argument specifies the exception behavior as described above.

Semantics:
""""""""""

The result produced is an unsigned integer converted from the floating
point argument. The value is truncated, so it is rounded towards zero.

'``llvm.experimental.constrained.fptosi``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <ty2>
      @llvm.experimental.constrained.fptosi(<type> <value>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.fptosi``' intrinsic converts
:ref:`floating-point <t_floating>` ``value`` to type ``ty2``.

Arguments:
""""""""""

The first argument to the '``llvm.experimental.constrained.fptosi``'
intrinsic must be :ref:`floating point <t_floating>` or :ref:`vector
<t_vector>` of floating point values.

The second argument specifies the exception behavior as described above.

Semantics:
""""""""""

The result produced is a signed integer converted from the floating
point argument. The value is truncated, so it is rounded towards zero.

'``llvm.experimental.constrained.uitofp``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <ty2>
      @llvm.experimental.constrained.uitofp(<type> <value>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.uitofp``' intrinsic converts an
unsigned integer ``value`` to a floating-point of type ``ty2``.

Arguments:
""""""""""

The first argument to the '``llvm.experimental.constrained.uitofp``'
intrinsic must be an :ref:`integer <t_integer>` or :ref:`vector
<t_vector>` of integer values.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

An inexact floating-point exception will be raised if rounding is required.
Any result produced is a floating point value converted from the input
integer argument.

'``llvm.experimental.constrained.sitofp``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <ty2>
      @llvm.experimental.constrained.sitofp(<type> <value>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.sitofp``' intrinsic converts a
signed integer ``value`` to a floating-point of type ``ty2``.

Arguments:
""""""""""

The first argument to the '``llvm.experimental.constrained.sitofp``'
intrinsic must be an :ref:`integer <t_integer>` or :ref:`vector
<t_vector>` of integer values.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

An inexact floating-point exception will be raised if rounding is required.
Any result produced is a floating point value converted from the input
integer argument.

'``llvm.experimental.constrained.fptrunc``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <ty2>
      @llvm.experimental.constrained.fptrunc(<type> <value>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.fptrunc``' intrinsic truncates ``value``
to type ``ty2``.

Arguments:
""""""""""

The first argument to the '``llvm.experimental.constrained.fptrunc``'
intrinsic must be :ref:`floating point <t_floating>` or :ref:`vector
<t_vector>` of floating point values. This argument must be larger in size
than the result.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

The result produced is a floating point value truncated to be smaller in size
than the argument.

'``llvm.experimental.constrained.fpext``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <ty2>
      @llvm.experimental.constrained.fpext(<type> <value>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.fpext``' intrinsic extends a
floating-point ``value`` to a larger floating-point value.

Arguments:
""""""""""

The first argument to the '``llvm.experimental.constrained.fpext``'
intrinsic must be :ref:`floating point <t_floating>` or :ref:`vector
<t_vector>` of floating point values. This argument must be smaller in size
than the result.

The second argument specifies the exception behavior as described above.

Semantics:
""""""""""

The result produced is a floating point value extended to be larger in size
than the argument. All restrictions that apply to the fpext instruction also
apply to this intrinsic.

'``llvm.experimental.constrained.fcmp``' and '``llvm.experimental.constrained.fcmps``' Intrinsics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <ty2>
      @llvm.experimental.constrained.fcmp(<type> <op1>, <type> <op2>,
                                          metadata <condition code>,
                                          metadata <exception behavior>)
      declare <ty2>
      @llvm.experimental.constrained.fcmps(<type> <op1>, <type> <op2>,
                                           metadata <condition code>,
                                           metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.fcmp``' and
'``llvm.experimental.constrained.fcmps``' intrinsics return a boolean
value or vector of boolean values based on comparison of its arguments.

If the arguments are floating-point scalars, then the result type is a
boolean (:ref:`i1 <t_integer>`).

If the arguments are floating-point vectors, then the result type is a
vector of boolean with the same number of elements as the arguments being
compared.

The '``llvm.experimental.constrained.fcmp``' intrinsic performs a quiet
comparison operation while the '``llvm.experimental.constrained.fcmps``'
intrinsic performs a signaling comparison operation.

Arguments:
""""""""""

The first two arguments to the '``llvm.experimental.constrained.fcmp``'
and '``llvm.experimental.constrained.fcmps``' intrinsics must be
:ref:`floating-point <t_floating>` or :ref:`vector <t_vector>`
of floating-point values. Both arguments must have identical types.

The third argument is the condition code indicating the kind of comparison
to perform. It must be a metadata string with one of the following values:

.. _fcmp_md_cc:

- "``oeq``": ordered and equal
- "``ogt``": ordered and greater than
- "``oge``": ordered and greater than or equal
- "``olt``": ordered and less than
- "``ole``": ordered and less than or equal
- "``one``": ordered and not equal
- "``ord``": ordered (no nans)
- "``ueq``": unordered or equal
- "``ugt``": unordered or greater than
- "``uge``": unordered or greater than or equal
- "``ult``": unordered or less than
- "``ule``": unordered or less than or equal
- "``une``": unordered or not equal
- "``uno``": unordered (either nans)

*Ordered* means that neither argument is a NAN while *unordered* means
that either argument may be a NAN.

The fourth argument specifies the exception behavior as described above.

Semantics:
""""""""""

``op1`` and ``op2`` are compared according to the condition code given
as the third argument. If the arguments are vectors, then the
vectors are compared element by element. Each comparison performed
always yields an :ref:`i1 <t_integer>` result, as follows:

.. _fcmp_md_cc_sem:

- "``oeq``": yields ``true`` if both arguments are not a NAN and ``op1``
  is equal to ``op2``.
- "``ogt``": yields ``true`` if both arguments are not a NAN and ``op1``
  is greater than ``op2``.
- "``oge``": yields ``true`` if both arguments are not a NAN and ``op1``
  is greater than or equal to ``op2``.
- "``olt``": yields ``true`` if both arguments are not a NAN and ``op1``
  is less than ``op2``.
- "``ole``": yields ``true`` if both arguments are not a NAN and ``op1``
  is less than or equal to ``op2``.
- "``one``": yields ``true`` if both arguments are not a NAN and ``op1``
  is not equal to ``op2``.
- "``ord``": yields ``true`` if both arguments are not a NAN.
- "``ueq``": yields ``true`` if either argument is a NAN or ``op1`` is
  equal to ``op2``.
- "``ugt``": yields ``true`` if either argument is a NAN or ``op1`` is
  greater than ``op2``.
- "``uge``": yields ``true`` if either argument is a NAN or ``op1`` is
  greater than or equal to ``op2``.
- "``ult``": yields ``true`` if either argument is a NAN or ``op1`` is
  less than ``op2``.
- "``ule``": yields ``true`` if either argument is a NAN or ``op1`` is
  less than or equal to ``op2``.
- "``une``": yields ``true`` if either argument is a NAN or ``op1`` is
  not equal to ``op2``.
- "``uno``": yields ``true`` if either argument is a NAN.

The quiet comparison operation performed by
'``llvm.experimental.constrained.fcmp``' will only raise an exception
if either argument is a SNAN.  The signaling comparison operation
performed by '``llvm.experimental.constrained.fcmps``' will raise an
exception if either argument is a NAN (QNAN or SNAN). Such an exception
does not preclude a result being produced (e.g. exception might only
set a flag), therefore the distinction between ordered and unordered
comparisons is also relevant for the
'``llvm.experimental.constrained.fcmps``' intrinsic.

'``llvm.experimental.constrained.fmuladd``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.fmuladd(<type> <op1>, <type> <op2>,
                                             <type> <op3>,
                                             metadata <rounding mode>,
                                             metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.fmuladd``' intrinsic represents
multiply-add expressions that can be fused if the code generator determines
that (a) the target instruction set has support for a fused operation,
and (b) that the fused operation is more efficient than the equivalent,
separate pair of mul and add instructions.

Arguments:
""""""""""

The first three arguments to the '``llvm.experimental.constrained.fmuladd``'
intrinsic must be floating-point or vector of floating-point values.
All three arguments must have identical types.

The fourth and fifth arguments specify the rounding mode and exception behavior
as described above.

Semantics:
""""""""""

The expression:

::

      %0 = call float @llvm.experimental.constrained.fmuladd.f32(%a, %b, %c,
                                                                 metadata <rounding mode>,
                                                                 metadata <exception behavior>)

is equivalent to the expression:

::

      %0 = call float @llvm.experimental.constrained.fmul.f32(%a, %b,
                                                              metadata <rounding mode>,
                                                              metadata <exception behavior>)
      %1 = call float @llvm.experimental.constrained.fadd.f32(%0, %c,
                                                              metadata <rounding mode>,
                                                              metadata <exception behavior>)

except that it is unspecified whether rounding will be performed between the
multiplication and addition steps. Fusion is not guaranteed, even if the target
platform supports it.
If a fused multiply-add is required, the corresponding
:ref:`llvm.experimental.constrained.fma <int_fma>` intrinsic function should be
used instead.
This never sets errno, just as '``llvm.experimental.constrained.fma.*``'.

Constrained libm-equivalent Intrinsics
--------------------------------------

In addition to the basic floating-point operations for which constrained
intrinsics are described above, there are constrained versions of various
operations which provide equivalent behavior to a corresponding libm function.
These intrinsics allow the precise behavior of these operations with respect to
rounding mode and exception behavior to be controlled.

As with the basic constrained floating-point intrinsics, the rounding mode
and exception behavior arguments only control the behavior of the optimizer.
They do not change the runtime floating-point environment.


'``llvm.experimental.constrained.sqrt``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.sqrt(<type> <op1>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.sqrt``' intrinsic returns the square root
of the specified value, returning the same value as the libm '``sqrt``'
functions would, but without setting ``errno``.

Arguments:
""""""""""

The first argument and the return type are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the nonnegative square root of the specified value.
If the value is less than negative zero, a floating-point exception occurs
and the return value is architecture specific.


'``llvm.experimental.constrained.pow``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.pow(<type> <op1>, <type> <op2>,
                                         metadata <rounding mode>,
                                         metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.pow``' intrinsic returns the first argument
raised to the (positive or negative) power specified by the second argument.

Arguments:
""""""""""

The first two arguments and the return value are floating-point numbers of the
same type.  The second argument specifies the power to which the first argument
should be raised.

The third and fourth arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the first value raised to the second power,
returning the same values as the libm ``pow`` functions would, and
handles error conditions in the same way.


'``llvm.experimental.constrained.powi``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.powi(<type> <op1>, i32 <op2>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.powi``' intrinsic returns the first argument
raised to the (positive or negative) power specified by the second argument. The
order of evaluation of multiplications is not defined. When a vector of
floating-point type is used, the second argument remains a scalar integer value.


Arguments:
""""""""""

The first argument and the return value are floating-point numbers of the same
type.  The second argument is a 32-bit signed integer specifying the power to
which the first argument should be raised.

The third and fourth arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the first value raised to the second power with an
unspecified sequence of rounding operations.


'``llvm.experimental.constrained.ldexp``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type0>
      @llvm.experimental.constrained.ldexp(<type0> <op1>, <type1> <op2>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.ldexp``' performs the ldexp function.


Arguments:
""""""""""

The first argument and the return value are :ref:`floating-point
<t_floating>` or :ref:`vector <t_vector>` of floating-point values of
the same type. The second argument is an integer with the same number
of elements.


The third and fourth arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function multiplies the first argument by 2 raised to the second
argument's power. If the first argument is NaN or infinite, the same
value is returned. If the result underflows a zero with the same sign
is returned. If the result overflows, the result is an infinity with
the same sign.


'``llvm.experimental.constrained.sin``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.sin(<type> <op1>,
                                         metadata <rounding mode>,
                                         metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.sin``' intrinsic returns the sine of the
first argument.

Arguments:
""""""""""

The first argument and the return type are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the sine of the specified argument, returning the
same values as the libm ``sin`` functions would, and handles error
conditions in the same way.


'``llvm.experimental.constrained.cos``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.cos(<type> <op1>,
                                         metadata <rounding mode>,
                                         metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.cos``' intrinsic returns the cosine of the
first argument.

Arguments:
""""""""""

The first argument and the return type are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the cosine of the specified argument, returning the
same values as the libm ``cos`` functions would, and handles error
conditions in the same way.


'``llvm.experimental.constrained.tan``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.tan(<type> <op1>,
                                         metadata <rounding mode>,
                                         metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.tan``' intrinsic returns the tangent of the
first argument.

Arguments:
""""""""""

The first argument and the return type are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the tangent of the specified argument, returning the
same values as the libm ``tan`` functions would, and handles error
conditions in the same way.

'``llvm.experimental.constrained.asin``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.asin(<type> <op1>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.asin``' intrinsic returns the arcsine of the
first operand.

Arguments:
""""""""""

The first argument and the return type are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the arcsine of the specified operand, returning the
same values as the libm ``asin`` functions would, and handles error
conditions in the same way.


'``llvm.experimental.constrained.acos``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.acos(<type> <op1>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.acos``' intrinsic returns the arccosine of the
first operand.

Arguments:
""""""""""

The first argument and the return type are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the arccosine of the specified operand, returning the
same values as the libm ``acos`` functions would, and handles error
conditions in the same way.


'``llvm.experimental.constrained.atan``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.atan(<type> <op1>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.atan``' intrinsic returns the arctangent of the
first operand.

Arguments:
""""""""""

The first argument and the return type are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the arctangent of the specified operand, returning the
same values as the libm ``atan`` functions would, and handles error
conditions in the same way.

'``llvm.experimental.constrained.atan2``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.atan2(<type> <op1>,
                                           <type> <op2>,
                                           metadata <rounding mode>,
                                           metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.atan2``' intrinsic returns the arctangent
of ``<op1>`` divided by ``<op2>`` accounting for the quadrant.

Arguments:
""""""""""

The first two arguments and the return value are floating-point numbers of the
same type.

The third and fourth arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the quadrant-specific arctangent using the specified
operands, returning the same values as the libm ``atan2`` functions would, and
handles error conditions in the same way.

'``llvm.experimental.constrained.sinh``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.sinh(<type> <op1>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.sinh``' intrinsic returns the hyperbolic sine of the
first operand.

Arguments:
""""""""""

The first argument and the return type are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the hyperbolic sine of the specified operand, returning the
same values as the libm ``sinh`` functions would, and handles error
conditions in the same way.


'``llvm.experimental.constrained.cosh``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.cosh(<type> <op1>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.cosh``' intrinsic returns the hyperbolic cosine of the
first operand.

Arguments:
""""""""""

The first argument and the return type are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the hyperbolic cosine of the specified operand, returning the
same values as the libm ``cosh`` functions would, and handles error
conditions in the same way.


'``llvm.experimental.constrained.tanh``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.tanh(<type> <op1>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.tanh``' intrinsic returns the hyperbolic tangent of the
first operand.

Arguments:
""""""""""

The first argument and the return type are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the hyperbolic tangent of the specified operand, returning the
same values as the libm ``tanh`` functions would, and handles error
conditions in the same way.

'``llvm.experimental.constrained.exp``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.exp(<type> <op1>,
                                         metadata <rounding mode>,
                                         metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.exp``' intrinsic computes the base-e
exponential of the specified value.

Arguments:
""""""""""

The first argument and the return value are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the same values as the libm ``exp`` functions
would, and handles error conditions in the same way.


'``llvm.experimental.constrained.exp2``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.exp2(<type> <op1>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.exp2``' intrinsic computes the base-2
exponential of the specified value.


Arguments:
""""""""""

The first argument and the return value are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the same values as the libm ``exp2`` functions
would, and handles error conditions in the same way.


'``llvm.experimental.constrained.log``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.log(<type> <op1>,
                                         metadata <rounding mode>,
                                         metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.log``' intrinsic computes the base-e
logarithm of the specified value.

Arguments:
""""""""""

The first argument and the return value are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.


Semantics:
""""""""""

This function returns the same values as the libm ``log`` functions
would, and handles error conditions in the same way.


'``llvm.experimental.constrained.log10``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.log10(<type> <op1>,
                                           metadata <rounding mode>,
                                           metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.log10``' intrinsic computes the base-10
logarithm of the specified value.

Arguments:
""""""""""

The first argument and the return value are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the same values as the libm ``log10`` functions
would, and handles error conditions in the same way.


'``llvm.experimental.constrained.log2``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.log2(<type> <op1>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.log2``' intrinsic computes the base-2
logarithm of the specified value.

Arguments:
""""""""""

The first argument and the return value are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the same values as the libm ``log2`` functions
would, and handles error conditions in the same way.


'``llvm.experimental.constrained.rint``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.rint(<type> <op1>,
                                          metadata <rounding mode>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.rint``' intrinsic returns the first
argument rounded to the nearest integer. It may raise an inexact floating-point
exception if the argument is not an integer.

Arguments:
""""""""""

The first argument and the return value are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the same values as the libm ``rint`` functions
would, and handles error conditions in the same way.  The rounding mode is
described, not determined, by the rounding mode argument.  The actual rounding
mode is determined by the runtime floating-point environment.  The rounding
mode argument is only intended as information to the compiler.


'``llvm.experimental.constrained.lrint``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <inttype>
      @llvm.experimental.constrained.lrint(<fptype> <op1>,
                                           metadata <rounding mode>,
                                           metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.lrint``' intrinsic returns the first
argument rounded to the nearest integer. An inexact floating-point exception
will be raised if the argument is not an integer. An invalid exception is
raised if the result is too large to fit into a supported integer type,
and in this case the result is undefined.

Arguments:
""""""""""

The first argument is a floating-point number. The return value is an
integer type. Not all types are supported on all targets. The supported
types are the same as the ``llvm.lrint`` intrinsic and the ``lrint``
libm functions.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the same values as the libm ``lrint`` functions
would, and handles error conditions in the same way.

The rounding mode is described, not determined, by the rounding mode
argument.  The actual rounding mode is determined by the runtime floating-point
environment.  The rounding mode argument is only intended as information
to the compiler.

If the runtime floating-point environment is using the default rounding mode
then the results will be the same as the llvm.lrint intrinsic.


'``llvm.experimental.constrained.llrint``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <inttype>
      @llvm.experimental.constrained.llrint(<fptype> <op1>,
                                            metadata <rounding mode>,
                                            metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.llrint``' intrinsic returns the first
argument rounded to the nearest integer. An inexact floating-point exception
will be raised if the argument is not an integer. An invalid exception is
raised if the result is too large to fit into a supported integer type,
and in this case the result is undefined.

Arguments:
""""""""""

The first argument is a floating-point number. The return value is an
integer type. Not all types are supported on all targets. The supported
types are the same as the ``llvm.llrint`` intrinsic and the ``llrint``
libm functions.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the same values as the libm ``llrint`` functions
would, and handles error conditions in the same way.

The rounding mode is described, not determined, by the rounding mode
argument.  The actual rounding mode is determined by the runtime floating-point
environment.  The rounding mode argument is only intended as information
to the compiler.

If the runtime floating-point environment is using the default rounding mode
then the results will be the same as the llvm.llrint intrinsic.


'``llvm.experimental.constrained.nearbyint``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.nearbyint(<type> <op1>,
                                               metadata <rounding mode>,
                                               metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.nearbyint``' intrinsic returns the first
argument rounded to the nearest integer. It will not raise an inexact
floating-point exception if the argument is not an integer.


Arguments:
""""""""""

The first argument and the return value are floating-point numbers of the same
type.

The second and third arguments specify the rounding mode and exception
behavior as described above.

Semantics:
""""""""""

This function returns the same values as the libm ``nearbyint`` functions
would, and handles error conditions in the same way.  The rounding mode is
described, not determined, by the rounding mode argument.  The actual rounding
mode is determined by the runtime floating-point environment.  The rounding
mode argument is only intended as information to the compiler.


'``llvm.experimental.constrained.maxnum``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.maxnum(<type> <op1>, <type> <op2>
                                            metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.maxnum``' intrinsic returns the maximum
of the two arguments.

Arguments:
""""""""""

The first two arguments and the return value are floating-point numbers
of the same type.

The third argument specifies the exception behavior as described above.

Semantics:
""""""""""

This function follows the IEEE-754-2008 semantics for maxNum.


'``llvm.experimental.constrained.minnum``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.minnum(<type> <op1>, <type> <op2>
                                            metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.minnum``' intrinsic returns the minimum
of the two arguments.

Arguments:
""""""""""

The first two arguments and the return value are floating-point numbers
of the same type.

The third argument specifies the exception behavior as described above.

Semantics:
""""""""""

This function follows the IEEE-754-2008 semantics for minNum.


'``llvm.experimental.constrained.maximum``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.maximum(<type> <op1>, <type> <op2>
                                             metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.maximum``' intrinsic returns the maximum
of the two arguments, propagating NaNs and treating -0.0 as less than +0.0.

Arguments:
""""""""""

The first two arguments and the return value are floating-point numbers
of the same type.

The third argument specifies the exception behavior as described above.

Semantics:
""""""""""

This function follows semantics specified in the draft of IEEE 754-2019.


'``llvm.experimental.constrained.minimum``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.minimum(<type> <op1>, <type> <op2>
                                             metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.minimum``' intrinsic returns the minimum
of the two arguments, propagating NaNs and treating -0.0 as less than +0.0.

Arguments:
""""""""""

The first two arguments and the return value are floating-point numbers
of the same type.

The third argument specifies the exception behavior as described above.

Semantics:
""""""""""

This function follows semantics specified in the draft of IEEE 754-2019.


'``llvm.experimental.constrained.ceil``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.ceil(<type> <op1>,
                                          metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.ceil``' intrinsic returns the ceiling of the
first argument.

Arguments:
""""""""""

The first argument and the return value are floating-point numbers of the same
type.

The second argument specifies the exception behavior as described above.

Semantics:
""""""""""

This function returns the same values as the libm ``ceil`` functions
would and handles error conditions in the same way.


'``llvm.experimental.constrained.floor``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.floor(<type> <op1>,
                                           metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.floor``' intrinsic returns the floor of the
first argument.

Arguments:
""""""""""

The first argument and the return value are floating-point numbers of the same
type.

The second argument specifies the exception behavior as described above.

Semantics:
""""""""""

This function returns the same values as the libm ``floor`` functions
would and handles error conditions in the same way.


'``llvm.experimental.constrained.round``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.round(<type> <op1>,
                                           metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.round``' intrinsic returns the first
argument rounded to the nearest integer.

Arguments:
""""""""""

The first argument and the return value are floating-point numbers of the same
type.

The second argument specifies the exception behavior as described above.

Semantics:
""""""""""

This function returns the same values as the libm ``round`` functions
would and handles error conditions in the same way.


'``llvm.experimental.constrained.roundeven``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.roundeven(<type> <op1>,
                                               metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.roundeven``' intrinsic returns the first
argument rounded to the nearest integer in floating-point format, rounding
halfway cases to even (that is, to the nearest value that is an even integer),
regardless of the current rounding direction.

Arguments:
""""""""""

The first argument and the return value are floating-point numbers of the same
type.

The second argument specifies the exception behavior as described above.

Semantics:
""""""""""

This function implements IEEE-754 operation ``roundToIntegralTiesToEven``. It
also behaves in the same way as C standard function ``roundeven`` and can signal
the invalid operation exception for a SNAN argument.


'``llvm.experimental.constrained.lround``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <inttype>
      @llvm.experimental.constrained.lround(<fptype> <op1>,
                                            metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.lround``' intrinsic returns the first
argument rounded to the nearest integer with ties away from zero.  It will
raise an inexact floating-point exception if the argument is not an integer.
An invalid exception is raised if the result is too large to fit into a
supported integer type, and in this case the result is undefined.

Arguments:
""""""""""

The first argument is a floating-point number. The return value is an
integer type. Not all types are supported on all targets. The supported
types are the same as the ``llvm.lround`` intrinsic and the ``lround``
libm functions.

The second argument specifies the exception behavior as described above.

Semantics:
""""""""""

This function returns the same values as the libm ``lround`` functions
would and handles error conditions in the same way.


'``llvm.experimental.constrained.llround``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <inttype>
      @llvm.experimental.constrained.llround(<fptype> <op1>,
                                             metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.llround``' intrinsic returns the first
argument rounded to the nearest integer with ties away from zero. It will
raise an inexact floating-point exception if the argument is not an integer.
An invalid exception is raised if the result is too large to fit into a
supported integer type, and in this case the result is undefined.

Arguments:
""""""""""

The first argument is a floating-point number. The return value is an
integer type. Not all types are supported on all targets. The supported
types are the same as the ``llvm.llround`` intrinsic and the ``llround``
libm functions.

The second argument specifies the exception behavior as described above.

Semantics:
""""""""""

This function returns the same values as the libm ``llround`` functions
would and handles error conditions in the same way.


'``llvm.experimental.constrained.trunc``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.experimental.constrained.trunc(<type> <op1>,
                                           metadata <exception behavior>)

Overview:
"""""""""

The '``llvm.experimental.constrained.trunc``' intrinsic returns the first
argument rounded to the nearest integer not larger in magnitude than the
argument.

Arguments:
""""""""""

The first argument and the return value are floating-point numbers of the same
type.

The second argument specifies the exception behavior as described above.

Semantics:
""""""""""

This function returns the same values as the libm ``trunc`` functions
would and handles error conditions in the same way.

.. _int_experimental_noalias_scope_decl:

'``llvm.experimental.noalias.scope.decl``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""


::

      declare void @llvm.experimental.noalias.scope.decl(metadata !id.scope.list)

Overview:
"""""""""

The ``llvm.experimental.noalias.scope.decl`` intrinsic identifies where a
noalias scope is declared. When the intrinsic is duplicated, a decision must
also be made about the scope: depending on the reason of the duplication,
the scope might need to be duplicated as well.


Arguments:
""""""""""

The ``!id.scope.list`` argument is metadata that is a list of ``noalias``
metadata references. The format is identical to that required for ``noalias``
metadata. This list must have exactly one element.

Semantics:
""""""""""

The ``llvm.experimental.noalias.scope.decl`` intrinsic identifies where a
noalias scope is declared. When the intrinsic is duplicated, a decision must
also be made about the scope: depending on the reason of the duplication,
the scope might need to be duplicated as well.

For example, when the intrinsic is used inside a loop body, and that loop is
unrolled, the associated noalias scope must also be duplicated. Otherwise, the
noalias property it signifies would spill across loop iterations, whereas it
was only valid within a single iteration.

.. code-block:: llvm

  ; This examples shows two possible positions for noalias.decl and how they impact the semantics:
  ; If it is outside the loop (Version 1), then %a and %b are noalias across *all* iterations.
  ; If it is inside the loop (Version 2), then %a and %b are noalias only within *one* iteration.
  declare void @decl_in_loop(ptr %a.base, ptr %b.base) {
  entry:
    ; call void @llvm.experimental.noalias.scope.decl(metadata !2) ; Version 1: noalias decl outside loop
    br label %loop

  loop:
    %a = phi ptr [ %a.base, %entry ], [ %a.inc, %loop ]
    %b = phi ptr [ %b.base, %entry ], [ %b.inc, %loop ]
    ; call void @llvm.experimental.noalias.scope.decl(metadata !2) ; Version 2: noalias decl inside loop
    %val = load i8, ptr %a, !alias.scope !2
    store i8 %val, ptr %b, !noalias !2
    %a.inc = getelementptr inbounds i8, ptr %a, i64 1
    %b.inc = getelementptr inbounds i8, ptr %b, i64 1
    %cond = call i1 @cond()
    br i1 %cond, label %loop, label %exit

  exit:
    ret void
  }

  !0 = !{!0} ; domain
  !1 = !{!1, !0} ; scope
  !2 = !{!1} ; scope list

Multiple calls to `@llvm.experimental.noalias.scope.decl` for the same scope
are possible, but one should never dominate another. Violations are pointed out
by the verifier as they indicate a problem in either a transformation pass or
the input.


Floating Point Environment Manipulation intrinsics
--------------------------------------------------

These functions read or write floating point environment, such as rounding
mode or state of floating point exceptions. Altering the floating point
environment requires special care. See :ref:`Floating Point Environment <floatenv>`.

.. _int_get_rounding:

'``llvm.get.rounding``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i32 @llvm.get.rounding()

Overview:
"""""""""

The '``llvm.get.rounding``' intrinsic reads the current rounding mode.

Semantics:
""""""""""

The '``llvm.get.rounding``' intrinsic returns the current rounding mode.
Encoding of the returned values is same as the result of ``FLT_ROUNDS``,
specified by C standard:

::

    0  - toward zero
    1  - to nearest, ties to even
    2  - toward positive infinity
    3  - toward negative infinity
    4  - to nearest, ties away from zero

Other values may be used to represent additional rounding modes, supported by a
target. These values are target-specific.

.. _int_set_rounding:

'``llvm.set.rounding``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.set.rounding(i32 <val>)

Overview:
"""""""""

The '``llvm.set.rounding``' intrinsic sets current rounding mode.

Arguments:
""""""""""

The argument is the required rounding mode. Encoding of rounding mode is
the same as used by '``llvm.get.rounding``'.

Semantics:
""""""""""

The '``llvm.set.rounding``' intrinsic sets the current rounding mode. It is
similar to C library function 'fesetround', however this intrinsic does not
return any value and uses platform-independent representation of IEEE rounding
modes.

.. _int_get_fpenv:

'``llvm.get.fpenv``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <integer_type> @llvm.get.fpenv()

Overview:
"""""""""

The '``llvm.get.fpenv``' intrinsic returns bits of the current floating-point
environment. The return value type is platform-specific.

Semantics:
""""""""""

The '``llvm.get.fpenv``' intrinsic reads the current floating-point environment
and returns it as an integer value.

.. _int_set_fpenv:

'``llvm.set.fpenv``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.set.fpenv(<integer_type> <val>)

Overview:
"""""""""

The '``llvm.set.fpenv``' intrinsic sets the current floating-point environment.

Arguments:
""""""""""

The argument is an integer representing the new floating-point environment. The
integer type is platform-specific.

Semantics:
""""""""""

The '``llvm.set.fpenv``' intrinsic sets the current floating-point environment
to the state specified by the argument. The state may be previously obtained by a
call to '``llvm.get.fpenv``' or synthesized in a platform-dependent way.


'``llvm.reset.fpenv``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.reset.fpenv()

Overview:
"""""""""

The '``llvm.reset.fpenv``' intrinsic sets the default floating-point environment.

Semantics:
""""""""""

The '``llvm.reset.fpenv``' intrinsic sets the current floating-point environment
to default state. It is similar to the call 'fesetenv(FE_DFL_ENV)', except it
does not return any value.

.. _int_get_fpmode:

'``llvm.get.fpmode``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

The '``llvm.get.fpmode``' intrinsic returns bits of the current floating-point
control modes. The return value type is platform-specific.

::

      declare <integer_type> @llvm.get.fpmode()

Overview:
"""""""""

The '``llvm.get.fpmode``' intrinsic reads the current dynamic floating-point
control modes and returns it as an integer value.

Arguments:
""""""""""

None.

Semantics:
""""""""""

The '``llvm.get.fpmode``' intrinsic reads the current dynamic floating-point
control modes, such as rounding direction, precision, treatment of denormals and
so on. It is similar to the C library function 'fegetmode', however this
function does not store the set of control modes into memory but returns it as
an integer value. Interpretation of the bits in this value is target-dependent.

'``llvm.set.fpmode``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

The '``llvm.set.fpmode``' intrinsic sets the current floating-point control modes.

::

      declare void @llvm.set.fpmode(<integer_type> <val>)

Overview:
"""""""""

The '``llvm.set.fpmode``' intrinsic sets the current dynamic floating-point
control modes.

Arguments:
""""""""""

The argument is a set of floating-point control modes, represented as an integer
value in a target-dependent way.

Semantics:
""""""""""

The '``llvm.set.fpmode``' intrinsic sets the current dynamic floating-point
control modes to the state specified by the argument, which must be obtained by
a call to '``llvm.get.fpmode``' or constructed in a target-specific way. It is
similar to the C library function 'fesetmode', however this function does not
read the set of control modes from memory but gets it as integer value.

'``llvm.reset.fpmode``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.reset.fpmode()

Overview:
"""""""""

The '``llvm.reset.fpmode``' intrinsic sets the default dynamic floating-point
control modes.

Arguments:
""""""""""

None.

Semantics:
""""""""""

The '``llvm.reset.fpmode``' intrinsic sets the current dynamic floating-point
environment to default state. It is similar to the C library function call
'fesetmode(FE_DFL_MODE)', however this function does not return any value.


Floating-Point Test Intrinsics
------------------------------

These functions get properties of floating-point values.


.. _llvm.is.fpclass:

'``llvm.is.fpclass``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i1 @llvm.is.fpclass(<fptype> <op>, i32 <test>)
      declare <N x i1> @llvm.is.fpclass(<vector-fptype> <op>, i32 <test>)

Overview:
"""""""""

The '``llvm.is.fpclass``' intrinsic returns a boolean value or vector of boolean
values depending on whether the first argument satisfies the test specified by
the second argument.

If the first argument is a floating-point scalar, then the result type is a
boolean (:ref:`i1 <t_integer>`).

If the first argument is a floating-point vector, then the result type is a
vector of boolean with the same number of elements as the first argument.

Arguments:
""""""""""

The first argument to the '``llvm.is.fpclass``' intrinsic must be
:ref:`floating-point <t_floating>` or :ref:`vector <t_vector>`
of floating-point values.

The second argument specifies, which tests to perform. It must be a compile-time
integer constant, each bit in which specifies floating-point class:

+-------+----------------------+
| Bit # | floating-point class |
+=======+======================+
| 0     | Signaling NaN        |
+-------+----------------------+
| 1     | Quiet NaN            |
+-------+----------------------+
| 2     | Negative infinity    |
+-------+----------------------+
| 3     | Negative normal      |
+-------+----------------------+
| 4     | Negative subnormal   |
+-------+----------------------+
| 5     | Negative zero        |
+-------+----------------------+
| 6     | Positive zero        |
+-------+----------------------+
| 7     | Positive subnormal   |
+-------+----------------------+
| 8     | Positive normal      |
+-------+----------------------+
| 9     | Positive infinity    |
+-------+----------------------+

Semantics:
""""""""""

The function checks if ``op`` belongs to any of the floating-point classes
specified by ``test``. If ``op`` is a vector, then the check is made element by
element. Each check yields an :ref:`i1 <t_integer>` result, which is ``true``,
if the element value satisfies the specified test. The argument ``test`` is a
bit mask where each bit specifies floating-point class to test. For example, the
value 0x108 makes test for normal value, - bits 3 and 8 in it are set, which
means that the function returns ``true`` if ``op`` is a positive or negative
normal value. The function never raises floating-point exceptions. The
function does not canonicalize its input value and does not depend
on the floating-point environment. If the floating-point environment
has a zeroing treatment of subnormal input values (such as indicated
by the ``"denormal-fp-math"`` attribute), a subnormal value will be
observed (will not be implicitly treated as zero).


General Intrinsics
------------------

This class of intrinsics is designed to be generic and has no specific
purpose.

'``llvm.var.annotation``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.var.annotation(ptr <val>, ptr <str>, ptr <str>, i32  <int>)

Overview:
"""""""""

The '``llvm.var.annotation``' intrinsic.

Arguments:
""""""""""

The first argument is a pointer to a value, the second is a pointer to a
global string, the third is a pointer to a global string which is the
source file name, and the last argument is the line number.

Semantics:
""""""""""

This intrinsic allows annotation of local variables with arbitrary
strings. This can be useful for special purpose optimizations that want
to look for these annotations. These have no other defined use; they are
ignored by code generation and optimization.

'``llvm.ptr.annotation.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use '``llvm.ptr.annotation``' on a
pointer to an integer of any width. *NOTE* you must specify an address space for
the pointer. The identifier for the default address space is the integer
'``0``'.

::

      declare ptr @llvm.ptr.annotation.p0(ptr <val>, ptr <str>, ptr <str>, i32 <int>)
      declare ptr @llvm.ptr.annotation.p1(ptr addrspace(1) <val>, ptr <str>, ptr <str>, i32 <int>)

Overview:
"""""""""

The '``llvm.ptr.annotation``' intrinsic.

Arguments:
""""""""""

The first argument is a pointer to an integer value of arbitrary bitwidth
(result of some expression), the second is a pointer to a global string, the
third is a pointer to a global string which is the source file name, and the
last argument is the line number. It returns the value of the first argument.

Semantics:
""""""""""

This intrinsic allows annotation of a pointer to an integer with arbitrary
strings. This can be useful for special purpose optimizations that want to look
for these annotations. These have no other defined use; transformations preserve
annotations on a best-effort basis but are allowed to replace the intrinsic with
its first argument without breaking semantics and the intrinsic is completely
dropped during instruction selection.

'``llvm.annotation.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use '``llvm.annotation``' on
any integer bit width.

::

      declare i8 @llvm.annotation.i8(i8 <val>, ptr <str>, ptr <str>, i32  <int>)
      declare i16 @llvm.annotation.i16(i16 <val>, ptr <str>, ptr <str>, i32  <int>)
      declare i32 @llvm.annotation.i32(i32 <val>, ptr <str>, ptr <str>, i32  <int>)
      declare i64 @llvm.annotation.i64(i64 <val>, ptr <str>, ptr <str>, i32  <int>)
      declare i256 @llvm.annotation.i256(i256 <val>, ptr <str>, ptr <str>, i32  <int>)

Overview:
"""""""""

The '``llvm.annotation``' intrinsic.

Arguments:
""""""""""

The first argument is an integer value (result of some expression), the
second is a pointer to a global string, the third is a pointer to a
global string which is the source file name, and the last argument is
the line number. It returns the value of the first argument.

Semantics:
""""""""""

This intrinsic allows annotations to be put on arbitrary expressions with
arbitrary strings. This can be useful for special purpose optimizations that
want to look for these annotations. These have no other defined use;
transformations preserve annotations on a best-effort basis but are allowed to
replace the intrinsic with its first argument without breaking semantics and the
intrinsic is completely dropped during instruction selection.

'``llvm.codeview.annotation``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This annotation emits a label at its program point and an associated
``S_ANNOTATION`` codeview record with some additional string metadata. This is
used to implement MSVC's ``__annotation`` intrinsic. It is marked
``noduplicate``, so calls to this intrinsic prevent inlining and should be
considered expensive.

::

      declare void @llvm.codeview.annotation(metadata)

Arguments:
""""""""""

The argument should be an MDTuple containing any number of MDStrings.

.. _llvm.trap:

'``llvm.trap``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.trap() cold noreturn nounwind

Overview:
"""""""""

The '``llvm.trap``' intrinsic.

Arguments:
""""""""""

None.

Semantics:
""""""""""

This intrinsic is lowered to the target dependent trap instruction. If
the target does not have a trap instruction, this intrinsic will be
lowered to a call of the ``abort()`` function.

.. _llvm.debugtrap:

'``llvm.debugtrap``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.debugtrap() nounwind

Overview:
"""""""""

The '``llvm.debugtrap``' intrinsic.

Arguments:
""""""""""

None.

Semantics:
""""""""""

This intrinsic is lowered to code which is intended to cause an
execution trap with the intention of requesting the attention of a
debugger.

.. _llvm.ubsantrap:

'``llvm.ubsantrap``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.ubsantrap(i8 immarg) cold noreturn nounwind

Overview:
"""""""""

The '``llvm.ubsantrap``' intrinsic.

Arguments:
""""""""""

An integer describing the kind of failure detected.

Semantics:
""""""""""

This intrinsic is lowered to code which is intended to cause an execution trap,
embedding the argument into encoding of that trap somehow to discriminate
crashes if possible.

Equivalent to ``@llvm.trap`` for targets that do not support this behavior.

'``llvm.stackprotector``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.stackprotector(ptr <guard>, ptr <slot>)

Overview:
"""""""""

The ``llvm.stackprotector`` intrinsic takes the ``guard`` and stores it
onto the stack at ``slot``. The stack slot is adjusted to ensure that it
is placed on the stack before local variables.

Arguments:
""""""""""

The ``llvm.stackprotector`` intrinsic requires two pointer arguments.
The first argument is the value loaded from the stack guard
``@__stack_chk_guard``. The second variable is an ``alloca`` that has
enough space to hold the value of the guard.

Semantics:
""""""""""

This intrinsic causes the prologue/epilogue inserter to force the position of
the ``AllocaInst`` stack slot to be before local variables on the stack. This is
to ensure that if a local variable on the stack is overwritten, it will destroy
the value of the guard. When the function exits, the guard on the stack is
checked against the original guard by ``llvm.stackprotectorcheck``. If they are
different, then ``llvm.stackprotectorcheck`` causes the program to abort by
calling the ``__stack_chk_fail()`` function.

'``llvm.stackguard``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare ptr @llvm.stackguard()

Overview:
"""""""""

The ``llvm.stackguard`` intrinsic returns the system stack guard value.

It should not be generated by frontends, since it is only for internal usage.
The reason why we create this intrinsic is that we still support IR form Stack
Protector in FastISel.

Arguments:
""""""""""

None.

Semantics:
""""""""""

On some platforms, the value returned by this intrinsic remains unchanged
between loads in the same thread. On other platforms, it returns the same
global variable value, if any, e.g. ``@__stack_chk_guard``.

Currently some platforms have IR-level customized stack guard loading (e.g.
X86 Linux) that is not handled by ``llvm.stackguard()``, while they should be
in the future.

'``llvm.objectsize``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i32 @llvm.objectsize.i32(ptr <object>, i1 <min>, i1 <nullunknown>, i1 <dynamic>)
      declare i64 @llvm.objectsize.i64(ptr <object>, i1 <min>, i1 <nullunknown>, i1 <dynamic>)

Overview:
"""""""""

The ``llvm.objectsize`` intrinsic is designed to provide information to the
optimizer to determine whether a) an operation (like memcpy) will overflow a
buffer that corresponds to an object, or b) that a runtime check for overflow
isn't necessary. An object in this context means an allocation of a specific
class, structure, array, or other object.

Arguments:
""""""""""

The ``llvm.objectsize`` intrinsic takes four arguments. The first argument is a
pointer to or into the ``object``. The second argument determines whether
``llvm.objectsize`` returns 0 (if true) or -1 (if false) when the object size is
unknown. The third argument controls how ``llvm.objectsize`` acts when ``null``
in address space 0 is used as its pointer argument. If it's ``false``,
``llvm.objectsize`` reports 0 bytes available when given ``null``. Otherwise, if
the ``null`` is in a non-zero address space or if ``true`` is given for the
third argument of ``llvm.objectsize``, we assume its size is unknown. The fourth
argument to ``llvm.objectsize`` determines if the value should be evaluated at
runtime.

The second, third, and fourth arguments only accept constants.

Semantics:
""""""""""

The ``llvm.objectsize`` intrinsic is lowered to a value representing the size of
the object concerned. If the size cannot be determined, ``llvm.objectsize``
returns ``i32/i64 -1 or 0`` (depending on the ``min`` argument).

'``llvm.expect``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.expect`` on any
integer bit width.

::

      declare i1 @llvm.expect.i1(i1 <val>, i1 <expected_val>)
      declare i32 @llvm.expect.i32(i32 <val>, i32 <expected_val>)
      declare i64 @llvm.expect.i64(i64 <val>, i64 <expected_val>)

Overview:
"""""""""

The ``llvm.expect`` intrinsic provides information about expected (the
most probable) value of ``val``, which can be used by optimizers.

Arguments:
""""""""""

The ``llvm.expect`` intrinsic takes two arguments. The first argument is
a value. The second argument is an expected value.

Semantics:
""""""""""

This intrinsic is lowered to the ``val``.

'``llvm.expect.with.probability``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This intrinsic is similar to ``llvm.expect``. This is an overloaded intrinsic.
You can use ``llvm.expect.with.probability`` on any integer bit width.

::

      declare i1 @llvm.expect.with.probability.i1(i1 <val>, i1 <expected_val>, double <prob>)
      declare i32 @llvm.expect.with.probability.i32(i32 <val>, i32 <expected_val>, double <prob>)
      declare i64 @llvm.expect.with.probability.i64(i64 <val>, i64 <expected_val>, double <prob>)

Overview:
"""""""""

The ``llvm.expect.with.probability`` intrinsic provides information about
expected value of ``val`` with probability(or confidence) ``prob``, which can
be used by optimizers.

Arguments:
""""""""""

The ``llvm.expect.with.probability`` intrinsic takes three arguments. The first
argument is a value. The second argument is an expected value. The third
argument is a probability.

Semantics:
""""""""""

This intrinsic is lowered to the ``val``.

.. _int_assume:

'``llvm.assume``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.assume(i1 %cond)

Overview:
"""""""""

The ``llvm.assume`` allows the optimizer to assume that the provided
condition is true. This information can then be used in simplifying other parts
of the code.

More complex assumptions can be encoded as
:ref:`assume operand bundles <assume_opbundles>`.

Arguments:
""""""""""

The argument of the call is the condition which the optimizer may assume is
always true.

Semantics:
""""""""""

The intrinsic allows the optimizer to assume that the provided condition is
always true whenever the control flow reaches the intrinsic call. No code is
generated for this intrinsic, and instructions that contribute only to the
provided condition are not used for code generation. If the condition is
violated during execution, the behavior is undefined.

Note that the optimizer might limit the transformations performed on values
used by the ``llvm.assume`` intrinsic in order to preserve the instructions
only used to form the intrinsic's input argument. This might prove undesirable
if the extra information provided by the ``llvm.assume`` intrinsic does not cause
sufficient overall improvement in code quality. For this reason,
``llvm.assume`` should not be used to document basic mathematical invariants
that the optimizer can otherwise deduce or facts that are of little use to the
optimizer.

.. _int_ssa_copy:

'``llvm.ssa.copy``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare type @llvm.ssa.copy(type returned %operand) memory(none)

Arguments:
""""""""""

The first argument is an operand which is used as the returned value.

Overview:
""""""""""

The ``llvm.ssa.copy`` intrinsic can be used to attach information to
operations by copying them and giving them new names.  For example,
the PredicateInfo utility uses it to build Extended SSA form, and
attach various forms of information to operands that dominate specific
uses.  It is not meant for general use, only for building temporary
renaming forms that require value splits at certain points.

.. _type.test:

'``llvm.type.test``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i1 @llvm.type.test(ptr %ptr, metadata %type) nounwind memory(none)


Arguments:
""""""""""

The first argument is a pointer to be tested. The second argument is a
metadata object representing a :doc:`type identifier <../TypeMetadata>`.

Overview:
"""""""""

The ``llvm.type.test`` intrinsic tests whether the given pointer is associated
with the given type identifier.

.. _type.checked.load:

'``llvm.type.checked.load``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare {ptr, i1} @llvm.type.checked.load(ptr %ptr, i32 %offset, metadata %type) nounwind memory(argmem: read)


Arguments:
""""""""""

The first argument is a pointer from which to load a function pointer. The
second argument is the byte offset from which to load the function pointer. The
third argument is a metadata object representing a :doc:`type identifier
<../TypeMetadata>`.

Overview:
"""""""""

The ``llvm.type.checked.load`` intrinsic safely loads a function pointer from a
virtual table pointer using type metadata. This intrinsic is used to implement
control flow integrity in conjunction with virtual call optimization. The
virtual call optimization pass will optimize away ``llvm.type.checked.load``
intrinsics associated with devirtualized calls, thereby removing the type
check in cases where it is not needed to enforce the control flow integrity
constraint.

If the given pointer is associated with a type metadata identifier, this
function returns true as the second element of its return value. (Note that
the function may also return true if the given pointer is not associated
with a type metadata identifier.) If the function's return value's second
element is true, the following rules apply to the first element:

- If the given pointer is associated with the given type metadata identifier,
  it is the function pointer loaded from the given byte offset from the given
  pointer.

- If the given pointer is not associated with the given type metadata
  identifier, it is one of the following (the choice of which is unspecified):

  1. The function pointer that would have been loaded from an arbitrarily chosen
     (through an unspecified mechanism) pointer associated with the type
     metadata.

  2. If the function has a non-void return type, a pointer to a function that
     returns an unspecified value without causing side effects.

If the function's return value's second element is false, the value of the
first element is undefined.

.. _type.checked.load.relative:

'``llvm.type.checked.load.relative``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare {ptr, i1} @llvm.type.checked.load.relative(ptr %ptr, i32 %offset, metadata %type) nounwind memory(argmem: read)

Overview:
"""""""""

The ``llvm.type.checked.load.relative`` intrinsic loads a relative pointer to a
function from a virtual table pointer using metadata. Otherwise, its semantic is
identical to the ``llvm.type.checked.load`` intrinsic.

A relative pointer is a pointer to an offset. This is the offset between the destination
pointer and the original pointer. The address of the destination pointer is obtained
by loading this offset and adding it to the original pointer. This calculation is the
same as that of the ``llvm.load.relative`` intrinsic.

'``llvm.arithmetic.fence``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <type>
      @llvm.arithmetic.fence(<type> <op>)

Overview:
"""""""""

The purpose of the ``llvm.arithmetic.fence`` intrinsic
is to prevent the optimizer from performing fast-math optimizations,
particularly reassociation,
between the argument and the expression that contains the argument.
It can be used to preserve the parentheses in the source language.

Arguments:
""""""""""

The ``llvm.arithmetic.fence`` intrinsic takes only one argument.
The argument and the return value are floating-point numbers,
or vector floating-point numbers, of the same type.

Semantics:
""""""""""

This intrinsic returns the value of its operand. The optimizer can optimize
the argument, but the optimizer cannot hoist any component of the operand
to the containing context, and the optimizer cannot move the calculation of
any expression in the containing context into the operand.


'``llvm.donothing``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.donothing() nounwind memory(none)

Overview:
"""""""""

The ``llvm.donothing`` intrinsic doesn't perform any operation. It's one of only
three intrinsics (besides ``llvm.experimental.patchpoint`` and
``llvm.experimental.gc.statepoint``) that can be called with an invoke
instruction.

Arguments:
""""""""""

None.

Semantics:
""""""""""

This intrinsic does nothing, and it's removed by optimizers and ignored
by codegen.

'``llvm.experimental.deoptimize``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare type @llvm.experimental.deoptimize(...) [ "deopt"(...) ]

Overview:
"""""""""

This intrinsic, together with :ref:`deoptimization operand bundles
<deopt_opbundles>`, allow frontends to express transfer of control and
frame-local state from the currently executing (typically more specialized,
hence faster) version of a function into another (typically more generic, hence
slower) version.

In languages with a fully integrated managed runtime like Java and JavaScript
this intrinsic can be used to implement "uncommon trap" or "side exit" like
functionality.  In unmanaged languages like C and C++, this intrinsic can be
used to represent the slow paths of specialized functions.


Arguments:
""""""""""

The intrinsic takes an arbitrary number of arguments, whose meaning is
decided by the :ref:`lowering strategy<deoptimize_lowering>`.

Semantics:
""""""""""

The ``@llvm.experimental.deoptimize`` intrinsic executes an attached
deoptimization continuation (denoted using a :ref:`deoptimization
operand bundle <deopt_opbundles>`) and returns the value returned by
the deoptimization continuation.  Defining the semantic properties of
the continuation itself is out of scope of the language reference --
as far as LLVM is concerned, the deoptimization continuation can
invoke arbitrary side effects, including reading from and writing to
the entire heap.

Deoptimization continuations expressed using ``"deopt"`` operand bundles always
continue execution to the end of the physical frame containing them, so all
calls to ``@llvm.experimental.deoptimize`` must be in "tail position":

   - ``@llvm.experimental.deoptimize`` cannot be invoked.
   - The call must immediately precede a :ref:`ret <i_ret>` instruction.
   - The ``ret`` instruction must return the value produced by the
     ``@llvm.experimental.deoptimize`` call if there is one, or void.

Note that the above restrictions imply that the return type for a call to
``@llvm.experimental.deoptimize`` will match the return type of its immediate
caller.

The inliner composes the ``"deopt"`` continuations of the caller into the
``"deopt"`` continuations present in the inlinee, and also updates calls to this
intrinsic to return directly from the frame of the function it inlined into.

All declarations of ``@llvm.experimental.deoptimize`` must share the
same calling convention.

.. _deoptimize_lowering:

Lowering:
"""""""""

Calls to ``@llvm.experimental.deoptimize`` are lowered to calls to the
symbol ``__llvm_deoptimize`` (it is the frontend's responsibility to
ensure that this symbol is defined).  The call arguments to
``@llvm.experimental.deoptimize`` are lowered as if they were formal
arguments of the specified types, and not as varargs.


'``llvm.experimental.guard``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.experimental.guard(i1, ...) [ "deopt"(...) ]

Overview:
"""""""""

This intrinsic, together with :ref:`deoptimization operand bundles
<deopt_opbundles>`, allows frontends to express guards or checks on
optimistic assumptions made during compilation.  The semantics of
``@llvm.experimental.guard`` is defined in terms of
``@llvm.experimental.deoptimize`` -- its body is defined to be
equivalent to:

.. code-block:: text

  define void @llvm.experimental.guard(i1 %pred, <args...>) {
    %realPred = and i1 %pred, undef
    br i1 %realPred, label %continue, label %leave [, !make.implicit !{}]

  leave:
    call void @llvm.experimental.deoptimize(<args...>) [ "deopt"() ]
    ret void

  continue:
    ret void
  }


with the optional ``[, !make.implicit !{}]`` present if and only if it
is present on the call site.  For more details on ``!make.implicit``,
see :doc:`../FaultMaps`.

In words, ``@llvm.experimental.guard`` executes the attached
``"deopt"`` continuation if (but **not** only if) its first argument
is ``false``.  Since the optimizer is allowed to replace the ``undef``
with an arbitrary value, it can optimize guard to fail "spuriously",
i.e. without the original condition being false (hence the "not only
if"); and this allows for "check widening" type optimizations.

``@llvm.experimental.guard`` cannot be invoked.

After ``@llvm.experimental.guard`` was first added, a more general
formulation was found in ``@llvm.experimental.widenable.condition``.
Support for ``@llvm.experimental.guard`` is slowly being rephrased in
terms of this alternate.

'``llvm.experimental.widenable.condition``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i1 @llvm.experimental.widenable.condition()

Overview:
"""""""""

This intrinsic represents a "widenable condition" which is
boolean expressions with the following property: whether this
expression is `true` or `false`, the program is correct and
well-defined.

Together with :ref:`deoptimization operand bundles <deopt_opbundles>`,
``@llvm.experimental.widenable.condition`` allows frontends to
express guards or checks on optimistic assumptions made during
compilation and represent them as branch instructions on special
conditions.

While this may appear similar in semantics to `undef`, it is very
different in that an invocation produces a particular, singular
value. It is also intended to be lowered late, and remain available
for specific optimizations and transforms that can benefit from its
special properties.

Arguments:
""""""""""

None.

Semantics:
""""""""""

The intrinsic ``@llvm.experimental.widenable.condition()``
returns either `true` or `false`. For each evaluation of a call
to this intrinsic, the program must be valid and correct both if
it returns `true` and if it returns `false`. This allows
transformation passes to replace evaluations of this intrinsic
with either value whenever one is beneficial.

When used in a branch condition, it allows us to choose between
two alternative correct solutions for the same problem, like
in example below:

.. code-block:: text

    %cond = call i1 @llvm.experimental.widenable.condition()
    br i1 %cond, label %fast_path, label %slow_path

  fast_path:
    ; Apply memory-consuming but fast solution for a task.

  slow_path:
    ; Cheap in memory but slow solution.

Whether the result of intrinsic's call is `true` or `false`,
it should be correct to pick either solution. We can switch
between them by replacing the result of
``@llvm.experimental.widenable.condition`` with different
`i1` expressions.

This is how it can be used to represent guards as widenable branches:

.. code-block:: text

  block:
    ; Unguarded instructions
    call void @llvm.experimental.guard(i1 %cond, <args...>) ["deopt"(<deopt_args...>)]
    ; Guarded instructions

Can be expressed in an alternative equivalent form of explicit branch using
``@llvm.experimental.widenable.condition``:

.. code-block:: text

  block:
    ; Unguarded instructions
    %widenable_condition = call i1 @llvm.experimental.widenable.condition()
    %guard_condition = and i1 %cond, %widenable_condition
    br i1 %guard_condition, label %guarded, label %deopt

  guarded:
    ; Guarded instructions

  deopt:
    call type @llvm.experimental.deoptimize(<args...>) [ "deopt"(<deopt_args...>) ]

So the block `guarded` is only reachable when `%cond` is `true`,
and it should be valid to go to the block `deopt` whenever `%cond`
is `true` or `false`.

``@llvm.experimental.widenable.condition`` will never throw, thus
it cannot be invoked.

Guard widening:
"""""""""""""""

When ``@llvm.experimental.widenable.condition()`` is used in
condition of a guard represented as explicit branch, it is
legal to widen the guard's condition with any additional
conditions.

Guard widening looks like replacement of

.. code-block:: text

  %widenable_cond = call i1 @llvm.experimental.widenable.condition()
  %guard_cond = and i1 %cond, %widenable_cond
  br i1 %guard_cond, label %guarded, label %deopt

with

.. code-block:: text

  %widenable_cond = call i1 @llvm.experimental.widenable.condition()
  %new_cond = and i1 %any_other_cond, %widenable_cond
  %new_guard_cond = and i1 %cond, %new_cond
  br i1 %new_guard_cond, label %guarded, label %deopt

for this branch. Here `%any_other_cond` is an arbitrarily chosen
well-defined `i1` value. By making guard widening, we may
impose stricter conditions on `guarded` block and bail to the
deopt when the new condition is not met.

Lowering:
"""""""""

Default lowering strategy is replacing the result of
call of ``@llvm.experimental.widenable.condition``  with
constant `true`. However it is always correct to replace
it with any other `i1` value. Any pass can
freely do it if it can benefit from non-default lowering.

'``llvm.allow.ubsan.check``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i1 @llvm.allow.ubsan.check(i8 immarg %kind)

Overview:
"""""""""

This intrinsic returns ``true`` if and only if the compiler opted to enable the
ubsan check in the current basic block.

Rules to allow ubsan checks are not part of the intrinsic declaration, and
controlled by compiler options.

This intrinsic is the ubsan specific version of ``@llvm.allow.runtime.check()``.

Arguments:
""""""""""

An integer describing the kind of ubsan check guarded by the intrinsic.

Semantics:
""""""""""

The intrinsic ``@llvm.allow.ubsan.check()`` returns either ``true`` or
``false``, depending on compiler options.

For each evaluation of a call to this intrinsic, the program must be valid and
correct both if it returns ``true`` and if it returns ``false``.

When used in a branch condition, it selects one of the two paths:

* `true``: Executes the UBSan check and reports any failures.

* `false`: Bypasses the check, assuming it always succeeds.

Example:

.. code-block:: text

    %allow = call i1 @llvm.allow.ubsan.check(i8 5)
    %not.allow = xor i1 %allow, true
    %cond = or i1 %ubcheck, %not.allow
    br i1 %cond, label %cont, label %trap

  cont:
    ; Proceed

  trap:
    call void @llvm.ubsantrap(i8 5)
    unreachable


'``llvm.allow.runtime.check``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i1 @llvm.allow.runtime.check(metadata %kind)

Overview:
"""""""""

This intrinsic returns ``true`` if and only if the compiler opted to enable
runtime checks in the current basic block.

Rules to allow runtime checks are not part of the intrinsic declaration, and
controlled by compiler options.

This intrinsic is non-ubsan specific version of ``@llvm.allow.ubsan.check()``.

Arguments:
""""""""""

A string identifying the kind of runtime check guarded by the intrinsic. The
string can be used to control rules to allow checks.

Semantics:
""""""""""

The intrinsic ``@llvm.allow.runtime.check()`` returns either ``true`` or
``false``, depending on compiler options.

For each evaluation of a call to this intrinsic, the program must be valid and
correct both if it returns ``true`` and if it returns ``false``.

When used in a branch condition, it allows us to choose between
two alternative correct solutions for the same problem.

If the intrinsic is evaluated as ``true``, program should execute a guarded
check. If the intrinsic is evaluated as ``false``, the program should avoid any
unnecessary checks.

Example:

.. code-block:: text

    %allow = call i1 @llvm.allow.runtime.check(metadata !"my_check")
    br i1 %allow, label %fast_path, label %slow_path

  fast_path:
    ; Omit diagnostics.

  slow_path:
    ; Additional diagnostics.


'``llvm.load.relative``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare ptr @llvm.load.relative.iN(ptr %ptr, iN %offset) nounwind memory(argmem: read)

Overview:
"""""""""

This intrinsic loads a 32-bit value from the address ``%ptr + %offset``,
adds ``%ptr`` to that value and returns it. The constant folder specifically
recognizes the form of this intrinsic and the constant initializers it may
load from; if a loaded constant initializer is known to have the form
``i32 trunc(x - %ptr)``, the intrinsic call is folded to ``x``.

LLVM provides that the calculation of such a constant initializer will
not overflow at link time under the medium code model if ``x`` is an
``unnamed_addr`` function. However, it does not provide this guarantee for
a constant initializer folded into a function body. This intrinsic can be
used to avoid the possibility of overflows when loading from such a constant.

.. _llvm_sideeffect:

'``llvm.sideeffect``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.sideeffect() inaccessiblememonly nounwind willreturn

Overview:
"""""""""

The ``llvm.sideeffect`` intrinsic doesn't perform any operation. Optimizers
treat it as having side effects, so it can be inserted into a loop to
indicate that the loop shouldn't be assumed to terminate (which could
potentially lead to the loop being optimized away entirely), even if it's
an infinite loop with no other side effects.

Arguments:
""""""""""

None.

Semantics:
""""""""""

This intrinsic actually does nothing, but optimizers must assume that it
has externally observable side effects.

'``llvm.is.constant.*``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use llvm.is.constant with any argument type.

::

      declare i1 @llvm.is.constant.i32(i32 %operand) nounwind memory(none)
      declare i1 @llvm.is.constant.f32(float %operand) nounwind memory(none)
      declare i1 @llvm.is.constant.TYPENAME(TYPE %operand) nounwind memory(none)

Overview:
"""""""""

The '``llvm.is.constant``' intrinsic will return true if the argument
is known to be a manifest compile-time constant. It is guaranteed to
fold to either true or false before generating machine code.

Semantics:
""""""""""

This intrinsic generates no code. If its argument is known to be a
manifest compile-time constant value, then the intrinsic will be
converted to a constant true value. Otherwise, it will be converted to
a constant false value.

In particular, note that if the argument is a constant expression
which refers to a global (the address of which _is_ a constant, but
not manifest during the compile), then the intrinsic evaluates to
false.

The result also intentionally depends on the result of optimization
passes -- e.g., the result can change depending on whether a
function gets inlined or not. A function's parameters are
obviously not constant. However, a call like
``llvm.is.constant.i32(i32 %param)`` *can* return true after the
function is inlined, if the value passed to the function parameter was
a constant.

.. _int_ptrmask:

'``llvm.ptrmask``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare ptrty llvm.ptrmask(ptrty %ptr, intty %mask) speculatable memory(none)

Arguments:
""""""""""

The first argument is a pointer or vector of pointers. The second argument is
an integer or vector of integers with the same bit width as the index type
size of the first argument.

Overview:
""""""""""

The ``llvm.ptrmask`` intrinsic masks out bits of the pointer according to a mask.
This allows stripping data from tagged pointers without converting them to an
integer (ptrtoint/inttoptr). As a consequence, we can preserve more information
to facilitate alias analysis and underlying-object detection.

Semantics:
""""""""""

The result of ``ptrmask(%ptr, %mask)`` is equivalent to the following expansion,
where ``iPtrIdx`` is the index type size of the pointer::

    %intptr = ptrtoint ptr %ptr to iPtrIdx ; this may truncate
    %masked = and iPtrIdx %intptr, %mask
    %diff = sub iPtrIdx %masked, %intptr
    %result = getelementptr i8, ptr %ptr, iPtrIdx %diff

If the pointer index type size is smaller than the pointer type size, this
implies that pointer bits beyond the index size are not affected by this
intrinsic. For integral pointers, it behaves as if the mask were extended with
1 bits to the pointer type size.

Both the returned pointer(s) and the first argument are based on the same
underlying object (for more information on the *based on* terminology see
:ref:`the pointer aliasing rules <pointeraliasing>`).

The intrinsic only captures the pointer argument through the return value.

.. _int_threadlocal_address:

'``llvm.threadlocal.address``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare ptr @llvm.threadlocal.address(ptr) nounwind willreturn memory(none)

Arguments:
""""""""""

The `llvm.threadlocal.address` intrinsic requires a global value argument (a
:ref:`global variable <globalvars>` or alias) that is thread local.

Semantics:
""""""""""

The address of a thread local global is not a constant, since it depends on
the calling thread. The `llvm.threadlocal.address` intrinsic returns the
address of the given thread local global in the calling thread.

.. _int_vscale:

'``llvm.vscale``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare i32 llvm.vscale.i32()
      declare i64 llvm.vscale.i64()

Overview:
"""""""""

The ``llvm.vscale`` intrinsic returns the value for ``vscale`` in scalable
vectors such as ``<vscale x 16 x i8>``.

Semantics:
""""""""""

``vscale`` is a positive value that is constant throughout program
execution, but is unknown at compile time.
If the result value does not fit in the result type, then the result is
a :ref:`poison value <poisonvalues>`.

.. _llvm_fake_use:

'``llvm.fake.use``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare void @llvm.fake.use(...)

Overview:
"""""""""

The ``llvm.fake.use`` intrinsic is a no-op. It takes a single
value as an operand and is treated as a use of that operand, to force the
optimizer to preserve that value prior to the fake use. This is used for
extending the lifetimes of variables, where this intrinsic placed at the end of
a variable's scope helps prevent that variable from being optimized out.

Arguments:
""""""""""

The ``llvm.fake.use`` intrinsic takes one argument, which may be any
function-local SSA value. Note that the signature is variadic so that the
intrinsic can take any type of argument, but passing more than one argument will
result in an error.

Semantics:
""""""""""

This intrinsic does nothing, but optimizers must consider it a use of its single
operand and should try to preserve the intrinsic and its position in the
function.


Stack Map Intrinsics
--------------------

LLVM provides experimental intrinsics to support runtime patching
mechanisms commonly desired in dynamic language JITs. These intrinsics
are described in :doc:`../StackMaps`.

Element Wise Atomic Memory Intrinsics
-------------------------------------

These intrinsics are similar to the standard library memory intrinsics except
that they perform memory transfer as a sequence of atomic memory accesses.

.. _int_memcpy_element_unordered_atomic:

'``llvm.memcpy.element.unordered.atomic``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.memcpy.element.unordered.atomic`` on
any integer bit width and for different address spaces. Not all targets
support all bit widths however.

::

      declare void @llvm.memcpy.element.unordered.atomic.p0.p0.i32(ptr <dest>,
                                                                   ptr <src>,
                                                                   i32 <len>,
                                                                   i32 <element_size>)
      declare void @llvm.memcpy.element.unordered.atomic.p0.p0.i64(ptr <dest>,
                                                                   ptr <src>,
                                                                   i64 <len>,
                                                                   i32 <element_size>)

Overview:
"""""""""

The '``llvm.memcpy.element.unordered.atomic.*``' intrinsic is a specialization of the
'``llvm.memcpy.*``' intrinsic. It differs in that the ``dest`` and ``src`` are treated
as arrays with elements that are exactly ``element_size`` bytes, and the copy between
buffers uses a sequence of :ref:`unordered atomic <ordering>` load/store operations
that are a positive integer multiple of the ``element_size`` in size.

Arguments:
""""""""""

The first three arguments are the same as they are in the :ref:`@llvm.memcpy <int_memcpy>`
intrinsic, with the added constraint that ``len`` is required to be a positive integer
multiple of the ``element_size``. If ``len`` is not a positive integer multiple of
``element_size``, then the behavior of the intrinsic is undefined.

``element_size`` must be a compile-time constant positive power of two no greater than
target-specific atomic access size limit.

For each of the input pointers ``align`` parameter attribute must be specified. It
must be a power of two no less than the ``element_size``. Caller guarantees that
both the source and destination pointers are aligned to that boundary.

Semantics:
""""""""""

The '``llvm.memcpy.element.unordered.atomic.*``' intrinsic copies ``len`` bytes of
memory from the source location to the destination location. These locations are not
allowed to overlap. The memory copy is performed as a sequence of load/store operations
where each access is guaranteed to be a multiple of ``element_size`` bytes wide and
aligned at an ``element_size`` boundary.

The order of the copy is unspecified. The same value may be read from the source
buffer many times, but only one write is issued to the destination buffer per
element. It is well defined to have concurrent reads and writes to both source and
destination provided those reads and writes are unordered atomic when specified.

This intrinsic does not provide any additional ordering guarantees over those
provided by a set of unordered loads from the source location and stores to the
destination.

Lowering:
"""""""""

In the most general case call to the '``llvm.memcpy.element.unordered.atomic.*``' is
lowered to a call to the symbol ``__llvm_memcpy_element_unordered_atomic_*``. Where '*'
is replaced with an actual element size. See :ref:`RewriteStatepointsForGC intrinsic
lowering <RewriteStatepointsForGC_intrinsic_lowering>` for details on GC specific
lowering.

Optimizer is allowed to inline memory copy when it's profitable to do so.

'``llvm.memmove.element.unordered.atomic``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use
``llvm.memmove.element.unordered.atomic`` on any integer bit width and for
different address spaces. Not all targets support all bit widths however.

::

      declare void @llvm.memmove.element.unordered.atomic.p0.p0.i32(ptr <dest>,
                                                                    ptr <src>,
                                                                    i32 <len>,
                                                                    i32 <element_size>)
      declare void @llvm.memmove.element.unordered.atomic.p0.p0.i64(ptr <dest>,
                                                                    ptr <src>,
                                                                    i64 <len>,
                                                                    i32 <element_size>)

Overview:
"""""""""

The '``llvm.memmove.element.unordered.atomic.*``' intrinsic is a specialization
of the '``llvm.memmove.*``' intrinsic. It differs in that the ``dest`` and
``src`` are treated as arrays with elements that are exactly ``element_size``
bytes, and the copy between buffers uses a sequence of
:ref:`unordered atomic <ordering>` load/store operations that are a positive
integer multiple of the ``element_size`` in size.

Arguments:
""""""""""

The first three arguments are the same as they are in the
:ref:`@llvm.memmove <int_memmove>` intrinsic, with the added constraint that
``len`` is required to be a positive integer multiple of the ``element_size``.
If ``len`` is not a positive integer multiple of ``element_size``, then the
behavior of the intrinsic is undefined.

``element_size`` must be a compile-time constant positive power of two no
greater than a target-specific atomic access size limit.

For each of the input pointers the ``align`` parameter attribute must be
specified. It must be a power of two no less than the ``element_size``. Caller
guarantees that both the source and destination pointers are aligned to that
boundary.

Semantics:
""""""""""

The '``llvm.memmove.element.unordered.atomic.*``' intrinsic copies ``len`` bytes
of memory from the source location to the destination location. These locations
are allowed to overlap. The memory copy is performed as a sequence of load/store
operations where each access is guaranteed to be a multiple of ``element_size``
bytes wide and aligned at an ``element_size`` boundary.

The order of the copy is unspecified. The same value may be read from the source
buffer many times, but only one write is issued to the destination buffer per
element. It is well defined to have concurrent reads and writes to both source
and destination provided those reads and writes are unordered atomic when
specified.

This intrinsic does not provide any additional ordering guarantees over those
provided by a set of unordered loads from the source location and stores to the
destination.

Lowering:
"""""""""

In the most general case call to the
'``llvm.memmove.element.unordered.atomic.*``' is lowered to a call to the symbol
``__llvm_memmove_element_unordered_atomic_*``. Where '*' is replaced with an
actual element size. See :ref:`RewriteStatepointsForGC intrinsic lowering
<RewriteStatepointsForGC_intrinsic_lowering>` for details on GC specific
lowering.

The optimizer is allowed to inline the memory copy when it's profitable to do so.

.. _int_memset_element_unordered_atomic:

'``llvm.memset.element.unordered.atomic``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

This is an overloaded intrinsic. You can use ``llvm.memset.element.unordered.atomic`` on
any integer bit width and for different address spaces. Not all targets
support all bit widths however.

::

      declare void @llvm.memset.element.unordered.atomic.p0.i32(ptr <dest>,
                                                                i8 <value>,
                                                                i32 <len>,
                                                                i32 <element_size>)
      declare void @llvm.memset.element.unordered.atomic.p0.i64(ptr <dest>,
                                                                i8 <value>,
                                                                i64 <len>,
                                                                i32 <element_size>)

Overview:
"""""""""

The '``llvm.memset.element.unordered.atomic.*``' intrinsic is a specialization of the
'``llvm.memset.*``' intrinsic. It differs in that the ``dest`` is treated as an array
with elements that are exactly ``element_size`` bytes, and the assignment to that array
uses uses a sequence of :ref:`unordered atomic <ordering>` store operations
that are a positive integer multiple of the ``element_size`` in size.

Arguments:
""""""""""

The first three arguments are the same as they are in the :ref:`@llvm.memset <int_memset>`
intrinsic, with the added constraint that ``len`` is required to be a positive integer
multiple of the ``element_size``. If ``len`` is not a positive integer multiple of
``element_size``, then the behavior of the intrinsic is undefined.

``element_size`` must be a compile-time constant positive power of two no greater than
target-specific atomic access size limit.

The ``dest`` input pointer must have the ``align`` parameter attribute specified. It
must be a power of two no less than the ``element_size``. Caller guarantees that
the destination pointer is aligned to that boundary.

Semantics:
""""""""""

The '``llvm.memset.element.unordered.atomic.*``' intrinsic sets the ``len`` bytes of
memory starting at the destination location to the given ``value``. The memory is
set with a sequence of store operations where each access is guaranteed to be a
multiple of ``element_size`` bytes wide and aligned at an ``element_size`` boundary.

The order of the assignment is unspecified. Only one write is issued to the
destination buffer per element. It is well defined to have concurrent reads and
writes to the destination provided those reads and writes are unordered atomic
when specified.

This intrinsic does not provide any additional ordering guarantees over those
provided by a set of unordered stores to the destination.

Lowering:
"""""""""

In the most general case call to the '``llvm.memset.element.unordered.atomic.*``' is
lowered to a call to the symbol ``__llvm_memset_element_unordered_atomic_*``. Where '*'
is replaced with an actual element size.

The optimizer is allowed to inline the memory assignment when it's profitable to do so.

Objective-C ARC Runtime Intrinsics
----------------------------------

LLVM provides intrinsics that lower to Objective-C ARC runtime entry points.
LLVM is aware of the semantics of these functions, and optimizes based on that
knowledge. You can read more about the details of Objective-C ARC `here
<https://clang.llvm.org/docs/AutomaticReferenceCounting.html>`_.

'``llvm.objc.autorelease``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare ptr @llvm.objc.autorelease(ptr)

Lowering:
"""""""""

Lowers to a call to `objc_autorelease <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-autorelease>`_.

'``llvm.objc.autoreleasePoolPop``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare void @llvm.objc.autoreleasePoolPop(ptr)

Lowering:
"""""""""

Lowers to a call to `objc_autoreleasePoolPop <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#void-objc-autoreleasepoolpop-void-pool>`_.

'``llvm.objc.autoreleasePoolPush``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare ptr @llvm.objc.autoreleasePoolPush()

Lowering:
"""""""""

Lowers to a call to `objc_autoreleasePoolPush <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#void-objc-autoreleasepoolpush-void>`_.

'``llvm.objc.autoreleaseReturnValue``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare ptr @llvm.objc.autoreleaseReturnValue(ptr)

Lowering:
"""""""""

Lowers to a call to `objc_autoreleaseReturnValue <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-autoreleasereturnvalue>`_.

'``llvm.objc.copyWeak``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare void @llvm.objc.copyWeak(ptr, ptr)

Lowering:
"""""""""

Lowers to a call to `objc_copyWeak <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#void-objc-copyweak-id-dest-id-src>`_.

'``llvm.objc.destroyWeak``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare void @llvm.objc.destroyWeak(ptr)

Lowering:
"""""""""

Lowers to a call to `objc_destroyWeak <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#void-objc-destroyweak-id-object>`_.

'``llvm.objc.initWeak``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare ptr @llvm.objc.initWeak(ptr, ptr)

Lowering:
"""""""""

Lowers to a call to `objc_initWeak <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-initweak>`_.

'``llvm.objc.loadWeak``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare ptr @llvm.objc.loadWeak(ptr)

Lowering:
"""""""""

Lowers to a call to `objc_loadWeak <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-loadweak>`_.

'``llvm.objc.loadWeakRetained``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare ptr @llvm.objc.loadWeakRetained(ptr)

Lowering:
"""""""""

Lowers to a call to `objc_loadWeakRetained <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-loadweakretained>`_.

'``llvm.objc.moveWeak``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare void @llvm.objc.moveWeak(ptr, ptr)

Lowering:
"""""""""

Lowers to a call to `objc_moveWeak <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#void-objc-moveweak-id-dest-id-src>`_.

'``llvm.objc.release``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare void @llvm.objc.release(ptr)

Lowering:
"""""""""

Lowers to a call to `objc_release <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#void-objc-release-id-value>`_.

'``llvm.objc.retain``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare ptr @llvm.objc.retain(ptr)

Lowering:
"""""""""

Lowers to a call to `objc_retain <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-retain>`_.

'``llvm.objc.retainAutorelease``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare ptr @llvm.objc.retainAutorelease(ptr)

Lowering:
"""""""""

Lowers to a call to `objc_retainAutorelease <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-retainautorelease>`_.

'``llvm.objc.retainAutoreleaseReturnValue``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare ptr @llvm.objc.retainAutoreleaseReturnValue(ptr)

Lowering:
"""""""""

Lowers to a call to `objc_retainAutoreleaseReturnValue <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-retainautoreleasereturnvalue>`_.

'``llvm.objc.retainAutoreleasedReturnValue``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare ptr @llvm.objc.retainAutoreleasedReturnValue(ptr)

Lowering:
"""""""""

Lowers to a call to `objc_retainAutoreleasedReturnValue <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-retainautoreleasedreturnvalue>`_.

'``llvm.objc.retainBlock``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare ptr @llvm.objc.retainBlock(ptr)

Lowering:
"""""""""

Lowers to a call to `objc_retainBlock <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-retainblock>`_.

'``llvm.objc.storeStrong``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare void @llvm.objc.storeStrong(ptr, ptr)

Lowering:
"""""""""

Lowers to a call to `objc_storeStrong <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#void-objc-storestrong-id-object-id-value>`_.

'``llvm.objc.storeWeak``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare ptr @llvm.objc.storeWeak(ptr, ptr)

Lowering:
"""""""""

Lowers to a call to `objc_storeWeak <https://clang.llvm.org/docs/AutomaticReferenceCounting.html#arc-runtime-objc-storeweak>`_.

Preserving Debug Information Intrinsics
---------------------------------------

These intrinsics are used to carry certain debuginfo together with
IR-level operations. For example, it may be desirable to
know the structure/union name and the original user-level field
indices. Such information got lost in IR GetElementPtr instruction
since the IR types are different from debugInfo types and unions
are converted to structs in IR.

'``llvm.preserve.array.access.index``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare <ret_type>
      @llvm.preserve.array.access.index.p0s_union.anons.p0a10s_union.anons(<type> base,
                                                                           i32 dim,
                                                                           i32 index)

Overview:
"""""""""

The '``llvm.preserve.array.access.index``' intrinsic returns the getelementptr address
based on array base ``base``, array dimension ``dim`` and the last access index ``index``
into the array. The return type ``ret_type`` is a pointer type to the array element.
The array ``dim`` and ``index`` are preserved which is more robust than
getelementptr instruction which may be subject to compiler transformation.
The ``llvm.preserve.access.index`` type of metadata is attached to this call instruction
to provide array or pointer debuginfo type.
The metadata is a ``DICompositeType`` or ``DIDerivedType`` representing the
debuginfo version of ``type``.

Arguments:
""""""""""

The ``base`` is the array base address.  The ``dim`` is the array dimension.
The ``base`` is a pointer if ``dim`` equals 0.
The ``index`` is the last access index into the array or pointer.

The ``base`` argument must be annotated with an :ref:`elementtype
<attr_elementtype>` attribute at the call-site. This attribute specifies the
getelementptr element type.

Semantics:
""""""""""

The '``llvm.preserve.array.access.index``' intrinsic produces the same result
as a getelementptr with base ``base`` and access operands ``{dim's 0's, index}``.

'``llvm.preserve.union.access.index``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare <type>
      @llvm.preserve.union.access.index.p0s_union.anons.p0s_union.anons(<type> base,
                                                                        i32 di_index)

Overview:
"""""""""

The '``llvm.preserve.union.access.index``' intrinsic carries the debuginfo field index
``di_index`` and returns the ``base`` address.
The ``llvm.preserve.access.index`` type of metadata is attached to this call instruction
to provide union debuginfo type.
The metadata is a ``DICompositeType`` representing the debuginfo version of ``type``.
The return type ``type`` is the same as the ``base`` type.

Arguments:
""""""""""

The ``base`` is the union base address. The ``di_index`` is the field index in debuginfo.

Semantics:
""""""""""

The '``llvm.preserve.union.access.index``' intrinsic returns the ``base`` address.

'``llvm.preserve.struct.access.index``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""
::

      declare <ret_type>
      @llvm.preserve.struct.access.index.p0i8.p0s_struct.anon.0s(<type> base,
                                                                 i32 gep_index,
                                                                 i32 di_index)

Overview:
"""""""""

The '``llvm.preserve.struct.access.index``' intrinsic returns the getelementptr address
based on struct base ``base`` and IR struct member index ``gep_index``.
The ``llvm.preserve.access.index`` type of metadata is attached to this call instruction
to provide struct debuginfo type.
The metadata is a ``DICompositeType`` representing the debuginfo version of ``type``.
The return type ``ret_type`` is a pointer type to the structure member.

Arguments:
""""""""""

The ``base`` is the structure base address. The ``gep_index`` is the struct member index
based on IR structures. The ``di_index`` is the struct member index based on debuginfo.

The ``base`` argument must be annotated with an :ref:`elementtype
<attr_elementtype>` attribute at the call-site. This attribute specifies the
getelementptr element type.

Semantics:
""""""""""

The '``llvm.preserve.struct.access.index``' intrinsic produces the same result
as a getelementptr with base ``base`` and access operands ``{0, gep_index}``.

'``llvm.fptrunc.round``' Intrinsic
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Syntax:
"""""""

::

      declare <ty2>
      @llvm.fptrunc.round(<type> <value>, metadata <rounding mode>)

Overview:
"""""""""

The '``llvm.fptrunc.round``' intrinsic truncates
:ref:`floating-point <t_floating>` ``value`` to type ``ty2``
with a specified rounding mode.

Arguments:
""""""""""

The '``llvm.fptrunc.round``' intrinsic takes a :ref:`floating-point
<t_floating>` value to cast and a :ref:`floating-point <t_floating>` type
to cast it to. This argument must be larger in size than the result.

The second argument specifies the rounding mode as described in the constrained
intrinsics section.
For this intrinsic, the "round.dynamic" mode is not supported.

Semantics:
""""""""""

The '``llvm.fptrunc.round``' intrinsic casts a ``value`` from a larger
:ref:`floating-point <t_floating>` type to a smaller :ref:`floating-point
<t_floating>` type.
This intrinsic is assumed to execute in the default :ref:`floating-point
environment <floatenv>` *except* for the rounding mode.
This intrinsic is not supported on all targets. Some targets may not support
all rounding modes.

