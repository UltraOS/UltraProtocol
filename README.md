# Introduction

Ultra loader protocol defines a set of services and interfaces used by a bootloader to load a kernel/binary file.

The protocol doesn't require any kind of sections residing inside the kernel binary and fully depends on loader-specific
configuration file to define all necessary load options and parameters. This allows to dynamically change all the parameters
without having to recompile the kernel as well as being hybrid ultra and any other protocol compatible at the same time.

# Protocol Description

After handoff, the kernel is provided with a boot context that has the following structure:

```c
struct ultra_boot_context {
    uint8_t protocol_major;
    uint8_t protocol_minor;
    uint16_t reserved;

    uint32_t attribute_count;
    struct ultra_attribute_header attributes[];
};
```

- `protocol_major` - major version of the protocol, valid versions start at `1`
- `protocol_minor` - minor version of the protocol, valid versions start at `0`
- `attribute_count` - the number of attributes in the `attributes` array
- `attributes` - a contiguous array of attributes provided by the loader

The following C macro can be used to retrieve the next attribute from current:

```c
#define ULTRA_NEXT_ATTRIBUTE(current) ((struct ultra_attribute_header*)(((uint8_t*)(current)) + (current)->size))
```

`ultra_attribute_header` and other structures are described in the following sections.

The kernel is also provided with a protocol magic number `ULTRA_MAGIC`, which is defined as:  
`0x554c5442` - 'ULTB' in ASCII

The kernel entrypoint ends up looking something like this in C:
```c
void kernel_entry(struct ultra_boot_context *ctx, uint32_t magic)
```

The way the kernel receives this magic number is platform defined and is discussed in the later sections.

In case the kernel detects an invalid magic number the `ultra_boot_context*` must be considered invalid and it's recommended that the kernel aborts boot.

---

# Protocol Features And Options

Current protocol version is defined as `1.0`.

This section defines various options and features defined by the protocol. The actual way to enable/set
an option is loader specific and is defined by its configuration file format.

### Binary Options

Defines various kernel binary related options.

- `binary` - (string) - shorthand alias for `binary/path`
- `binary/path` - (string) - specifies the path to the kernel in a loader-defined format, if applicable.
- `binary/allocate-anywhere` (bool, optional, default=false) - allows the kernel physical memory mappings to be
  allocated anywhere in memory. Only valid for 64 bit higher-half kernels.

### Page Table Options

Defines various options related to the page table properties set up at handover.

- `page-table/levels` - (integer, optional, default=4) - specifies the desired depth of the page table
- `page-table/constraint` - (string, optional, default="maximum") - specifies the constraint for the "levels" value, one of "maximum", "at-least", "exactly"
- `page-table/null-guard` - (bool, optional, default=false) - if set to `true`, the first physical page is not identity mapped to prevent
  accidental `NULL` accesses.

### Command Line

Defines a command line to give to the kernel. This attribute is not generated if `cmdline` is set to null.

- `cmdline` (string, optional, default=null) - kernel command line, passed via `ultra_command_line_attribute`.

### Binary Stack

Defines various options related to the address passed in the arch-specific SP register on handover.

- `stack` (string/unsigned) - shorthand alias for `stack/size`. Also allows a literal "auto".
- `stack/size` (unsigned, optional, default=16K) - page aligned size of the stack to allocate.
- `stack/allocate-at` (string/unsigned, default=\<implementation-defined\>) - address where to allocate the stack

### Video Mode & Framebuffer

Defines options related to the video-mode set by the loader before handover.

- `video-mode` - (string, optional, default="auto") - shorthand. `"auto"` implies native width/height, `null` or `"unset"` are also
  allowed, `ultra_framebuffer_attribute` is not generated if this is the case.
