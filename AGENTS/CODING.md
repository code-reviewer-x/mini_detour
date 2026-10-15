# Coding Rules

These rules apply to code implementation and code review. They supplement the shared instructions in the root
`AGENTS.md`. Code that does not follow them is a bug to fix, not an example to copy.

**mini_detour** is a minimalistic cross-platform function hooking library: inline hooks, export and import table
replacement, and memory helpers. It supports Windows, Linux, and macOS; x86, x64, ARM (Thumb), and ARM64. It has a C
API with a thin C++ wrapper and one vendored dependency, Capstone.

## Build and tests

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug -DMINIDETOUR_BUILD_TESTS=ON
cmake --build build --parallel 4
```

- Run `tests` from its output directory (`build/tests`, or `build/<Config>/tests.exe` with Visual Studio): it loads
  `export_hook_test_library` from the working directory. On macOS, run `./macos_permission_fix tests` after every
  relink and before `./tests`.
- The test build (`mini_detour_test`) uses `USE_SPDLOG`. Only warnings are shown unless
  `MINIDETOUR_TEST_LOG_LEVEL=info` (or `debug`) is set. Run one test subject with a tag, for example
  `./tests "[hook_twice]"`.
- CI (`.github/workflows/main.yml`) tests Debug and Release on Windows x86/x64/ARM64, Linux x86/x64/ARM64, Linux ARM
  (cross-compiled and run through the ARM loader on an ARM64 runner), and macOS ARM64. macOS x64 is tested locally.
- A code change must build without warnings (`/W4`, `-Wall -Wextra` on the library) and pass tests in Debug and
  Release on every platform and architecture it affects before handoff. Report any configuration that could not be
  run.

## Repository layout

- `include/mini_detour/mini_detour.h`: the only public header (C API and inline C++ wrapper).
- `src/mini_detour.cpp`: the single translation unit. It includes one architecture header and one platform header:
  - `src/mini_detour_<arch>.h` (`x86`, `x64`, `arm`, `arm64`): jump encodings, thunk following, and relocation.
  - `src/mini_detour_<os>.h` (`windows`, `linux`, `macos`): memory, thread freezing, and module imports/exports.
  - `src/capstone_arch.h`: force-included in Capstone sources so only the target disassembler is built.
- `tests/`: Catch2 tests (`test.cpp` holds `main`, one `test_<subject>.cpp` per subject), `test_utils.h`, test
  libraries, toolchains, and CMake consumer projects.
- Vendored files are not modified: `deps/capstone/`, `include/spdlog/`, and `tests/catch.hpp`.

## Architecture rules

### Architecture and platform headers

- An architecture header defines `TrampolineSlot_t`, `AbsJump_t`, `RelJump_t` (`WriteOpcodes`, `GetOpcodeSize`, and
  `constexpr GetMaxOpcodeSize`), `_AddressesAreRelativeJumpable`, `_GetCodeMode`, `_GetCodeAddress`,
  `_GetCallAddress`, `_InitDisasm`, `_EnterRecursiveThunk`, `_GetRelocatableSize`, and `_RelocateCode`.
- A platform header defines `OtherThreadsFreezer_t` (`Freeze`, `Unfreeze`), the `_Query*` functions used by
  `Globals_t`, the `MemoryManipulation::Implementation` entry points (`GetRegionInfos`, `GetAllRegions`,
  `GetFreeRegions`, `MemoryProtect`, `MemoryFree`, `_MemoryAllocAt`, `_MemoryAllocAnywhere`, `SafeMemoryRead`,
  `SafeMemoryWrite`, `FlushInstructionCache`), and the `ModuleManipulation::Implementation` entry points
  (`_QueryModuleFunctions`, `GetAllExportedSymbols`, `GetAllIATSymbols`, `Replace/RestoreModuleExports`,
  `Replace/RestoreModuleIATs`).
- A platform without writable and executable memory defines `MINIDETOUR_JIT_CODE_MEMORY` and its `_JitCode*` /
  `_WriteFunctionCodeByRemap` functions (macOS).
- A helper has the same name on every platform (`_MemoryProtectRightsToNative`, `_NativeToMemoryRights`, and so on),
  and a concept has the same name everywhere (`dynamicSymbolsNames`, not `dynStr` in one function).
- Platform and architecture specifics stay in their headers. `#if defined(MINIDETOUR_OS_*)` in `mini_detour.cpp` is
  only for declarations needed before the platform header is included (`ModuleFunctions_t`).
- A behavior change shared by multiple platforms or architectures is implemented for all of them, or the change
  explicitly documents which ones are not covered.

### Code and table writes

- Code owned by the library (jump slots and trampolines) comes from `AllocCode` and is written only between
  `BeginCodeWrite` and `EndCodeWrite`; `EndCodeWrite` flushes the instruction cache.
- Code not owned by the library (the hooked function) is written through `FunctionCodeWriter_t`: call `Prepare` (which
  may allocate), then `Write` while `OtherThreadsFreezer_t` holds the other threads, then `Finish`. The writer keeps
  the page's original rights.
