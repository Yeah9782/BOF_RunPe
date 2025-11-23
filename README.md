# BOF_RunPE

BOF RunPE is a Beacon Object File for Cobalt Strike that executes PE files entirely in-memory within the beacon process. Unlike traditional fork&run, **no child process is spawned, no console is created, and no pipe is used** - all output is captured via IAT hooking and redirected to the beacon console.

**Architecture:** x64 only

## Overview

```
┌──────────────────────────────────────────────────────────────┐
│                   Cobalt Strike Beacon                       │
│                    (Current Process)                         │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         │  beacon_inline_execute()
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                    BOF RunPE                                 │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  VxTable + Draugr Initialization                       │  │
│  │  (Syscall Resolution + Stack Spoofing)                 │  │
│  └──────────────────────┬─────────────────────────────────┘  │
│                         │                                    │
│  ┌──────────────────────▼─────────────────────────────────┐  │
│  │  PE Mapping                                            │  │
│  │  - Section Copy    - IAT Patching (with hooks)         │  │
│  │  - Relocations     - Memory Protection                 │  │
│  └──────────────────────┬─────────────────────────────────┘  │
│                         │                                    │
│  ┌──────────────────────▼─────────────────────────────────┐  │
│  │  Thread Execution                                      │  │
│  │  - Spoofed Start Address                               │  │
│  │  - RIP Hijacking to Entry Point                        │  │
│  │  - Output Redirection via Hooks                        │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

## Key Features

- **No Process Creation**: PE runs inside the beacon process
- **No Console/Pipe**: Output captured via `printf`/`WriteConsole` hooks
- **Multiple Allocation Methods**: Heap, VirtualAlloc, Module Stomping
- **Proxy Loading**: Timer Queue, RegisterWait, or direct calls
- **Ntdll Unhooking**: Optional fresh copy from disk
- **RWX** : Optional allocate memory in RWX
- **Thread Start Spoofing**: Legitimate start address with RIP hijacking

## Configuration Options

The behaviorus of BOF can be edited in `Additionals postex` -> `RunPe Config`

![Custom BOF](/img/custom_bof.png)

### Proxy Methods

| Method |  Description |
|--------|--------------|
| `None`      | Direct API calls |
| `Draugr`    | Stack spoofed API calls |
| `Regwait`   | RegisterWaitForSingleObject callback |
| `Timer`     | Timer Queue callback |

### Allocation Methods

| Method | Description  | 
|--------|--------------|
| `Heap`                | Private heap via RtlCreateHeap with Draugr | 
| `VirtualAlloc`        | NtAllocateVirtualMemory with Draugr | 
| `Module stomping`     | Overwrites legitimate DLL .text section | 

### General Options

| Option |  Description |
|--------|-------------------|
| `AllocRWX`    |  Allocate as RWX (vs RW→RX transition) |
| `UnhookNtdll` |  Replace ntdll.dll .text with fresh copy from disk |
| `Timeout`     |  Execution timeout in milliseconds (0 = infinite) |
| `StompModule` |  DLL path for module stomping (e.g., `chakra.dll`) |

### Thread Spoofing

| Option | Description |
|--------|-------------|
| `ModuleName`      | Legitimate module for start address (e.g., `Kernel32.dll`) |
| `ProcedureName`   | Function name within module (e.g., `BaseThreadInitThunk`) |
| `Offset`          | Offset from function start |

## Output Capture

All PE output is redirected to the beacon console via IAT hooks. **No console window or named pipe is created.**

| Hooked Function | Target |
|-----------------|--------|
| `GetCommandLineA/W`                   | Returns spoofed arguments |
| `__getmainargs` / `__wgetmainargs`    | CRT argument initialization |
| `printf` / `wprintf`                  | BeaconPrintf redirection |
| `WriteConsoleA/W`                     | BeaconPrintf redirection |
| `__stdio_common_vfprintf`             | UCRT print functions |
| `ExitProcess` / `exit`                | Converted to ExitThread |

## Evasion Techniques

| Technique | Bypasses |
|-----------|----------|
| Indirect Syscalls         | Userland API hooks (EDR/AV) |
| Draugr Stack Spoofing     | Call stack inspection |
| Thread Start Spoofing     | Thread start address analysis |
| Module Stomping           | Unbacked memory detection |
| Private Heap Allocation   | VirtualAlloc monitoring |
| Ntdll Unhooking           | Overwrite in memory ntdll with Ntdll on a disk |
| IAT Hooking (no pipes)    | Named pipe monitoring |

## Detection Vectors

### Kernel Telemetry (ETW-TI)

NtGetContextThread / NtSetContextThread:
- Thread context manipulation on suspended threads then resume it


Memory Operations:
- NtAllocateMemory allocation, can be in RWX (depend with config)
- NtProtectVirtualMemory transitions (RW → RX) 
- Executable memory in heap regions is suspicious (depend with config)
- Module stomping detectable via section hash mismatch (depend with config)

### Behavioral Indicators

- Suspended thread created, take the context then change the value of RIP
- Heap memory marked as executable (if memory allocator is heap)
- DLL loaded with `DONT_RESOLVE_DLL_REFERENCES` (if memory allocator is module stomping)
- Ntdll .text section modified (if unhooking enabled)

## Usage

### Loading the Script

```
Cobalt Strike → Script Manager → Load → BOF_RunPe.cna
```

### Aggressor Commands

```
beacon> runpe /path/to/binary.exe --arg1 value1
```

![Mimikatz](/img/mimikatz_coffee.png)

```
beacon> help runpe
```

![Help](/img/runpe_help.png)

## Compilation

Requires GCC 13 (mingw-w64). Use provided Dockerfile:

```bash
sudo docker build -t ubuntu-gcc-13 .
sudo docker run --rm -it -v "$PWD":/work -w /work ubuntu-gcc-13:latest make
 ```


Output: `Bin/runpe.o`

## Limitations

| Limitation | Description |
|------------|-------------|
| **CET** | Control-flow Enforcement Technology may block synthetic stack frames |
| **x64 Only** | No x86/WoW64 support |
| **Kernel Visibility** | Thread creation visible to kernel callbacks |
| **.NET** | Managed executables not supported  |


## Credits / Ressources use for dev

## Repos/blogpost
- https://github.com/susMdT/LoudSunRun
- https://github.com/Octoberfest7/Inline-Execute-PE
- https://www.coresecurity.com/core-labs/articles/running-pes-inline-without-console
- https://0xdarkvortex.dev/proxying-dll-loads-for-hiding-etwti-stack-tracing/

## Books
- `Windows Native API Programming` by *Pavel Yosifovich*
- `Windows Internals, Part 1` by *Pavel Yosifovich*
- `Windows Internals, Part 2` by *Andrea Allievi*