- `video-mode/width` - (unsigned, optional, default=\<native\>) - requests a specific framebuffer width
- `video-mode/height` - (unsigned, optional, default=\<native\>) - requests a specific framebuffer height
- `video-mode/bpp` (unsigned, optional, default=32) - requests a specific framebuffer bits per pixel value
- `video-mode/format` (string, optional, default="auto") - requests a specific framebuffer format, one of `"auto"`,
  `"rgb888"`, `"bgr888"`, `"rgbx8888"`, `"xrgb8888"` (any case). This option is not affected by `constraint` and
  is always matched exactly if specified.
- `video-mode/constraint` (string, optional, default="at-least") - specifies a constraint for the video mode, one of "at-least", "exactly"

### Kernel Modules

Ultra protocol offers different types of modules: classic file-backed modules as well as raw RAM allocations.

The following options must be used to request a module:

- `kernel-as-module` - (bool, optional, default=false) - requests the loader to pass the full kernel binary
  as a separate module, this can be used for parsing additional debug information, enforcing memory protection
  for program headers from an ELF file or any other purpose.
- `module` - (string) - shorthand alias for `module/path`
- `module/type` (string, optional, default="file") - one of "file" or "memory". File modules require a path argument
  and simply make the loader preload a file from disk into RAM. Memory modules are used to request general purpose
  contiguous zeroed memory allocations without any backing, and can be used for bootstrapping kernel allocators or any other purpose.
- `module/size` (unsigned/string, optional*, default="auto") - defines the size of the module. Mandatory for "memory" modules.
  Can be used to truncate or extend "file" modules, if this is bigger than the file size, the rest of the memory is zeroed.
- `module/name` - (string, optional, default=\<implementation-defined\>) - name of the module that the kernel receives
- `module/path` - (string, optional*) - specifies the path for the module to load in a loader-defined format, if applicable
- `module/load-at` (unsigned/string, optional, default="anywhere") - specifies the load address for the module or "anywhere"

Each directive generates a separate `ultra_module_info_attribute`

### Miscellaneous

- `higher-half-exclusive` - (bool, optional, default=false) - if set to `true`, no identity mappings are provided for lower half.
  All loader-provided attribute `address` fields are relocated to higher half if present. Only applicable for higher-half kernels.

---

# State After Handoff

This section describes the system state for each supported architecture that system software must expect after gaining control.

## x86

Page size is defined as 4096 bytes.

- A20 - enabled
- R/EFLAGS - zeroed, reserved bit 1 set
- IDTR - contents are unspecified
- GDTR - set to a valid GDT with at least ring 0 flat code/data descriptors
- Long/protected mode - set as determined by the kernel binary
- CS - set to a flat ring 0 code segment
- DS, ES, FS, GS, SS - set to a flat ring 0 data segment

### i386
Higher half is defined as `0xC000'0000`  
The kernel is considered higher half if it wants to be loaded at or above `0xC010'0000`

- Paging enabled
- EIP - set to the entrypoint as specified by the kernel binary
- EAX, ECX, EDX, EBX, EBP, ESI, EDI - zeroed
- ESP - set to a valid stack pointer as determined by the configuration, aligned according to SysV ABI
- *ESP+8 - `ULTRA_MAGIC`
- *ESP+4 - `ultra_boot_context*`

| virtual address | physical address | length of the mapping |
|-----------------|------------------|-----------------------|
| 0x0000'0000     | 0x0000'0000      | 3 GiB                 |
| 0xC000'0000     | 0x0000'0000      | 1 GiB                 |

The first mapping is not provided for the `higher-half-exclusive` mode.

### AMD64

Higher half is defined as `0xFFFF'8000'0000'0000`  
The kernel is considered higher half if it wants to be loaded at or above `0xFFFF'FFFF'8000'0000`

- RIP - set to the entrypoint as specified by the kernel binary
- RDI - `ultra_boot_context*`
- RSI - `ULTRA_MAGIC`
- RAX, RCX, RDX, RBX, RBP, R8, R9, R10, R11, R12, R13, R14, R15 - zeroed
- RSP - set to a valid stack pointer as determined by the configuration, aligned according to SysV ABI
- CR3 - a valid address of a PML4 with the following mappings:

| virtual address       | physical address      | length of the mapping     |
|-----------------------|-----------------------|---------------------------|
| 0x0000'0000'0000'0000 | 0x0000'0000'0000'0000 | 4 GiB + any entries above |
| 0xFFFF'8000'0000'0000 | 0x0000'0000'0000'0000 | 4 GiB + any entries above |
| 0xFFFF'FFFF'8000'0000 | ????????????????????? | ?????????????????????     |

First two mappings are guaranteed to cover the first 4 GiB of physical memory
as well as any other entries present in the memory map on top of that.

The first mapping is not provided for the `higher-half-exclusive` mode.

For higher half kernels loaded with `allocate-anywhere` set to `true`
the last mapping contains the kernel binary mappings with an arbitrary
physical base picked by the loader.
For all other kernels it is a direct mapping of the first 2 GiB of physical ram.

---

Address pointed to by CR3 is located somewhere within `ULTRA_MEMORY_TYPE_LOADER_RECLAIMABLE`.
Whether the memory is mapped using 4K/2M/4M/1G pages is unspecified.

The contents of all other registers are unspecified.

---

# Attributes

The way the loader provides various information to the kernel is via attributes.

Guarantees about the attribute array & attributes:
- All attributes are guaranteed to be aligned on an 8 (or more if required) byte
  boundary.
- All attributes and `address` fields, as well as any loader-allocated memory
  that might be accessed by the kernel is guaranteed to be within the mapped
  address space range. E.g for a 32-bit higher half exclusive kernel all attributes
  and modules are guaranteed to be allocated under 1 GiB of physical memory so that
  they can be safely accessed by the kernel.
- The location of any specific attribute within the array is not fixed
  unless specified otherwise (in the attribute description).
- All attributes of the same type are guaranteed to be a contiguous stream.
- Every attribute type can only appear in the attribute array once unless
  specified otherwise (in the attribute description).
- All attribute types are guaranteed to have `ultra_attribute_header`
  as the first member, this includes both current & future attributes.
- It is safe to ignore any unknown attribute types.
- All currently defined attributes are guaranteed to keep the same layout
  in the future protocol versions, but might have new members added at the end,
  which shouldn't affect any valid software as the header `size` field will be adjusted
  accordingly.

Every attribute has a distinct type and starts with the following header:

```c
struct ultra_attribute_header {
    uint32_t type;
    uint32_t size;
};
```

- `type` - one of the following values (each type is detailed in the following sections):
```c
#define ULTRA_ATTRIBUTE_INVALID          0
#define ULTRA_ATTRIBUTE_PLATFORM_INFO    1
#define ULTRA_ATTRIBUTE_KERNEL_INFO      2
#define ULTRA_ATTRIBUTE_MEMORY_MAP       3
#define ULTRA_ATTRIBUTE_MODULE_INFO      4
#define ULTRA_ATTRIBUTE_COMMAND_LINE     5
#define ULTRA_ATTRIBUTE_FRAMEBUFFER_INFO 6
```

- `size` - size of the entire attribute including the header. This size is often used for calculating the number of entries
  in a variable length attribute. This isn't always possible, because sometimes the size is increased on purpose to align entries to 8 bytes.
  Attributes like this are explicitly documented and provide a separate member that indicates the actual size of the variable size field.

---

# Attribute Types

This section describes all currently implemented attribute types and their structure.

---

## ULTRA_ATTRIBUTE_INVALID
Reserved. If encountered, must be considered a fatal error.

---

## ULTRA_ATTRIBUTE_PLATFORM_INFO
This attribute is guaranteed to be the first entry in the attribute array.

This attribute provides various information about the platform and has the following structure:

