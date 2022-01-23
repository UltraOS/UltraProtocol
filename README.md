# Introduction

Ultra loader protocol defines a set of services and interfaces used by a bootloader to load a kernel/binary file.

The protocol doesn't require any kind of sections residing inside the kernel binary and fully depends on loader-specific
configuration file to define all necessary load options and parameters. This allows to dynamically change all the parameters
without having to recompile the kernel as well as being hybrid ultra and any other protocol compatible at the same time.

# Protocol Description

After handoff, the kernel is provided with a boot context that has the following structure:

```c
struct ultra_boot_context {
    uint32_t unused;
    uint32_t attribute_count;
    struct ultra_attribute_header attributes[];
};
```

- `attribute_count` - the number of the attributes in the `attributes` array
- `attributes` - pointer to a contiguous array of attributes provided by the loader.

The following C macro can be used to retrieve the next attribute from current:

```c
#define NEXT_ATTRIBUTE(current) ((struct ultra_attribute_header*)(((uint8_t*)(current)) + (current)->size_in_bytes))
```

`ultra_attribute_header` and other structures are described in the following sections.

The kernel is also provided with a protocol magic number, which is defined as:  
`0x554c5442` - 'ULTB' in ASCII

The kernel entrypoint ends up looking something like this in C:
```c
void kernel_entry(struct ultra_boot_context *ctx, uint32_t magic)
```

The way the kernel receives this magic number is platform defined and is discussed in the later sections.

In case the kernel detects an invalid magic number the `ultra_boot_context*` must be considered invalid and it's recommended that the kernel aborts boot.

---

# Protocol Features And Options

This section defines various options and featrues defined by the protocol. The actual way to enable/set
an option is loader specific and is defined by its configuration file format.

### Binary Options
 
Defines various kernel binary related options.

- `binary` - (string) - shorthand alias for `binary/path`
- `binary/path` - (string) - specifies path to the kernel in a loader-defined format, if applicable.
- `binary/allocate-anywhere` (bool, optional, default=false) - allows the kernel physical memory mappings to be
allocated anywhere in memory. Only valid for 64 bit higher-half kernels.

### Command Line

Defines a command line to give to the kernel. This attribute is not generated if `cmdline` is set to null.

- `cmdline` (string, optional, default=null) - kernel command line, passed via `ultra_command_line_attribute`.

### Binary Stack

Defines various options related to the address passed in the arch-specific SP register on handover.
           
- `stack` (string/unsigned) - shorthand alias for `stack/size`. Also allows a literal "auto".
- `stack/size` (unsigned, optional, default=16K) - page aligned size of the stack to allocate.
- `stack/allocate-at` (string/unsigned, default=<implementation-defined>) - address where to allocate the stack

### Video Mode & Framebuffer
              
Defines options related to the video-mode set by the loader before handover.

- `video-mode` - (string) - shorthand, allowed values are the literal "unset" or `null`. `ultra_framebuffer_attribute` is not generated
if this is the case
- `video-mode/width` - (unsigned, optional, default=1024) - requests a specific framebuffer width
- `video-mode/height` - (unsigned, optional, default=768) - requests a specific framebuffer height
- `video-mode/bpp` (unsigned, optional, default=32) - requests a specific framebuffer bits per pixel value                                                                             
- `video-mode/constraint` (string, optional, default="at-least") - specifies a constraint for the video mode, one of "at-least", "exactly"
  
### Kernel Modules
                           
Defines a list of files for the loader to preload into ram before handover.

- `module` - (string) - shorthand alias for `module/path`
- `module/path` - (string) - specifies the path for the module to load
- `module/load-at` (unsigned/string, optional, unique, default="anywhere") - specifies the load address for the module
                  
Each directive generates a separate `ultra_module_info_attribute`

---

# State After Handoff

## x86
- A20 - enabled
- R/EFLAGS - zeroed, reserved bit 1 set
- IDTR - contents are undefined
- GDTR - set to a valid GDT with at least ring 0 flat code/data descriptors
- Long/protected mode (with paging) - set as determined by the kernel binary

- CS - set to a flat ring 0 code segment
- DS, ES, FS, GS, SS - set to a flat ring 0 data segment

### i386
- EIP - set to the entrypoint as specified by the kernel binary
- EAX, ECX, EDX, EBX, EBP, ESI, EDI - zeroed
- ESP - set to a valid stack pointer as determined by the configuration, aligned according to SysV ABI
- *ESP+8 - protocol magic
- *ESP+4 - `ultra_boot_context*`
- *ESP - zero (for SysV ABI alignment)

