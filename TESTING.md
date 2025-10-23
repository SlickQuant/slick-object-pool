# Testing Guide - slick_object_pool

## Overview

This document covers testing strategies, sanitizer usage, and best practices for validating the object pool implementation.

## Table of Contents

- [Quick Start](#quick-start)
- [Sanitizers](#sanitizers)
  - [AddressSanitizer (ASan)](#addresssanitizer-asan)
  - [ThreadSanitizer (TSan)](#threadsanitizer-tsan)
  - [UndefinedBehaviorSanitizer (UBSan)](#undefinedbehaviorsanitizer-ubsan)
- [Platform-Specific Notes](#platform-specific-notes)
- [CI/CD Integration](#cicd-integration)
- [Troubleshooting](#troubleshooting)

## Quick Start

### Basic Testing

```bash
# Build and run tests
mkdir build && cd build
cmake .. -DBUILD_SLICK_OBJECTPOOL_TESTS=ON
cmake --build .
ctest --output-on-failure
```

### With AddressSanitizer

```bash
# Linux/macOS
mkdir build-asan && cd build-asan
cmake .. -DBUILD_SLICK_OBJECTPOOL_TESTS=ON -DENABLE_ASAN=ON
cmake --build .
ctest --output-on-failure

# Windows (MSVC)
mkdir build-asan && cd build-asan
cmake .. -DBUILD_SLICK_OBJECTPOOL_TESTS=ON -DENABLE_ASAN=ON
cmake --build .
ctest --output-on-failure
```

### With ThreadSanitizer

```bash
# Linux/macOS only
mkdir build-tsan && cd build-tsan
cmake .. -DBUILD_SLICK_OBJECTPOOL_TESTS=ON -DENABLE_TSAN=ON
cmake --build .
ctest --output-on-failure
```

## Sanitizers

### AddressSanitizer (ASan)

**Purpose:** Detects memory errors

**Detects:**
- ✅ Use-after-free
- ✅ Heap buffer overflow
- ✅ Stack buffer overflow
- ✅ Global buffer overflow
- ✅ Use-after-return
- ✅ Use-after-scope
- ✅ Memory leaks
- ✅ Double-free

#### Enable ASan

**CMake:**
```bash
cmake .. -DENABLE_ASAN=ON
```

**Manual (GCC/Clang):**
```bash
g++ -fsanitize=address -fno-omit-frame-pointer -g test.cpp -o test
```

**Manual (MSVC):**
```bash
cl /fsanitize=address test.cpp
```

#### ASan Output Example

```
=================================================================
==12345==ERROR: AddressSanitizer: heap-use-after-free on address 0x603000000010
READ of size 4 at 0x603000000010 thread T0
    #0 0x4a5678 in MyClass::use() test.cpp:42
    #1 0x4a5890 in main test.cpp:55

0x603000000010 is located 0 bytes inside of 16-byte region [0x603000000010,0x603000000020)
freed by thread T0 here:
    #0 0x7f123456789a in operator delete(void*)
    #1 0x4a5823 in MyClass::destroy() test.cpp:38
```

#### ASan Environment Variables

```bash
# Detect memory leaks
export ASAN_OPTIONS=detect_leaks=1

# More verbose output
export ASAN_OPTIONS=verbosity=1

# Fast unwinding for better performance
export ASAN_OPTIONS=fast_unwind_on_malloc=1

# Multiple options
export ASAN_OPTIONS=detect_leaks=1:verbosity=1
```

### ThreadSanitizer (TSan)

**Purpose:** Detects data races and threading issues

**Detects:**
- ✅ Data races
- ✅ Deadlocks
- ✅ Thread leaks
- ✅ Unprotected concurrent access
- ✅ Lock order inversions

**Important:** Cannot be used with ASan simultaneously!

#### Enable TSan

**CMake:**
```bash
cmake .. -DENABLE_TSAN=ON
```

**Manual (GCC/Clang):**
```bash
g++ -fsanitize=thread -fno-omit-frame-pointer -g test.cpp -o test
```

**Not available on MSVC**

#### TSan Output Example

```
==================
WARNING: ThreadSanitizer: data race (pid=12345)
  Write of size 4 at 0x7b0000000000 by thread T1:
    #0 ObjectPool<int>::allocate_object() object_pool.h:367
    #1 worker_thread() test.cpp:145

  Previous read of size 4 at 0x7b0000000000 by main thread:
    #0 ObjectPool<int>::free_object() object_pool.h:413
    #1 main test.cpp:160

  Location is heap block of size 4 at 0x7b0000000000
```

#### TSan Environment Variables

```bash
# Suppress known issues
export TSAN_OPTIONS=suppressions=tsan.supp

# Second deadlock detection
export TSAN_OPTIONS=second_deadlock_stack=1

# Multiple options
export TSAN_OPTIONS=second_deadlock_stack=1:halt_on_error=1
```

#### TSan Suppressions File

Create `tsan.supp`:
```
# Suppress known false positives
race:std::__1::vector
deadlock:gtest_main
```

### UndefinedBehaviorSanitizer (UBSan)

**Purpose:** Detects undefined behavior

**Detects:**
- ✅ Integer overflow
- ✅ Null pointer dereference
- ✅ Misaligned pointer usage
- ✅ Signed integer overflow
- ✅ Division by zero
- ✅ Invalid shifts
- ✅ Out-of-bounds array access

#### Enable UBSan

**CMake:**
```bash
cmake .. -DENABLE_UBSAN=ON
```

**Manual (GCC/Clang):**
```bash
g++ -fsanitize=undefined -fno-omit-frame-pointer -g test.cpp -o test
```

**Not available on MSVC**

#### UBSan Output Example

```
test.cpp:42:15: runtime error: signed integer overflow:
  2147483647 + 1 cannot be represented in type 'int'
```

#### UBSan Environment Variables

```bash
# Print stack traces
export UBSAN_OPTIONS=print_stacktrace=1

# Halt on first error
export UBSAN_OPTIONS=halt_on_error=1
```

## Platform-Specific Notes

### Windows (MSVC)

**Supported:**
- ✅ AddressSanitizer (ASan)

**Not Supported:**
- ❌ ThreadSanitizer (TSan)
- ❌ UndefinedBehaviorSanitizer (UBSan)

**Requirements:**
- Visual Studio 2019 16.9 or later
- Windows 10 SDK

**Build Command:**
```powershell
cmake .. -DBUILD_SLICK_OBJECTPOOL_TESTS=ON -DENABLE_ASAN=ON
cmake --build . --config Debug
```

**Run Tests:**

⚠️ **Known Issue:** AddressSanitizer on Windows has interception failures with GTest discovery.

**Workaround - Option 1 (Manual execution):**
```powershell
cd build\tests\Debug
set ASAN_WIN_CONTINUE_ON_INTERCEPTION_FAILURE=1
slick_object_pool_tests.exe
```

**Workaround - Option 2 (CTest with environment variable):**
```powershell
set ASAN_WIN_CONTINUE_ON_INTERCEPTION_FAILURE=1
ctest -C Debug --output-on-failure
```

**Workaround - Option 3 (PowerShell):**
```powershell
$env:ASAN_WIN_CONTINUE_ON_INTERCEPTION_FAILURE=1
ctest -C Debug --output-on-failure
```

### Linux (GCC/Clang)

**Supported:**
- ✅ AddressSanitizer (ASan)
- ✅ ThreadSanitizer (TSan)
- ✅ UndefinedBehaviorSanitizer (UBSan)

**Requirements:**
- GCC 4.8+ or Clang 3.1+
- For TSan: GCC 7+ or Clang 3.2+

**Install Dependencies:**
```bash
# Ubuntu/Debian
sudo apt-get install build-essential cmake libgtest-dev

# Build GTest
cd /usr/src/gtest
sudo cmake .
sudo make
sudo cp lib/*.a /usr/lib
```

**Build Command:**
```bash
# ASan
cmake .. -DCMAKE_BUILD_TYPE=Debug -DENABLE_ASAN=ON

# TSan
cmake .. -DCMAKE_BUILD_TYPE=Debug -DENABLE_TSAN=ON

# UBSan
cmake .. -DCMAKE_BUILD_TYPE=Debug -DENABLE_UBSAN=ON

# Combine ASan + UBSan
cmake .. -DCMAKE_BUILD_TYPE=Debug -DENABLE_ASAN=ON -DENABLE_UBSAN=ON
```

### macOS (Clang)

**Supported:**
- ✅ AddressSanitizer (ASan)
- ✅ ThreadSanitizer (TSan)
- ✅ UndefinedBehaviorSanitizer (UBSan)

**Requirements:**
- Xcode Command Line Tools
- Homebrew (for GTest)

**Install Dependencies:**
```bash
brew install cmake googletest
```

**Build Command:**
```bash
cmake .. -DCMAKE_BUILD_TYPE=Debug -DENABLE_ASAN=ON
cmake --build .
```

## Complete Testing Matrix

### Recommended Test Combinations

| Configuration | Platform | Purpose |
|--------------|----------|---------|
| No sanitizers | All | Basic functionality |
| ASan | All | Memory safety |
| TSan | Linux/macOS | Thread safety |
| ASan + UBSan | Linux/macOS | Memory + UB |
| Release build | All | Performance |

### Full Test Suite

```bash
#!/bin/bash
# Run complete test suite

# Clean build
rm -rf build-*

# 1. Basic tests
mkdir build-basic && cd build-basic
cmake .. -DBUILD_SLICK_OBJECTPOOL_TESTS=ON
cmake --build .
ctest --output-on-failure
cd ..

# 2. AddressSanitizer
mkdir build-asan && cd build-asan
cmake .. -DENABLE_ASAN=ON
cmake --build .
ctest --output-on-failure
cd ..

# 3. ThreadSanitizer (Linux/macOS only)
if [[ "$OSTYPE" != "msys" && "$OSTYPE" != "win32" ]]; then
    mkdir build-tsan && cd build-tsan
    cmake .. -DENABLE_TSAN=ON
    cmake --build .
    ctest --output-on-failure
    cd ..
fi

# 4. UndefinedBehaviorSanitizer (Linux/macOS only)
if [[ "$OSTYPE" != "msys" && "$OSTYPE" != "win32" ]]; then
    mkdir build-ubsan && cd build-ubsan
    cmake .. -DENABLE_UBSAN=ON
    cmake --build .
    ctest --output-on-failure
    cd ..
fi

# 5. Release build (performance check)
mkdir build-release && cd build-release
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build .
ctest --output-on-failure
cd ..

echo "All test configurations passed!"
```

## CI/CD Integration

### GitHub Actions Example

```yaml
name: CI Tests

on: [push, pull_request]

jobs:
  test-asan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y cmake libgtest-dev
      - name: Build with ASan
        run: |
          mkdir build && cd build
          cmake .. -DENABLE_ASAN=ON
          cmake --build .
      - name: Run tests
        run: |
          cd build
          ctest --output-on-failure

  test-tsan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y cmake libgtest-dev
      - name: Build with TSan
        run: |
          mkdir build && cd build
          cmake .. -DENABLE_TSAN=ON
          cmake --build .
      - name: Run tests
        run: |
          cd build
          ctest --output-on-failure

  test-windows:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v2
      - name: Build with ASan
        run: |
          mkdir build && cd build
          cmake .. -DENABLE_ASAN=ON
          cmake --build . --config Debug
      - name: Run tests
        run: |
          cd build
          ctest -C Debug --output-on-failure
```

## Troubleshooting

### Windows ASan Interception Failure

**Problem:** Error during build: `AddressSanitizer: CHECK failed: interception_win.cpp:171`

**Cause:** MSVC's AddressSanitizer has known issues with GTest's automatic test discovery phase.

**Solutions:**

1. **Run tests manually with environment variable:**
   ```powershell
   cd build\tests\Debug
   set ASAN_WIN_CONTINUE_ON_INTERCEPTION_FAILURE=1
   slick_object_pool_tests.exe
   ```

2. **Build without ASan for automated testing:**
   ```powershell
   cmake .. -DENABLE_ASAN=OFF
   cmake --build .
   ctest --output-on-failure
   ```

3. **Use Linux/WSL for ASan testing:**
   - ASan works perfectly on Linux without interception issues
   - Consider using WSL2 for Windows development

**Note:** This is a Microsoft ASan implementation limitation, not a bug in your code. The CMake configuration automatically handles this by:
- Setting the environment variable for CTest
- Disabling GTest discovery when ASan is enabled on Windows
- Showing a warning message with instructions

### ASan Reports Memory Leaks

**Problem:** ASan reports leaks in third-party code (GTest, system libraries)

**Solution:** Use leak suppression
```bash
export LSAN_OPTIONS=suppressions=lsan.supp
```

Create `lsan.supp`:
```
# Suppress GTest leaks
leak:testing::internal
```

### TSan Reports False Positives

**Problem:** TSan reports data races in lock-free code

**Solution:** This is expected for lock-free algorithms. Verify that:
1. Atomic operations use correct memory ordering
2. No actual data races exist (check algorithm carefully)
3. Use TSan suppressions if needed

### Performance Impact

**ASan:** ~2x slowdown
**TSan:** ~5-15x slowdown
**UBSan:** Minimal overhead

**Recommendation:** Use sanitizers in development/CI, not production

### Symbol Resolution Issues

**Problem:** Stack traces show `???` instead of function names

**Solution:**
```bash
# Ensure debug symbols
cmake .. -DCMAKE_BUILD_TYPE=Debug

# Or with sanitizers
cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo
```

## Best Practices

### 1. Test Early and Often
- Run sanitizers during development
- Don't wait until CI fails

### 2. Fix Issues Immediately
- Address sanitizer warnings promptly
- Don't accumulate technical debt

### 3. Use Multiple Sanitizers
- ASan for memory issues
- TSan for threading issues
- UBSan for undefined behavior

### 4. Automate Testing
- Include sanitizers in CI pipeline
- Test on multiple platforms

### 5. Monitor Performance
- Benchmark without sanitizers
- Use sanitizers for correctness only

## Additional Resources

### Documentation
- [AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer)
- [ThreadSanitizer](https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual)
- [UBSan](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html)

### Tools
- [Valgrind](https://valgrind.org/) - Alternative memory checker (slower than ASan)
- [Helgrind](https://valgrind.org/docs/manual/hg-manual.html) - Thread error detector

---

**Last Updated:** 2025
**Maintainer:** SlickQuant