```c
struct platform_info_attribute {
    struct ultra_attribute_header header;
    uint32_t platform_type;

    uint16_t loader_major;
    uint16_t loader_minor;
    char loader_name[32];

    uint64_t acpi_rsdp_address;
};
```
- `header` - standard attribute header
- `platform_type` - one of the following values:
```c
#define ULTRA_PLATFORM_INVALID 0
#define ULTRA_PLATFORM_BIOS    1
#define ULTRA_PLATFORM_UEFI    2
```
- `loader_major` - major version of the loader
- `loader_minor` - minor version of the loader
- `loader_name`  - null-terminated ASCII string with a human-readable name of the loader
- `acpi_rsdp_address` - physical address of the RSDP table, 0 if not applicable or not present

### ULTRA_PLATFORM_INVALID
Reserved. If encountered, must be considered a fatal error.

### ULTRA_PLATFORM_BIOS
The kernel was loaded using the BIOS services and platform.

### ULTRA_PLATFORM_UEFI
The kernel was loaded using the UEFI services and platform.

### Any Other Value
Reserved for future use, must be ignored by the kernel.

---

## ULTRA_ATTRIBUTE_KERNEL_INFO
This attribute is guaranteed to be the second entry in the attribute array.

This attribute provides various information about the kernel binary and has the following structure:

```c
struct ultra_kernel_info_attribute {
    struct ultra_attribute_header header;

    uint64_t physical_base;
    uint64_t virtual_base;
    uint64_t size;

    uint64_t partition_type;

    // only valid if partition_type == PARTITION_TYPE_GPT
    struct ultra_guid disk_guid;
    struct ultra_guid partition_guid;

    // always valid
    uint32_t disk_index;
    uint32_t partition_index;

    char fs_path[256];
};
```

- `header` - standard attribute header
- `physical_base` - physical address of the kernel base, page aligned
- `virtual_base` - virtual address of the kernel base, page aligned
- `size` - number of bytes taken by the kernel, page aligned
- `partition_type` - one of the following values:
```c
#define ULTRA_PARTITION_TYPE_INVALID 0
#define ULTRA_PARTITION_TYPE_RAW     1
#define ULTRA_PARTITION_TYPE_MBR     2
#define ULTRA_PARTITION_TYPE_GPT     3
```
- `disk_guid` - GUID of the disk that the kernel was loaded from, only valid for `ULTRA_PARTITION_TYPE_GPT`
- `partiton_guid` - GUID of the partition that the kernel was loaded from, only valid for `ULTRA_PARTITION_TYPE_GPT`
- `disk_index` - index of the disk the kernel was loaded from
- `partition_index` - index of the partition the kernel was loaded from, index >= 4 implies EBR partition N - 4 for an MBR disk
- `fs_path` - null terminated UTF-8 string, absolute POSIX path to the kernel binary on the partition

### ULTRA_PARTITION_TYPE_INVALID
Reserved. If encountered, must be considered a fatal error.

### ULTRA_PARTITION_TYPE_RAW
Unpartitioned device without an MBR/GPT header, the entire disk treated as one file system.

### ULTRA_PARTITION_TYPE_MBR
Standard MBR partition, either MBR or EBR.

### ULTRA_PARTITION_TYPE_GPT
Standard GPT partition, whether the device also contains a valid MBR is unspecified
as GPT always takes precedence.

`ultra_guid` is defined as:
```c
struct ultra_guid {
    uint32_t data1;
    uint16_t data2;
    uint16_t data3;
    uint8_t  data4[8];
};
```
Note that the structure is only guaranteed to be 8 byte aligned within `ultra_kernel_info_attribute`.

### Any Other Value
Reserved for future use, must be ignored by the kernel.

---

## ULTRA_ATTRIBUTE_MEMORY_MAP
This attributes provides a physical memory map of the entire system.
It has the following format:
```c
struct ultra_memory_map_attribute {
    struct ultra_attribute_header header;
    struct ultra_memory_map_entry entries[];
};
```

- `header` - standard attribute header
- `entries` - an array of non-overlapping and sorted in ascending order by address entries, where each entry has the following structure:

