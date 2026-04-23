==============================
LangRef: Module-Level Metadata
==============================

.. contents::
   :local:
   :depth: 3


Module Flags Metadata
=====================

Information about the module as a whole is difficult to convey to LLVM's
subsystems. The LLVM IR isn't sufficient to transmit this information.
The ``llvm.module.flags`` named metadata exists in order to facilitate
this. These flags are in the form of key / value pairs --- much like a
dictionary --- making it easy for any subsystem that cares about a flag to
look it up.

The ``llvm.module.flags`` metadata contains a list of metadata triplets.
Each triplet has the following form:

-  The first element is a *behavior* flag, which specifies the behavior
   when two (or more) modules are merged together, and it encounters two
   (or more) metadata with the same ID. The supported behaviors are
   described below.
-  The second element is a metadata string that is a unique ID for the
   metadata. Each module may only have one flag entry for each unique ID (not
   including entries with the **Require** behavior).
-  The third element is the value of the flag.

When two (or more) modules are merged together, the resulting
``llvm.module.flags`` metadata is the union of the modules' flags. That is, for
each unique metadata ID string, there will be exactly one entry in the merged
modules ``llvm.module.flags`` metadata table, and the value for that entry will
be determined by the merge behavior flag, as described below. The only exception
is that entries with the *Require* behavior are always preserved.

The following behaviors are supported:

.. list-table::
   :header-rows: 1
   :widths: 10 90

   * - Value
     - Behavior

   * - 1
     - **Error**
           Emits an error if two values disagree, otherwise the resulting value
           is that of the operands.

   * - 2
     - **Warning**
           Emits a warning if two values disagree. The result value will be the
           operand for the flag from the first module being linked, unless the
           other module uses **Min** or **Max**, in which case the result will
           be **Min** (with the min value) or **Max** (with the max value),
           respectively.

   * - 3
     - **Require**
           Adds a requirement that another module flag be present and have a
           specified value after linking is performed. The value must be a
           metadata pair, where the first element of the pair is the ID of the
           module flag to be restricted, and the second element of the pair is
           the value the module flag should be restricted to. This behavior can
           be used to restrict the allowable results (via triggering of an
           error) of linking IDs with the **Override** behavior.

   * - 4
     - **Override**
           Uses the specified value, regardless of the behavior or value of the
           other module. If both modules specify **Override**, but the values
           differ, an error will be emitted.

   * - 5
     - **Append**
           Appends the two values, which are required to be metadata nodes.

   * - 6
     - **AppendUnique**
           Appends the two values, which are required to be metadata
           nodes. However, duplicate entries in the second list are dropped
           during the append operation.

   * - 7
     - **Max**
           Takes the max of the two values, which are required to be integers.

   * - 8
     - **Min**
           Takes the min of the two values, which are required to be non-negative integers.
           An absent module flag is treated as having the value 0.

It is an error for a particular unique flag ID to have multiple behaviors,
except in the case of **Require** (which adds restrictions on another metadata
value) or **Override**.

An example of module flags:

.. code-block:: llvm

    !0 = !{ i32 1, !"foo", i32 1 }
    !1 = !{ i32 4, !"bar", i32 37 }
    !2 = !{ i32 2, !"qux", i32 42 }
    !3 = !{ i32 3, !"qux",
      !{
        !"foo", i32 1
      }
    }
    !llvm.module.flags = !{ !0, !1, !2, !3 }

-  Metadata ``!0`` has the ID ``!"foo"`` and the value '1'. The behavior
   if two or more ``!"foo"`` flags are seen is to emit an error if their
   values are not equal.

-  Metadata ``!1`` has the ID ``!"bar"`` and the value '37'. The
   behavior if two or more ``!"bar"`` flags are seen is to use the value
   '37'.

-  Metadata ``!2`` has the ID ``!"qux"`` and the value '42'. The
   behavior if two or more ``!"qux"`` flags are seen is to emit a
   warning if their values are not equal.