### AMD64
- RIP - set to the entrypoint as specified by the kernel binary
- RDI - `ultra_boot_context*`
- RSI - protocol magic
- RAX, RCX, RDX, RBX, RBP, r8, r9, r10, r11, r12, r13, r14, r15 - zeroed
- RSP - set to a valid stack pointer as determined by the configuration, aligned according to SysV ABI
- *RSP - zero (for SysV ABI alignment)
- CR3 - a valid address of a PML4 with the following mappings:

| physical address      | virtual address       | length of the mapping |
|-----------------------|-----------------------|-----------------------|
| 0x0000'0000'0000'0000 | 0x0000'0000'0000'0000 | 4 GB                  |
| 0x0000'0000'0000'0000 | 0xFFFF'8000'0000'0000 | 4 GB                  |
| ????????????????????? | 0xFFFF'FFFF'8000'0000 | ????????????????????? |

The last mapping depends on configuration-specific settings:
For higher half kernels loaded with allocate-anywhere set to on it contains
the kernel binary mappings with an arbitrary physical base picked by the loader.
For all other kernels it's a direct mapping of the first 2GB of physical ram.

Address pointed to by CR3 is located somewhere within ULTRA_MEMORY_TYPE_LOADER_RECLAIMABLE.

The contents of all other registers are unspecified.

---

# Attributes

The way the loader provides various information to the kernel is via attributes.  
Every attribute has a distinct type and starts with the following header:

```c
struct ultra_attribute_header {
    uint32_t type;
    uint32_t size_in_bytes;
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

- `size_in_bytes` - size of the entire attribute including the header. This size is often use for calculating the number of entries
  in a variable length attribute. This isn't always possible, because sometimes the size is increased on purpose to align entries to 8 bytes.
  Attributes like this are explicitly documented and provide a separate member that indicates the actual size of the variable size field.

---

# Attribute Types

This section describes all currently implemented attribute types and their structure.  
Each attribute type occurs in the `ultra_boot_context` exactly once unless specified otherwise.  
Kernel must ignore any attribute type that it's not familiar with.  
it's guaranteed that all future attribute types will start with an `ultra_attribute_header` as well.  

---

## ULTRA_ATTRIBUTE_INVALID
Reserved. If encountered, must be considered a fatal error.

---

## ULTRA_ATTRIBUTE_PLATFORM_INFO
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
- `loader_name`  - null-terminated ascii string with a human-readable name of the loader
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

```c
struct ultra_kernel_info_attribute {
    struct ultra_attribute_header header;

    uint64_t physical_base;
    uint64_t virtual_base;
    uint64_t range_length;

    uint64_t partition_type;

    // only valid if partition_type == ULTRA_PARTITION_TYPE_GPT
    struct ultra_guid disk_guid;
    struct ultra_guid partition_guid;

    // always valid
    uint32_t disk_index;
    uint32_t partition_index;