```c
struct ultra_memory_map_entry {
    uint64_t physical_address;
    uint64_t size;
    uint64_t type;
};
```
- `physical_address` - first byte of the physical range covered by this entry
- `size` - size of this range
- `type` - one of the following values:

```c
#define ULTRA_MEMORY_TYPE_INVALID            0x00000000
#define ULTRA_MEMORY_TYPE_FREE               0x00000001
#define ULTRA_MEMORY_TYPE_RESERVED           0x00000002
#define ULTRA_MEMORY_TYPE_RECLAIMABLE        0x00000003
#define ULTRA_MEMORY_TYPE_NVS                0x00000004
#define ULTRA_MEMORY_TYPE_LOADER_RECLAIMABLE 0xFFFF0001
#define ULTRA_MEMORY_TYPE_MODULE             0xFFFF0002
#define ULTRA_MEMORY_TYPE_KERNEL_STACK       0xFFFF0003
#define ULTRA_MEMORY_TYPE_KERNEL_BINARY      0xFFFF0004
```

### ULTRA_MEMORY_TYPE_INVALID
Reserved. If encountered, must be considered a fatal error.

### ULTRA_MEMORY_TYPE_FREE
General purpose memory free for use by the kernel.

### ULTRA_MEMORY_TYPE_RESERVED
Memory reserved by the firmware.

### ULTRA_MEMORY_TYPE_RECLAIMABLE
Memory tagged as reclaimable by the firmware. Usually contains ACPI tables.

### ULTRA_MEMORY_TYPE_NVS
Same as MEMORY_TYPE_RESERVED. Consult the ACPI specification for more information.

### ULTRA_MEMORY_TYPE_LOADER_RECLAIMABLE
Memory reserved by the bootloader. Contains temporary GDT, all attributes, memory map itself, loader code, and everything
else the loader has allocated the memory for. The actual location of all the aforementioned structures within this range is
unspecified.

Can be reclaimed by the kernel when it no longer needs the loader-provided structures.

### ULTRA_MEMORY_TYPE_MODULE
Memory region containing one or more kernel modules

### ULTRA_MEMORY_TYPE_KERNEL_STACK
Memory region reserved by the loader for the kernel stack. Not necessarily the same value as SP for higher half kernels.

### ULTRA_MEMORY_TYPE_KERNEL_BINARY
Memory region reserved by the loader for the loaded kernel binary (not the ELF copy).

### Any Other Value
Reserved for future use. Must be considered same as `ULTRA_MEMORY_TYPE_RESERVED` if encountered by the kernel.

The number of memory map entries can be calculated using the following C macro:

```c
#define ULTRA_MEMORY_MAP_ENTRY_COUNT(header) ((((header).size) - sizeof(struct ultra_attribute_header)) / sizeof(struct ultra_memory_map_entry))
```

---

## ULTRA_ATTRIBUTE_MODULE_INFO
Every loaded kernel module gets a respective attribute of this type, this means
`ultra_boot_context` must contain as many attributes of this type as there are modules.

This attribute provides information necessary to locate a kernel module in memory
and has the following structure:

```c
struct ultra_module_info_attribute {
    struct ultra_attribute_header header;
    uint32_t reserved;
    uint32_t type;
    char name[64];
    uint64_t address;
    uint64_t size;
};
```

- `header` - standard attribute header
- `reserved` - for use by future versions of the protocol, must be ignored
- `type` - type of this module, can be one of the following values:
```c
#define ULTRA_MODULE_TYPE_INVALID 0
#define ULTRA_MODULE_TYPE_FILE    1
#define ULTRA_MODULE_TYPE_MEMORY  2
```

### ULTRA_MODULE_TYPE_INVALID
Reserved. If encountered, must be considered a fatal error.

### ULTRA_MODULE_TYPE_FILE
File-backed module.

### ULTRA_MODULE_TYPE_MEMORY
Contiguous zeroed memory module.