- Never make code `mem_rwx` with `MemoryProtect` directly: Apple Silicon and the hardened runtime refuse writable and
  executable memory.
- Import slots and export entries are written with `_ExchangeTableEntry`, which keeps page rights and never adds
  execute permission; on macOS, use `_WriteProtected`.
- Between `Freeze` and `Unfreeze`, do not allocate on the heap, log, or call a function that could be hooked. A
  suspended thread can hold the heap lock or a lock needed by the detour. Resume every thread before releasing any
  handle or port.
- C entry points that write code or tables take `Globals_t::HookMutex`. The corresponding `Implementation` functions
  do not.

### Correctness

- Refusing to hook is better than generating incorrect code. Relocation either handles or explicitly refuses every
  instruction form it can encounter. Rewrite an instruction that cannot be copied as-is (IP-relative or branch) in a
  dedicated case, or stop relocation.
- Never write past a slot: trampolines fit in slot data, jumps fit in `AbsJump_t::GetMaxOpcodeSize()`, and code reads
  fit in local buffers. Read code near the end of a mapping through `_ReadCode` (safe read).
- Thunk following is bounded to 8 jumps; a thunk that jumps to itself must not hang.
- Check every OS return value (`mmap` returns `MAP_FAILED`, `VirtualAlloc` returns `nullptr`, and `mach_vm_*` returns a
  `kern_return_t`).
- A failed operation leaves no partial state: slots are freed and rights are restored. Use objects whose destructor
  or `Finish` undoes acquired resources (RAII), not `goto` cleanup.

### Loader functions

- Call loader functions (`GetModuleHandleExA`, `GetProcAddress`, `GetModuleFileNameW`, `dlinfo`, `dlopen`, `dlclose`,
  `dlsym`) only through `Globals_t::ModuleFunctions`, resolved once at startup. The imports of the image linking the
  library may be replaced by the user.
- Never take `&function` of an imported function in the library or tests: it can be the image's thunk, stub, or PLT
  entry. Ask the loader (`GetProcAddress`, `dlsym(RTLD_NEXT)` on Linux, `dlsym(RTLD_DEFAULT)` on macOS).

### Static data

- Do not add static objects. Process-lifetime state belongs in the singleton `MiniDetour::Globals_t`
  (`src/mini_detour.cpp`), constructed in declaration order:
  - objects with a constructor or destructor (mutex, memory manager);
  - values computed once and cached because they never change during the process (page size, address bounds, whether
    writable and executable memory exists, and loader functions).
- `constexpr` values and constant tables remain plain statics.
- Per-thread state is the only exception: use `static thread_local` next to the code that uses it and comment why it
  is per-thread (the macOS JIT write depth).
- A value needed for only one call stays a local variable.
- `Globals_t` is never destroyed (hooks with static storage are restored during static destruction). Its constructor
  must not call `Globals_t::Instance()`.

## Public API

- C first: every function is `MINIDETOUR_EXPORT(type) MiniDetour<Group><Function>(...)` (`Utils`,
  `MemoryManipulation`, `ModuleManipulation`, `HookT`); the C++ API (`MiniDetour::...`, `Hook_t`) is an `inline`
  wrapper over it. Keep both in sync.
- The header compiles as C and C++11. In C, booleans are `int` (1/0), never `bool` or `<stdbool.h>`
  (`tests/test_c_api.c` checks this). C prototypes without parameters use `(void)`.
- No exception crosses the C API (`new (std::nothrow)`). C entry points check pointers and counts before use.
- Preserve these patterns: a null output array returns the count only (`GetAllExportedSymbols(h, nullptr, 0)`), and that
  count matches the count written to a large enough array; output fields are reset at entry so a failed entry leaves
  them null; variable-size structures begin with `StructSize`.
- A parameter with the same meaning behaves the same on every platform (a null or empty module name matches any
  module). Document platform differences where the same meaning cannot be provided (ordinals).
- Never break the API or ABI unless the maintainer explicitly requests it.

## Documentation

- Document every public function and every public structure field on the C declaration in Doxygen form:

  ```c
  /// @brief Safely reads memory: unreadable or unallocated memory does not crash the process.
  /// @param address Where to read.
  /// @param buffer Receives the bytes read.
  /// @param size Number of bytes to read.
  /// @return The number of bytes read: size when everything was read. The copy stops at the first page that cannot be
  /// read.
  ```

- Structure fields state whether they are IN, OUT, or both for each function using them. Document per-platform
  differences. Do not leave `@brief`, `@param`, or `@return` empty.

## Coding style

### Format

- Run `clang-format -i` (`.clang-format`) on only the files changed. If the installed version changes untouched
  lines, revert those hunks.
- Use Allman braces, 4 spaces, no tabs (including comments), and 180 columns. Never reorder includes.
- Text files use LF (`.gitattributes`). CMake uses 2 spaces, no tabs, and no trailing spaces.

### Language