-  Metadata ``!3`` has the ID ``!"qux"`` and the value:

   ::

       !{ !"foo", i32 1 }

   The behavior is to emit an error if the ``llvm.module.flags`` does not
   contain a flag with the ID ``!"foo"`` that has the value '1' after linking is
   performed.

Synthesized Functions Module Flags Metadata
-------------------------------------------

These metadata specify the default attributes synthesized functions should have.
These metadata are currently respected by a few instrumentation passes, such as
sanitizers.

These metadata correspond to a few function attributes with significant code
generation behaviors. Function attributes with just optimization purposes
should not be listed because the performance impact of these synthesized
functions is small.

- "frame-pointer": **Max**. The value can be 0, 1, or 2. A synthesized function
  will get the "frame-pointer" function attribute, with value being "none",
  "non-leaf", or "all", respectively.
- "function_return_thunk_extern": The synthesized function will get the
  ``fn_return_thunk_extern`` function attribute.
- "uwtable": **Max**. The value can be 0, 1, or 2. If the value is 1, a synthesized
  function will get the ``uwtable(sync)`` function attribute, if the value is 2,
  a synthesized function will get the ``uwtable(async)`` function attribute.

Objective-C Garbage Collection Module Flags Metadata
----------------------------------------------------

On the Mach-O platform, Objective-C stores metadata about garbage
collection in a special section called "image info". The metadata
consists of a version number and a bitmask specifying what types of
garbage collection are supported (if any) by the file. If two or more
modules are linked together their garbage collection metadata needs to
be merged rather than appended together.

The Objective-C garbage collection module flags metadata consists of the
following key-value pairs:

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Key
     - Value

   * - ``Objective-C Version``
     - **[Required]** --- The Objective-C ABI version. Valid values are 1 and 2.

   * - ``Objective-C Image Info Version``
     - **[Required]** --- The version of the image info section. Currently
       always 0.

   * - ``Objective-C Image Info Section``
     - **[Required]** --- The section to place the metadata. Valid values are
       ``"__OBJC, __image_info, regular"`` for Objective-C ABI version 1, and
       ``"__DATA,__objc_imageinfo, regular, no_dead_strip"`` for
       Objective-C ABI version 2.

   * - ``Objective-C Garbage Collection``
     - **[Required]** --- Specifies whether garbage collection is supported or
       not. Valid values are 0, for no garbage collection, and 2, for garbage
       collection supported.

   * - ``Objective-C GC Only``
     - **[Optional]** --- Specifies that only garbage collection is supported.
       If present, its value must be 6. This flag requires that the
       ``Objective-C Garbage Collection`` flag have the value 2.

Some important flag interactions:

-  If a module with ``Objective-C Garbage Collection`` set to 0 is
   merged with a module with ``Objective-C Garbage Collection`` set to
   2, then the resulting module has the
   ``Objective-C Garbage Collection`` flag set to 0.
-  A module with ``Objective-C Garbage Collection`` set to 0 cannot be
   merged with a module with ``Objective-C GC Only`` set to 6.

C type width Module Flags Metadata
----------------------------------

The ARM backend emits a section into each generated object file describing the
options that it was compiled with (in a compiler-independent way) to prevent
linking incompatible objects, and to allow automatic library selection. Some
of these options are not visible at the IR level, namely wchar_t width and enum
width.

To pass this information to the backend, these options are encoded in module
flags metadata, using the following key-value pairs:

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Key
     - Value

   * - short_wchar
     - * 0 --- sizeof(wchar_t) == 4
       * 1 --- sizeof(wchar_t) == 2

   * - short_enum
     - * 0 --- Enums are at least as large as an ``int``.
       * 1 --- Enums are stored in the smallest integer type which can
         represent all of its values.