- `name` - null-terminated ASCII name of the module, as specified in the configuration file
  or `"__KERNEL__"` for the autogenerated kernel binary module (if enabled)
- `address` - address of the loaded module, page aligned
- `size` - size of the module as specified in `module/size`, the actual size in RAM is this value rounded up to page size

---

## ULTRA_ATTRIBUTE_COMMAND_LINE
This attribute forwards the configuration file command line to the kernel if one was provided and has the following structure:

```c
struct ultra_command_line_attribute {
    struct ultra_attribute_header header;
    char text[];
};
```

- `header` - standard attribute header
- `text` - null terminated ASCII command line, as specified in the configuration file

Length of the command line is not artificially limited in any way but must fit in the `header.size` field,
which is 32 bits wide.
Note that `header.size` is not necessarily equal to the length of the command line
because of alignment reasons.

---

## ULTRA_ATTRIBUTE_FRAMEBUFFER
This attributes provides framebuffer information if one was requested and has the following structure:
```c
struct ultra_framebuffer_attribute {
    struct ultra_attribute_header header;
    struct ultra_framebuffer fb;
};
```

- `header` - standard attribute header
- `fb` - describes the allocated framebuffer and has the following structure:

```c
struct ultra_framebuffer {
    uint32_t width;
    uint32_t height;
    uint32_t pitch;
    uint16_t bpp;
    uint16_t format;
    uint64_t physical_address;
};
```

- `width` - the number of visible pixels per row in the framebuffer
- `height` - the number of rows of visible pixels in the framebuffer
- `pitch` - the number of bytes that each row takes in memory
- `bpp` - width in bits of each visible pixel
- `format` - the format of the allocated framebuffer, and is one of the following values:

```c
#define ULTRA_FB_FORMAT_INVALID  0
#define ULTRA_FB_FORMAT_RGB888   1
#define ULTRA_FB_FORMAT_BGR888   2
#define ULTRA_FB_FORMAT_RGBX8888 3
#define ULTRA_FB_FORMAT_XRGB8888 4
```

Framebuffer format types are very similar to `DRM_FORMAT_*` and are defined as follows:

### ULTRA_FB_FORMAT_INVALID
Reserved. If encountered, must be considered a fatal error.

### ULTRA_FB_FORMAT_RGB888
Standard RBG format.

Layout of each pixel (low to high memory address):

| bits  | 0 ... 8 | 8 ... 16 | 16 ... 24 |
|-------|---------|----------|-----------|
| color | BLUE    | GREEN    | RED       |

`bpp` must be set to 24.

### ULTRA_FB_FORMAT_BGR888
Standard BGR format.

Layout of each pixel (low to high memory address):

| bits  | 0 ... 8 | 8 ... 16 | 16 ... 24 |
|-------|---------|----------|-----------|
| color | RED     | GREEN    | BLUE      |

`bpp` must be set to 24.

### ULTRA_FB_FORMAT_RGBX8888
Standard RGBX format padded to 32 bits.

Layout of each pixel (low to high memory address):

| bits  | 0 ... 8 | 8 ... 16 | 16 ... 24 | 24 ... 32 |
|-------|---------|----------|-----------|-----------|
| color | UNUSED  | BLUE     | GREEN     | RED       |

`bpp` must be set to 32.

### ULTRA_FB_FORMAT_XRGB8888
Standard RGBX format padded to 32 bits.

Layout of each pixel (low to high memory address):

| bits  | 0 ... 8 | 8 ... 16 | 16 ... 24 | 24 ... 32 |
|-------|---------|----------|-----------|-----------|
| color | BLUE    | GREEN    | RED       | UNUSED    |

`bpp` must be set to 32.

- `physical_address` - address of the allocated framebuffer

---

# Implementation

All structure definitions and macros can be found in the `ultra_protocol.h` file
compatible with both C and C++.

The current reference implementation of the protocol in C can be found at https://github.com/UltraOS/Hyper
