# Call Stack Spoofing

A proof-of-concept implementation of synthetic call stack spoofing on Windows x64. The goal is to mask the true origin of a syscall from EDR inspection by replacing the real call stack with a fake one built entirely from legitimate ntdll.dll gadgets.

## How It Works

When a syscall fires, the call stack reveals where it came from. EDRs watch this. This implementation tears down the real stack and plants a synthetic one made of return addresses resolved from deep inside ntdll — so when anything walks the stack, it sees only Windows doing Windows things.

The synthetic chain is built from the following ntdll functions:

- `LdrInitializeThunk`
- `LdrInitShimEngineDynamic`
- `RtlRegisterSecureMemoryCacheCallback`
- `RtlCreateHeap`

Stack frame sizes are computed by parsing the unwind information from the `.pdata` section of ntdll. Each return address is placed at the correct offset so the stack is structurally valid, not just visually convincing.

The syscall executes inside a dedicated thread. A return gadget at the tail of `RtlCreateHeap` redirects execution back to the thread after the syscall completes. The status code is retrieved from `EAX` and the original stack is restored cleanly.

## What Gets Called

`NtAllocateVirtualMemory` — allocates a 4096 byte region in the current process with `PAGE_EXECUTE_READWRITE` protection.

## Files

| File | Description |
|------|-------------|
| `main.c` | Entry point, thread creation and result handling |
| `NtAllocateVirtualMemory.c` | Core implementation — unwind parsing, stack construction, syscall thread |
| `NtAllocateVirtualMemory.h` | Type definitions and function declarations |
| `syscall.asm` | NASM assembly — argument setup and syscall dispatch |

## Build

Requires MinGW and NASM on Windows.

```bash
nasm -f win64 syscall.asm -o syscall.o
gcc main.c NtAllocateVirtualMemory.c syscall.o -o spoof.exe -lkernel32
```

## Note

This is for educational and research purposes. The call chain is stable — no exceptions occur during normal execution.