For example, the following metadata section specifies that the module was
compiled with a ``wchar_t`` width of 4 bytes, and the underlying type of an
enum is the smallest type which can represent all of its values::

    !llvm.module.flags = !{!0, !1}
    !0 = !{i32 1, !"short_wchar", i32 1}
    !1 = !{i32 1, !"short_enum", i32 0}

Stack Alignment Metadata
------------------------

Changes the default stack alignment from the target ABI's implicit default
stack alignment. Takes an i32 value in bytes. It is considered an error to link
two modules together with different values for this metadata.

For example:

    !llvm.module.flags = !{!0}
    !0 = !{i32 1, !"override-stack-alignment", i32 8}

This will change the stack alignment to 8B.

Embedded Objects Names Metadata
===============================

Offloading compilations need to embed device code into the host section table to
create a fat binary. This metadata node references each global that will be
embedded in the module. The primary use for this is to make referencing these
globals more efficient in the IR. The metadata references nodes containing
pointers to the global to be embedded followed by the section name it will be
stored at::

    !llvm.embedded.objects = !{!0}
    !0 = !{ptr @object, !".section"}

Automatic Linker Flags Named Metadata
=====================================

Some targets support embedding of flags to the linker inside individual object
files. Typically this is used in conjunction with language extensions which
allow source files to contain linker command-line options, and have these
automatically be transmitted to the linker via object files.

These flags are encoded in the IR using named metadata with the name
``!llvm.linker.options``. Each operand is expected to be a metadata node
which should be a list of other metadata nodes, each of which should be a
list of metadata strings defining linker options.

For example, the following metadata section specifies two separate sets of
linker options, presumably to link against ``libz`` and the ``Cocoa``
framework::

    !0 = !{ !"-lz" }
    !1 = !{ !"-framework", !"Cocoa" }
    !llvm.linker.options = !{ !0, !1 }

The metadata encoding as lists of lists of options, as opposed to a collapsed
list of options, is chosen so that the IR encoding can use multiple option
strings to specify e.g., a single library, while still having that specifier be
preserved as an atomic element that can be recognized by a target-specific
assembly writer or object file emitter.

Each individual option is required to be either a valid option for the target's
linker, or an option that is reserved by the target-specific assembly writer or
object file emitter. No other aspect of these options is defined by the IR.

Dependent Libs Named Metadata
=============================

Some targets support embedding of strings into object files to indicate
a set of libraries to add to the link. Typically this is used in conjunction
with language extensions which allow source files to explicitly declare the
libraries they depend on, and have these automatically be transmitted to the
linker via object files.

The list is encoded in the IR using named metadata with the name
``!llvm.dependent-libraries``. Each operand is expected to be a metadata node
which should contain a single string operand.

For example, the following metadata section contains two library specifiers::

    !0 = !{!"a library specifier"}
    !1 = !{!"another library specifier"}
    !llvm.dependent-libraries = !{ !0, !1 }

Each library specifier will be handled independently by the consuming linker.
The effect of the library specifiers are defined by the consuming linker.

'``llvm.errno.tbaa``' Named Metadata
====================================

The module-level ``!llvm.errno.tbaa`` metadata specifies the TBAA nodes used
for accessing ``errno``. These nodes are guaranteed to represent int-compatible
accesses according to C/C++ strict aliasing rules. This should let LLVM alias
analyses to reason about aliasing with ``errno`` when calling library functions
that may set ``errno``, allowing optimizations such as store-to-load forwarding
across such routines.

For example, the following is a valid metadata specifying the TBAA information
for an integer access::

    !llvm.errno.tbaa = !{!0}
    !0 = !{!1, !1, i64 0}
    !1 = !{!"int", !2, i64 0}
    !2 = !{!"omnipotent char", !3, i64 0}
    !3 = !{!"Simple C/C++ TBAA"}

Multiple TBAA operands are allowed to support merging of modules that may use
different TBAA hierarchies (e.g., when mixing C and C++).

