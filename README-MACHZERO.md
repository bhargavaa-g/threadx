# MACHZERO — ThreadX Issue #585

**Repository:** [eclipse-threadx/threadx](https://github.com/eclipse-threadx/threadx)  
**Issue:** [#585 — Module converter utilities misparse ELF files when built for a 64-bit host](https://github.com/eclipse-threadx/threadx/issues/585)  

## 1. Executive summary

ThreadX issue #585 concerns the host-side programs that convert ThreadX module ELF files into firmware inputs. The tools represented ELF32 fields such as addresses and offsets with C's `unsigned long`. On common 64-bit Linux systems, `unsigned long` is 8 bytes, while an ELF32 field is 4 bytes. The converters then copied raw file bytes directly into C structures. The host structures therefore did not match the file layout, so the converter could read offsets and sizes from the wrong byte positions and emit incorrect output.

The proposed fix makes the file format explicit: read raw bytes, validate that the input is supported ELF32 little-endian, decode fields at fixed byte offsets into fixed-width integer values, check requested reads against the input file size, then let the existing output logic run.

## 2. Repository and component architecture

ThreadX is an embedded real-time operating system. At a high level, its source tree separates common kernel code, processor/compiler-specific ports, modules, and tests/build scripts.

| Area | Main role | Relation to issue #585 |
|---|---|---|
| `common/` | Portable kernel services | Not changed by this issue |
| `ports/` and `ports_arch/` | Processor and compiler-specific kernel ports | Not changed by this issue |
| `common_smp/`, `ports_smp/` | Symmetric multiprocessing kernel code and ports | Not changed by this issue |
| `common_modules/module_manager/` | ThreadX module manager and support code | Context for module loading |
| `common_modules/module_manager/utilities/` | Host-side module conversion utilities | Contains the two affected ELF parsers |
| `test/`, `scripts/` | Regression tests and build/check scripts | Places to inspect for test integration conventions |

The data flow is:

```text
ELF32 module file
      ↓
Host converter parses ELF metadata and finds code sections
      ↓
module_to_binary writes binary     OR     module_to_c_array writes C source
      ↓
Firmware includes/packages the generated module bytes
      ↓
ThreadX module manager loads the module on the target
```

The failure occurs on the developer host while interpreting the ELF input. It is not a scheduler defect and does not require hardware to reproduce the structure-size mismatch.

## 3. Issue understanding

### 3.1 Important terms

- **ELF:** Executable and Linkable Format, a file format for executable files and object files. It contains metadata and records describing sections and program segments.
- **ELF32:** The ELF format variant whose address/offset fields are generally 32 bits wide.
- **Host:** The computer that runs the converter, such as the developer's Mac or a Linux build machine.
- **ABI:** Application Binary Interface. Among other things, it defines type widths, alignment, and calling conventions for a platform/compiler combination.
- **LP64:** A common 64-bit ABI where `long` and pointers are 64 bits. In this environment, `unsigned long` is typically 8 bytes.
- **Little-endian:** The least significant byte of a multi-byte value is stored first. The fix accepts this ELF byte encoding and decodes it explicitly.
- **Offset:** A byte position in the file where a record or data region begins.
- **Section header:** ELF metadata describing a section, including its file offset, size, address, and flags.
- **Fixed-width integer:** A type such as `uint16_t` or `uint32_t`, whose width is specified in bits rather than varying with the host's `long` size.

### 3.2 Root cause

The original code declared ELF header, program header, section header, and symbol entry fields with types including `unsigned long`. That type is host-dependent. It then called a byte-copy helper with a pointer to a C structure, in a pattern like:

```c
elf_object_read(0, &header, sizeof(header));
```

`sizeof(header)` describes the host compiler's in-memory structure. It does not describe the on-disk ELF32 record. On the 64-bit Linux build described in the issue, the reported mismatch is:

| Record | ELF32 record in file | Original host structure reported by #585 |
|---|---:|---:|
| ELF header | 52 bytes | 80 bytes |
| ELF section header | 40 bytes | 80 bytes |

The direct copy has two related problems:

1. The host fields can be wider than the fields in ELF32.
2. The compiler can insert padding/alignment, so field offsets in memory need not match byte offsets in the file.

As a result, values such as the section-table offset or a section's file offset can be read from the wrong bytes. Later converter code may use those incorrect values to select or read code sections. The issue warns that this may produce plausible-looking but incorrect binary/C-array output rather than an obvious crash. It does not mean every conversion fails on every host.

### 3.3 Current behavior versus desired behavior

| Current behavior | Desired behavior |
|---|---|
| Uses host-dependent types for ELF32 fields | Represents parsed values with fixed-width types |
| Copies raw file bytes directly into C structures | Reads raw bytes and decodes each field at its defined file offset |
| Does not establish the input encoding before interpreting the header | Accepts the supported ELF32 little-endian encoding and clearly rejects unsupported input |
| Relies on later reads to discover some invalid offsets | Checks that requested ranges fit within the source file before reading |
| May continue with wrongly interpreted metadata | Uses decoded values only after format and bounds checks succeed |

## 4. Which source files are affected?

The issue says three utilities contain the ELF structures. Inspection of the current repository directory we found three tools, but only two read ELF headers and declare/consume those ELF records:

1. `common_modules/module_manager/utilities/module_to_binary.c`
2. `common_modules/module_manager/utilities/module_to_c_array.c`

The third utility, `module_binary_to_c_array.c`, accepts an already converted ThreadX binary and writes a C array. It does not parse ELF. The draft therefore changes the two ELF parsers and leaves the binary-to-C-array tool alone.
In each affected source file, the main relevant areas are:

- ELF constants, value types, and compile-time size checks near the top of the file.
- `module_elf32_decode_header()` — checks ELF identity and decodes the ELF32 header.
- `module_elf32_decode_section_header()` — decodes one 40-byte section header.
- `elf_object_read()` — performs bounded reads from the input file.
- `main()` — records source-file size, reads/decodes metadata, and invokes existing conversion logic.
- `converter_cleanup()` — closes files and frees temporary allocations on success or error paths.


## 5. Solution architecture

### 5.1 Parsing and conversion flow

Raw bytes from ELF file
      ↓
Validate magic, class, data encoding, version, and expected entry sizes
      ↓
Decode supported ELF32 little-endian fields into fixed-width values
      ↓
Check table/read ranges against the input file size
      ↓
Existing converter logic consumes parsed section values
      ↓
Write binary output or generated C-array output

### 5.2 Key design decisions

| Decision | Reason |
|---|---|
| Use `uint8_t`, `uint16_t`, and `uint32_t` for ELF values | ELF field widths are part of the file format and should not change with the host ABI. |
| Decode from byte arrays instead of overlaying C structures | Fixed-width fields alone do not remove host byte-order, alignment, or padding assumptions. |
| Accept ELF32 little-endian and reject other encodings | This is the format the converter is designed to read; rejecting unsupported input is clearer than misinterpreting it. |
|  |
| Use C99-compatible compile-time size checks | They catch accidental changes to expected value-structure sizes without requiring C11 `_Static_assert`. |
| Keep existing output-selection logic | It limits the fix to parsing and reduces risk to valid module conversion behavior. |
| Add no third-party ELF dependency | The supported input subset is small enough for these local helpers. |

### 5.3 How the converters use the parsed values

- The ELF header is read into a 52-byte raw buffer and decoded into the `ELF_HEADER` value.
- Program-header bytes are read with the ELF32 program-header record size (32 bytes). The tools do not consume their individual fields, so they remain raw bytes and are used to validate that the requested table range is readable.
- The section-header table is read as raw bytes using 40 bytes per record. Each record is decoded into `ELF_SECTION_HEADER` values before the existing loops inspect section type, flags, address, file offset, size, and name.
- The string table and selected code-section contents are then read using decoded file offsets/sizes. These reads go through the bounded read helper.
- `module_to_binary.c` writes the selected module code to a binary file.
- `module_to_c_array.c` writes the selected module code as C array source.
- The output file is opened after the header passes validation, so unsupported header input is rejected before output creation/truncation.
