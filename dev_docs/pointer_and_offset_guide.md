# Guide to how pointers and offsets work in ACPICA source
This guide is targeted to help write the custom Compile/Dump handlers for a new
tables being added to ACPICA, updating or solving an issue in the Compile/Dump
path. Some constructs are not intuitive but have a very specific purpose which
might be hard to understand without a proper documentation of every trick or
practice used in the codebase.

## The Subtable ACPI_SUB_PTR trick
### Problem
One might want to modify table values when compiling the table. The simplest way
is to get a pointer into the table, casting that pointer with a table structure
from actbl*.h. These structures usually consist of the common ACPI header
structure and any additional table-specific data.

DtCompileTable() handles allocating space for DT_SUBTABLE->Buffer. Space for the
common 36-byte long ACPI header is allocated in the common code, while space for
subtables or any non-ACPI header data will be allocated in the custom
DtCompileX() handler. This means that one structure from actbl*.h representing
any main table fields will be covered by at least two separate allocations - one
for the common ACPI header part and at least one for the table-specific data.

Now consider the following code:
```C
    ParentTable = DtPeekSubtable ();

    Status = DtCompileTable (PFieldList, AcpiDmTableInfoX, &Subtable);
    if (ACPI_FAILURE (Status))
    {
        return (Status);
    }
    DtInsertSubtable (ParentTable, Subtable);
```

Subtable->Buffer is the pointer saved after DtCompileTable is done with
allocating space for the non-header part of AcpiDmTableInfoX. Let's say that the
table X has a field called 'Count' that is at offset 36 of the entire X table -
so just after the header. However accessing the 'Count' field using:
```C
    X = ACPI_CAST_PTR (ACPI_VIOT_HEADER, Subtable->Buffer);
    X->Count = ...
```

would be incorrect when using ACPI_CAST_PTR to get the address for X. That is
because X->Count means taking Subtable->Buffer and adding a 36 byte offset - the
ACPI header length - to it. This is wrong because Subtable->Buffer is the
allocated non ACPI header data - it already points at 'Count' so moving 36 bytes
further would mean a big out of bounds data access.

#### ParentTable
One might be tempted to try using the ParentTable->Buffer pointer instead since
it's pointing at the start of the whole X table - at the first byte of the ACPI
header of the X table:
```C
    X = ACPI_CAST_PTR (ACPI_VIOT_HEADER, ParentTable->Buffer);
    X->Count = ...
```

while this approach can turn out correct in many cases, it's still wrong. The
problem is the two separate allocations mentioned earlier - accessing
'X->Count' means accessing at least one byte out of bounds from the ACPI
header allocation. Since ParentTable->Buffer is the space allocated only for
the ACPI header it doesn't cover the X-specific data. So while in most cases
Subtable->Buffer will lie just after ParentTable->Buffer (not causing an out
of bounds access in practice) there are circumstances where the memory
allocator might add padding which will cause this relationship to fall
apart.

### Solution
The correct way is to use ACPI_SUB_PTR instead of ACPI_CAST_PTR:
```C
    X = ACPI_SUB_PTR (ACPI_TABLE_X, Subtable->Buffer,
        sizeof (ACPI_TABLE_HEADER));
    X->Count = ...
```

this allows safe access to 'X->Count' because the 'Count' field offset (+36)
will cancel out with the fact that the X pointer starts out at byte zero of the
ACPI_TABLE_X structure (offset -36 from the 'Count' field). It should also be
obvious that one shouldn't access any ACPI common header fields from the 'X'
pointer since the same padding problem might happen in the opposite direction.

### Profit
The only thing that this trick allows is the readability of using the 'X->Count'
notation. It's possible to use ACPI_CAST_PTR but in a less readable way:
```C
    UINT32 Count = *ACPI_CAST_PTR (UINT32, Subtable->Buffer);
    Count = ...
```

which will also become less readable if some further field of X should be
accessed.
