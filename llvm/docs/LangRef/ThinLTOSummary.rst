========================
LangRef: ThinLTO Summary
========================

.. contents::
   :local:
   :depth: 3


.. _summary:


ThinLTO Summary
===============

Compiling with `ThinLTO <https://clang.llvm.org/docs/ThinLTO.html>`_
causes the building of a compact summary of the module that is emitted into
the bitcode. The summary is emitted into the LLVM assembly and identified
in syntax by a caret ('``^``').

The summary is parsed into a bitcode output, along with the Module
IR, via the "``llvm-as``" tool. Tools that parse the Module IR for the purposes
of optimization (e.g., "``clang -x ir``" and "``opt``"), will ignore the
summary entries (just as they currently ignore summary entries in a bitcode
input file).

Eventually, the summary will be parsed into a ModuleSummaryIndex object under
the same conditions where summary index is currently built from bitcode.
Specifically, tools that test the Thin Link portion of a ThinLTO compile
(i.e., llvm-lto and llvm-lto2), or when parsing a combined index
for a distributed ThinLTO backend via clang's "``-fthinlto-index=<>``" flag
(this part is not yet implemented, use llvm-as to create a bitcode object
before feeding into thin link tools for now).

There are currently 3 types of summary entries in the LLVM assembly:
:ref:`module paths<module_path_summary>`,
:ref:`global values<gv_summary>`, and
:ref:`type identifiers<typeid_summary>`.

.. _module_path_summary:

Module Path Summary Entry
-------------------------

Each module path summary entry lists a module containing global values included
in the summary. For a single IR module there will be one such entry, but
in a combined summary index produced during the thin link, there will be
one module path entry per linked module with summary.

Example:

.. code-block:: text

    ^0 = module: (path: "/path/to/file.o", hash: (2468601609, 1329373163, 1565878005, 638838075, 3148790418))

The ``path`` field is a string path to the bitcode file, and the ``hash``
field is the 160-bit SHA-1 hash of the IR bitcode contents, used for
incremental builds and caching.

.. _gv_summary:

Global Value Summary Entry
--------------------------

Each global value summary entry corresponds to a global value defined or
referenced by a summarized module.

Example:

.. code-block:: text

    ^4 = gv: (name: "f"[, summaries: (Summary)[, (Summary)]*]?) ; guid = 14740650423002898831

For declarations, there will not be a summary list. For definitions, a
global value will contain a list of summaries, one per module containing
a definition. There can be multiple entries in a combined summary index
for symbols with weak linkage.

Each ``Summary`` format will depend on whether the global value is a
:ref:`function<function_summary>`, :ref:`variable<variable_summary>`, or
:ref:`alias<alias_summary>`.

.. _function_summary:

Function Summary
^^^^^^^^^^^^^^^^

If the global value is a function, the ``Summary`` entry will look like:

.. code-block:: text

    function: (module: ^0, flags: (linkage: external, notEligibleToImport: 0, live: 0, dsoLocal: 0), insts: 2[, FuncFlags]?[, Calls]?[, TypeIdInfo]?[, Params]?[, Refs]?

The ``module`` field includes the summary entry id for the module containing
this definition, and the ``flags`` field contains information such as
the linkage type, a flag indicating whether it is legal to import the
definition, whether it is globally live and whether the linker resolved it
to a local definition (the latter two are populated during the thin link).
The ``insts`` field contains the number of IR instructions in the function.
Finally, there are several optional fields: :ref:`FuncFlags<funcflags_summary>`,
:ref:`Calls<calls_summary>`, :ref:`TypeIdInfo<typeidinfo_summary>`,
:ref:`Params<params_summary>`, :ref:`Refs<refs_summary>`.

.. _variable_summary:

Global Variable Summary
^^^^^^^^^^^^^^^^^^^^^^^

If the global value is a variable, the ``Summary`` entry will look like:

.. code-block:: text

    variable: (module: ^0, flags: (linkage: external, notEligibleToImport: 0, live: 0, dsoLocal: 0)[, Refs]?

The variable entry contains a subset of the fields in a
:ref:`function summary <function_summary>`, see the descriptions there.

.. _alias_summary:

Alias Summary
^^^^^^^^^^^^^

If the global value is an alias, the ``Summary`` entry will look like:

.. code-block:: text

    alias: (module: ^0, flags: (linkage: external, notEligibleToImport: 0, live: 0, dsoLocal: 0), aliasee: ^2)

The ``module`` and ``flags`` fields are as described for a
:ref:`function summary <function_summary>`. The ``aliasee`` field
contains a reference to the global value summary entry of the aliasee.

.. _funcflags_summary:

Function Flags
^^^^^^^^^^^^^^

The optional ``FuncFlags`` field looks like:

.. code-block:: text

    funcFlags: (readNone: 0, readOnly: 0, noRecurse: 0, returnDoesNotAlias: 0, noInline: 0, alwaysInline: 0, noUnwind: 1, mayThrow: 0, hasUnknownCall: 0)

If unspecified, flags are assumed to hold the conservative ``false`` value of
``0``.

.. _calls_summary:

Calls
^^^^^

The optional ``Calls`` field looks like:

.. code-block:: text

    calls: ((Callee)[, (Callee)]*)

where each ``Callee`` looks like:

.. code-block:: text

    callee: ^1[, hotness: None]?[, relbf: 0]?

The ``callee`` refers to the summary entry id of the callee. At most one
of ``hotness`` (which can take the values ``Unknown``, ``Cold``, ``None``,
``Hot``, and ``Critical``), and ``relbf`` (which holds the integer
branch frequency relative to the entry frequency, scaled down by 2^8)
may be specified. The defaults are ``Unknown`` and ``0``, respectively.

.. _params_summary:

Params
^^^^^^

The optional ``Params`` is used by ``StackSafety`` and looks like:

.. code-block:: text

    Params: ((Param)[, (Param)]*)

where each ``Param`` describes pointer parameter access inside of the
function and looks like:

.. code-block:: text

    param: 4, offset: [0, 5][, calls: ((Callee)[, (Callee)]*)]?

where the first ``param`` is the number of the parameter it describes,
``offset`` is the inclusive range of offsets from the pointer parameter to bytes
which can be accessed by the function. This range does not include accesses by
function calls from ``calls`` list.

where each ``Callee`` describes how parameter is forwarded into other
functions and looks like:

.. code-block:: text

    callee: ^3, param: 5, offset: [-3, 3]

The ``callee`` refers to the summary entry id of the callee,  ``param`` is
the number of the callee parameter which points into the callers parameter
with offset known to be inside of the ``offset`` range. ``calls`` will be
consumed and removed by thin link stage to update ``Param::offset`` so it
covers all accesses possible by ``calls``.

Pointer parameter without corresponding ``Param`` is considered unsafe and we
assume that access with any offset is possible.

Example:

If we have the following function:

.. code-block:: text

    define i64 @foo(ptr %0, ptr %1, ptr %2, i8 %3) {
      store ptr %1, ptr @x
      %5 = getelementptr inbounds i8, ptr %2, i64 5
      %6 = load i8, ptr %5
      %7 = getelementptr inbounds i8, ptr %2, i8 %3
      tail call void @bar(i8 %3, ptr %7)
      %8 = load i64, ptr %0
      ret i64 %8
    }

We can expect the record like this:

.. code-block:: text

    params: ((param: 0, offset: [0, 7]),(param: 2, offset: [5, 5], calls: ((callee: ^3, param: 1, offset: [-128, 127]))))

The function may access just 8 bytes of the parameter %0 . ``calls`` is empty,
so the parameter is either not used for function calls or ``offset`` already
covers all accesses from nested function calls.
Parameter %1 escapes, so access is unknown.
The function itself can access just a single byte of the parameter %2. Additional
access is possible inside of the ``@bar`` or ``^3``. The function adds signed
offset to the pointer and passes the result as the argument %1 into ``^3``.
This record itself does not tell us how ``^3`` will access the parameter.
Parameter %3 is not a pointer.

.. _refs_summary:

Refs
^^^^

The optional ``Refs`` field looks like:

.. code-block:: text

    refs: ((Ref)[, (Ref)]*)

where each ``Ref`` contains a reference to the summary id of the referenced
value (e.g., ``^1``).

.. _typeidinfo_summary:

TypeIdInfo
^^^^^^^^^^

The optional ``TypeIdInfo`` field, used for
`Control Flow Integrity <https://clang.llvm.org/docs/ControlFlowIntegrity.html>`_,
looks like:

.. code-block:: text

    typeIdInfo: [(TypeTests)]?[, (TypeTestAssumeVCalls)]?[, (TypeCheckedLoadVCalls)]?[, (TypeTestAssumeConstVCalls)]?[, (TypeCheckedLoadConstVCalls)]?

These optional fields have the following forms:

TypeTests
"""""""""

.. code-block:: text

    typeTests: (TypeIdRef[, TypeIdRef]*)

Where each ``TypeIdRef`` refers to a :ref:`type id<typeid_summary>`
by summary id or ``GUID``.

TypeTestAssumeVCalls
""""""""""""""""""""

.. code-block:: text

    typeTestAssumeVCalls: (VFuncId[, VFuncId]*)

Where each VFuncId has the format:

.. code-block:: text

    vFuncId: (TypeIdRef, offset: 16)

Where each ``TypeIdRef`` refers to a :ref:`type id<typeid_summary>`
by summary id or ``GUID`` preceded by a ``guid:`` tag.

TypeCheckedLoadVCalls
"""""""""""""""""""""

.. code-block:: text

    typeCheckedLoadVCalls: (VFuncId[, VFuncId]*)

Where each VFuncId has the format described for ``TypeTestAssumeVCalls``.

TypeTestAssumeConstVCalls
"""""""""""""""""""""""""

.. code-block:: text

    typeTestAssumeConstVCalls: (ConstVCall[, ConstVCall]*)

Where each ConstVCall has the format:

.. code-block:: text

    (VFuncId, args: (Arg[, Arg]*))

and where each VFuncId has the format described for ``TypeTestAssumeVCalls``,
and each Arg is an integer argument number.

TypeCheckedLoadConstVCalls
""""""""""""""""""""""""""

.. code-block:: text

    typeCheckedLoadConstVCalls: (ConstVCall[, ConstVCall]*)

Where each ConstVCall has the format described for
``TypeTestAssumeConstVCalls``.

.. _typeid_summary:

Type ID Summary Entry
---------------------

Each type id summary entry corresponds to a type identifier resolution
which is generated during the LTO link portion of the compile when building
with `Control Flow Integrity <https://clang.llvm.org/docs/ControlFlowIntegrity.html>`_,
so these are only present in a combined summary index.

Example:

.. code-block:: text

    ^4 = typeid: (name: "_ZTS1A", summary: (typeTestRes: (kind: allOnes, sizeM1BitWidth: 7[, alignLog2: 0]?[, sizeM1: 0]?[, bitMask: 0]?[, inlineBits: 0]?)[, WpdResolutions]?)) ; guid = 7004155349499253778

The ``typeTestRes`` gives the type test resolution ``kind`` (which may
be ``unsat``, ``byteArray``, ``inline``, ``single``, or ``allOnes``), and
the ``size-1`` bit width. It is followed by optional flags, which default to 0,
and an optional WpdResolutions (whole program devirtualization resolution)
field that looks like:

.. code-block:: text

    wpdResolutions: ((offset: 0, WpdRes)[, (offset: 1, WpdRes)]*

where each entry is a mapping from the given byte offset to the whole-program
devirtualization resolution WpdRes, that has one of the following formats:

.. code-block:: text

    wpdRes: (kind: branchFunnel)
    wpdRes: (kind: singleImpl, singleImplName: "_ZN1A1nEi")
    wpdRes: (kind: indir)

Additionally, each wpdRes has an optional ``resByArg`` field, which
describes the resolutions for calls with all constant integer arguments:

.. code-block:: text

    resByArg: (ResByArg[, ResByArg]*)

where ResByArg is:

.. code-block:: text

    args: (Arg[, Arg]*), byArg: (kind: UniformRetVal[, info: 0][, byte: 0][, bit: 0])

Where the ``kind`` can be ``Indir``, ``UniformRetVal``, ``UniqueRetVal``
or ``VirtualConstProp``. The ``info`` field is only used if the kind
is ``UniformRetVal`` (indicates the uniform return value), or
``UniqueRetVal`` (holds the return value associated with the unique vtable
(0 or 1)). The ``byte`` and ``bit`` fields are only used if the target does
not support the use of absolute symbols to store constants.

.. _intrinsicglobalvariables:

Intrinsic Global Variables
==========================

LLVM has a number of "magic" global variables that contain data that
affect code generation or other IR semantics. These are documented here.
All globals of this sort should have a section specified as
"``llvm.metadata``". This section and all globals that start with
"``llvm.``" are reserved for use by LLVM.

.. _gv_llvmused:

The '``llvm.used``' Global Variable
-----------------------------------

The ``@llvm.used`` global is an array which has
:ref:`appending linkage <linkage_appending>`. This array contains a list of
pointers to named global variables, functions and aliases which may optionally
have a pointer cast formed of bitcast or getelementptr. For example, a legal
use of it is:

.. code-block:: llvm

    @X = global i8 4
    @Y = global i32 123

    @llvm.used = appending global [2 x ptr] [
       ptr @X,
       ptr @Y
    ], section "llvm.metadata"

If a symbol appears in the ``@llvm.used`` list, then the compiler, assembler,
and linker are required to treat the symbol as if there is a reference to the
symbol that it cannot see (which is why they have to be named). For example, if
a variable has internal linkage and no references other than that from the
``@llvm.used`` list, it cannot be deleted. This is commonly used to represent
references from inline asms and other things the compiler cannot "see", and
corresponds to "``attribute((used))``" in GNU C.

On some targets, the code generator must emit a directive to the
assembler or object file to prevent the assembler and linker from
removing the symbol.

.. _gv_llvmcompilerused:

The '``llvm.compiler.used``' Global Variable
--------------------------------------------

The ``@llvm.compiler.used`` directive is the same as the ``@llvm.used``
directive, except that it only prevents the compiler from touching the
symbol. On targets that support it, this allows an intelligent linker to
optimize references to the symbol without being impeded as it would be
by ``@llvm.used``.

This is a rare construct that should only be used in rare circumstances,
and should not be exposed to source languages.

.. _gv_llvmglobalctors:

The '``llvm.global_ctors``' Global Variable
-------------------------------------------

.. code-block:: llvm

    %0 = type { i32, ptr, ptr }
    @llvm.global_ctors = appending global [1 x %0] [%0 { i32 65535, ptr @ctor, ptr @data }]

The ``@llvm.global_ctors`` array contains a list of constructor
functions, priorities, and an associated global or function.
The functions referenced by this array will be called in ascending order
of priority (i.e., lowest first) when the module is loaded. The order of
functions with the same priority is not defined.

If the third field is non-null, and points to a global variable
or function, the initializer function will only run if the associated
data from the current module is not discarded.
On ELF the referenced global variable or function must be in a comdat.

.. _llvmglobaldtors:

The '``llvm.global_dtors``' Global Variable
-------------------------------------------

.. code-block:: llvm

    %0 = type { i32, ptr, ptr }
    @llvm.global_dtors = appending global [1 x %0] [%0 { i32 65535, ptr @dtor, ptr @data }]

The ``@llvm.global_dtors`` array contains a list of destructor
functions, priorities, and an associated global or function.
The functions referenced by this array will be called in descending
order of priority (i.e., highest first) when the module is unloaded. The
order of functions with the same priority is not defined.

If the third field is non-null, and points to a global variable
or function, the destructor function will only run if the associated
data from the current module is not discarded.
On ELF the referenced global variable or function must be in a comdat.