    char path_on_disk[256];
};
```

- `header` - standard attribute header
- `physical_base` - physical address of the kernel base, page aligned
- `virtual_base` - virtual address of the kernel basem, page aligned
- `range_length` - number of bytes taken by the kernel, page aligned
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
- `path_on_disk` - null terminated UTF-8 string, path to the kernel binary on the given partition

### ULTRA_PARTITION_TYPE_INVALID
Reserved. If encountered, must be considered a fatal error.

### ULTRA_PARTITION_TYPE_RAW
Unpartitioned device without an MBR/GPT header, the entire disk treated as one file system.

### ULTRA_PARTITION_TYPE_MBR
Standard MBR partitioned, either MBR or EBR.

### ULTRA_PARTITION_TYPE_GPT
Standard GPT partition, whether the disk also contains a valid MBR is undefined.

`struct ultra_guid` is defined as:
```c
struct ultra_guid {
    uint32_t data1;
    uint16_t data2;
    uint16_t data3;
    uint8_t  data4[8];
};
```

---

## ATTRIBUTE_MEMORY_MAP
This attributes provides a physical memory map of the entire system.
It has the following format:
```c
struct memory_map_attribute {
    struct attribute_header header;
    struct memory_map_entry entries[];
};
```

- `header` - standard attribute header
- `entries` - an array of non-overlapping and sorted in ascending order by address entries, where each entry has the following structure:

```c
struct ultra_memory_map_entry {
    uint64_t physical_address;
    uint64_t size_in_bytes;
    uint64_t type;
};
```
- `physical_address` - first byte of the physical range covered by this entry
- `size_in_bytes` - size of this range
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
Memory reserved by the firmware. Contents and read/write operations are undefined behavior.
               
### ULTRA_MEMORY_TYPE_RECLAIMABLE
Memory tagged as reclaimable by the firmware. Usually contains ACPI tables.

### ULTRA_MEMORY_TYPE_NVS
Same as MEMORY_TYPE_RESERVED. Consult the ACPI specification for more information.

### MEMORY_TYPE_LOADER_RECLAIMABLE
Memory reserved by the bootloader. Contains temporary GDT, all attributes, memory map itself, loader code, and everything
else the loader has allocated the memory for. The actual location of all the aforementioned structures within this range is
undefined.

Can be reclaimed by the kernel when it no longer needs the loader-provided structures.

### ULTRA_MEMORY_TYPE_MODULE
Memory region containing one or more kernel modules

### ULTRA_MEMORY_TYPE_KERNEL_STACK
Memory region reserved by the loader for the kernel stack. Not necessarily the same value as SP for higher half kernels.

### ULTRA_MEMORY_TYPE_KERNEL_BINARY
Memory region reserved by the loader for the loaded kernel binary (not the ELF copy).

### Any Other Value
Reserved for future use. Must be considered same as `MEMORY_TYPE_RESERVED` if encountered by the kernel.  
The number of memory map entries can be calculated using the following C macro:

```c
#define MEMORY_MAP_ENTRY_COUNT(header) ((((header).size_in_bytes) - sizeof(struct ultra_attribute_header)) / sizeof(struct ultra_memory_map_entry))
```

---

## ULTRA_ATTRIBUTE_MODULE_INFO
Every loaded kernel module gets a respective attribute of this type, this means  
`ultra_boot_context` must contain as many attributes of this type as there are modules.

This attribute provides information necessary to locate a kernel module in memory
and has the following structure:

```c
struct module_info_attribute {
    struct ultra_attribute_header header;
    char name[64];
    uint64_t physical_address;
    uint64_t length;
};
```

- `header` - standard attribute header
- `name` - null-terminated ascii name of the module, as specified in the configuration file
- `physical_address` - address of the first byte of the loaded module
- `length` - size of the module in memory

---

## ULTRA_ATTRIBUTE_COMMAND_LINE
This attribute forwards the configuration file command line to the kernel if one was provided and has the following structure:

```c
struct command_line_attribute {
    struct ultra_attribute_header header;
    char text[];
};
```

- `header` - standard attribute header
- `text` - null terminated ASCII command line, as specified in the configuration file

Length of the command line is not artifically limited in any way but must fit in the `header.size_in_bytes` field,
which is 32 bits wide. Note that `header.size_in_bytes` is not necessarily equal to the length of the command line
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
- `framebuffer` - describes the allocated framebuffer and has the following structure:

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
#define ULTRA_FB_FORMAT_INVALID 0
#define ULTRA_FB_FORMAT_RBG     1
#define ULTRA_FB_FORMAT_RGBA    2
```

### ULTRA_FB_FORMAT_INVALID
Reserved. If encountered, must be considered a fatal error.

### ULTRA_FB_FORMAT_RBG
This is a standard RBG format.

Layout of each pixel this format:

| bits  | 0 ... 8 | 8 ... 16 | 16 ... 24 |
|-------|---------|----------|-----------|
| color | BLUE    | GREEN    | RED       |

`bpp` must be set to 24.

### ULTRA_FB_FORMAT_RGBA
This is a standard RGB format padded to 32 bits.

Layout of each pixel for this format:

| bits  | 0 ... 8 | 8 ... 16 | 16 ... 24 | 24 ... 32 |
|-------|---------|----------|-----------|-----------|
| color | BLUE    | GREEN    | RED       | UNUSED    |

`bpp` must be set to 32.

- `physical_address` - address of the first physical byte of the allocated framebuffer

---
 
# Implementation

All structure definitions and macros can be found in the `ultra_protocol.h` file  
compatible with both C and C++.

The current reference implementation of the protocol in C can be found at https://github.com/UltraOS/Hyper
