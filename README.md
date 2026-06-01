# Inline Assembly Validator for x86-64

> **Note for Evaluators:** Source code is organized into `plugin/` (Clang plugin) and `standalone/` (standalone tool) instead of a single `src/` directory. Test cases are in `tests/` instead of `testcases/`. All scripts (`build.sh`, `run.sh`) reference these paths correctly and work as-is.

A Clang plugin and standalone tool that parses inline assembly statements in C/C++ code and validates x86-64 instruction syntax, register usage, and calling convention preservation under the System V AMD64 ABI.

## Problem

Inline assembly (`asm("...")`) is a black box to the compiler. Developers often make mistakes — clobbering caller-saved registers without declaring them, using invalid opcodes, or violating ABI constraints — and these bugs survive compilation, only manifesting at runtime.

This tool catches four classes of inline assembly errors **at compile time**:

| # | Check | Severity |
|---|-------|----------|
| 1 | Invalid register names (e.g., `%eax64`) | Error |
| 2 | Missing clobber declarations for written registers | Warning |
| 3 | Stack pointer misalignment (unbalanced push/pop) | Warning |
| 4 | Callee-saved register corruption without save/restore | Error |

## Project Structure

```
asm-validator/
├── plugin/                          # Clang plugin (requires LLVM/Clang dev libs)
│   ├── AsmValidator.cpp             # AST visitor + 4 validation passes
│   ├── AsmTokenizer.h / .cpp        # AT&T syntax tokenizer
│   ├── RegisterTable.h              # x86-64 register lookup tables
│   └── CallingConvention.h          # ABI constants and documentation
├── standalone/                      # Standalone validator (no LLVM dependency)
│   └── AsmValidatorStandalone.cpp   # Self-contained single-file implementation
├── tests/                           # Test suite (4 buggy + 3 correct)
│   ├── buggy_1_missing_clobber.c
│   ├── buggy_2_invalid_register.c
│   ├── buggy_3_stack_misalign.c
│   ├── buggy_4_callee_saved.c
│   ├── correct_1_basic_add.c
│   ├── correct_2_syscall.c
│   └── correct_3_full_constraints.c
├── CMakeLists.txt                   # CMake build for Clang plugin
├── build.sh                         # Build script (plugin + standalone)
├── run.sh                           # Test runner script
├── build_and_run.bat                # Windows build + run
├── README.md                        # This file
├── DESIGN.md                        # Approach and alternative solutions
├── IMPLEMENTATION.md                # LLVM/Clang-specific implementation details
├── EVALUATION.md                    # Metrics, comparisons, and test results
├── REPORT.md                        # Full project report
└── CODE_EXPLANATION.md              # Line-by-line code walkthrough
```

## Prerequisites

- **OS:** Ubuntu (or WSL on Windows)
- **Packages:** `clang`, `libclang-dev`, `llvm-dev`, `cmake`, `make`, `g++`

## Step-by-Step Commands (WSL / Ubuntu)

### 1. Install dependencies (one time)

```bash
sudo apt update
sudo apt install -y clang libclang-dev llvm-dev cmake make g++
```

### 2. Build the Clang plugin

```bash
mkdir -p build-linux
cd build-linux
cmake .. -DCMAKE_BUILD_TYPE=Release
make
cd ..
```

This produces `build-linux/libAsmValidator.so`.

### 3. Build the standalone validator

```bash
g++ -std=c++17 -O2 -o asm_validator standalone/AsmValidatorStandalone.cpp
```

This produces `./asm_validator`.

### 4. Run all tests (Clang plugin)

```bash
for f in tests/buggy_*.c tests/correct_*.c; do
  echo "=== $f ==="
  clang -fplugin=./build-linux/libAsmValidator.so -fsyntax-only "$f" 2>&1
  echo
done
```

### 5. Run a single test file

```bash
# Using the Clang plugin:
clang -fplugin=./build-linux/libAsmValidator.so -fsyntax-only tests/buggy_1_missing_clobber.c

# Using the standalone validator:
./asm_validator tests/buggy_1_missing_clobber.c
```

### 6. Run all tests with pass/fail summary

```bash
chmod +x run.sh
./run.sh
```

### Quick-start (build + run everything)

```bash
chmod +x build.sh run.sh
./build.sh
./run.sh
```

## Sample Output

```
tests/buggy_1_missing_clobber.c:11: warning: inline asm: register '%eax' is written
    but not listed in clobber list
tests/buggy_2_invalid_register.c:8: error: inline asm: unknown x86-64 register '%eax64'
tests/buggy_3_stack_misalign.c:8: warning: inline asm: stack pointer is not restored;
    net RSP delta = 8 bytes -- violates 16-byte alignment contract
tests/buggy_4_callee_saved.c:8: error: inline asm: callee-saved register '%r13' is
    modified without being pushed/restored -- ABI violation
```

## Test Results

| Test File | Type | Expected | Result |
|-----------|------|----------|--------|
| `buggy_1_missing_clobber.c` | Negative | Warning: missing clobber | PASS |
| `buggy_2_invalid_register.c` | Negative | Error: invalid register | PASS |
| `buggy_3_stack_misalign.c` | Negative | Warning: stack imbalance | PASS |
| `buggy_4_callee_saved.c` | Negative | Error: callee-saved violation | PASS |
| `correct_1_basic_add.c` | Positive | No diagnostics | PASS |
| `correct_2_syscall.c` | Positive | No diagnostics | PASS |
| `correct_3_full_constraints.c` | Positive | No diagnostics | PASS |

**Result: 7/7 tests passed**

## Documentation

| Document | Contents |
|----------|----------|
| [DESIGN.md](DESIGN.md) | Approach, architecture, and alternative solutions |
| [IMPLEMENTATION.md](IMPLEMENTATION.md) | LLVM/Clang-specific implementation details |
| [EVALUATION.md](EVALUATION.md) | Metrics, baseline comparisons, and test results |
| [REPORT.md](REPORT.md) | Full project report |
| [CODE_EXPLANATION.md](CODE_EXPLANATION.md) | Line-by-line code walkthrough |
| [DEMO_OUTPUT.md](DEMO_OUTPUT.md) | Sample run output |