- The library uses C++11 (`CMAKE_CXX_STANDARD 11`). No C++14 in `src/` or `include/`, except code under
  `#ifdef USE_SPDLOG`, which is only built by the C++14 test build. Tests use C++14.
- Do not add dependencies.
- Logging uses only `SPDLOG_*` macros, defined as nothing without `USE_SPDLOG`; their arguments have no side effects.
  Do not wrap those macros in `#ifdef USE_SPDLOG`. Use `#ifdef USE_SPDLOG` only around code that exists for logging
  (such as building a string to log).
- Use C++ casts (`static_cast`, `reinterpret_cast`, `const_cast`), never C casts, including when parsing PE, ELF, or
  Mach-O structures.
- Use `nullptr`, never `NULL` or `0` for pointers. Use `std::min` / `std::max` without parentheses (`NOMINMAX` is
  defined).
- Use `#if defined(MINIDETOUR_OS_*)` / `MINIDETOUR_ARCH_*` for platforms and `#ifdef` for feature macros
  (`USE_SPDLOG`).
- Use early returns, not `goto`.
- Avoid magic numbers: add a comment naming each instruction encoding (`0xE9 // jmp rel32`) and named constants for
  flags or kinds (an enum of bits for jump kinds returned by `CodeDisasm_t::GetJumpType`).
- Functions and types local to the library are `static` (`static inline` is fine).

### Naming

| What | Convention | Examples |
|---|---|---|
| Types (classes, structs, enums, aliases) | `PascalCase_t`, never a leading `_` | `Globals_t`, `ModuleFunctions_t`, `ElfDyn_t` |
| Internal helper functions, private methods | Leading `_` + PascalCase | `_ReadCode`, `_GetCodeMode`, `_TryFreeze` |
| Implementation entry points, public methods | PascalCase, same name as the API | `MemoryProtect`, `GetRegionInfos`, `Freeze` |
| Public data members | PascalCase | `PageSize`, `HookMutex`, `Name`, `Slot` |
| Private data members | `_` + PascalCase | `_Threads`, `_SavedCode` |
| Locals and parameters (public C functions included) | camelCase | `relocatableSize`, `moduleHandle`, `addressHint` |
| Macros | `MINIDETOUR_*` (library), `TEST_*` (tests) | `MINIDETOUR_JIT_CODE_MEMORY` |

- On functions and members, a leading `_` means private or internal: a public member or method has no `_`, and an
  internal helper function always has one. Types never start with `_`; they always end with `_t`.
- Exceptions: public C names kept for compatibility (`mem_r`, `mem_rwx`, `MiniDetourMemoryManipulationMemoryRights`,
  `minidetour_hook_handle_t`); OS and file-format names (`Elf64_Dyn`, `PIMAGE_*`, `nbuckets`); and instruction field
  names from architecture manuals (`S`, `J1`, `imm10`).

### Comments

- Write English sentences in the present tense, without “we” or “I”. Explain why or what the system does, not what the
  next line does: `// VirtualAlloc treats NULL as "anywhere".`, `// mmap refuses the addresses below vm.mmap_min_addr.`
- Add a comment above every non-trivial function or type describing its purpose, contract, and relevant platform
  facts (loader behavior, OS version, encoding). Write instruction encodings next to their code.
- Join short explanations with a colon rather than “because”: `// Free memory has no rights, as in GetAllRegions.`
- Update comments made incorrect by a change. Do not leave commented-out code.
- A `TODO` states what is missing and why it is not done yet: `// TODO: <what>. <why not now>.`.

## Test conventions

- A bug fix includes a regression test that fails without the fix when it can be tested. A feature includes tests on
  every supported platform.
- Use one subject per file: `tests/test_<subject>.cpp` (or `tests/test_<os>_<subject>.cpp`), picked up by the CMake
  glob. `tests/test.cpp` only holds `main`. Every test in a file uses its subject as a snake_case tag
  (`test_hook_twice.cpp` -> `[hook_twice]`).
- Name tests with a sentence stating expected behavior (`"A hook object in use refuses to hook another function"`).
  Add a comment above each test describing what used to go wrong.
- Guard platform-specific tests with `#if defined(MINIDETOUR_OS_*)` / `MINIDETOUR_ARCH_*`; never skip them at runtime.
  Build a deterministic layout instead of replacing an assertion with a conditional `WARN`. Use `WARN` only for an
  environmental limit the test cannot control (such as qemu hiding its own mappings).
- Use `REQUIRE` for preconditions (library loaded, buffer allocated) and `CHECK` for behavior.
- Create hand-written code through `CodeBuffer_t` and `GeneratedCode` (`tests/test_utils.h`): never use writable and
  executable memory. Pass Thumb addresses through `GeneratedCode::FunctionAddress`.
- A test replacing a system function forwards to the real one (obtained from the loader) and calls the replaced
  imports itself through a `noinline` helper so the optimizer keeps the import.
- Add new test libraries with `minidetour_add_test_library` in `CMakeLists.txt`.
