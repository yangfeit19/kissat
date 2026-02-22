# AGENTS.md - Kissat SAT Solver Codebase Guide

This document provides guidance for AI coding agents working on the Kissat SAT solver codebase.

## Project Overview

Kissat is a "keep it simple and clean bare metal SAT solver" written in C. It is a port of CaDiCaL back to C with improved data structures and optimized algorithms.

## Build Commands

```bash
# Initial setup (required once)
./configure

# Build everything (library and solver)
make

# Build specific targets
make kissat          # Build main solver binary
make tissat          # Build test runner
make libkissat.a     # Build static library

# Clean build
make clean           # Removes generated files and build directory
```

## Test Commands

```bash
# Run all tests
make test

# Run tests from build directory
cd build && ./tissat

# Run specific tests by pattern matching
./build/tissat add        # Run tests containing "add" in function name
./build/tissat solve      # Run tests containing "solve" in function name
./build/tissat parse      # Run tests containing "parse" in function name

# Test runner options
./build/tissat -h         # Show usage help
./build/tissat -v         # Verbose mode (implies sequential)
./build/tissat -s         # Sequential (non-parallel) testing
./build/tissat -j4        # Use 4 parallel processes
./build/tissat -b         # Run big/thorough tests
```

## Code Formatting

```bash
# Format all C files
make format
```

Uses clang-format with settings in `.clang-format`:
- Column limit: 76 characters
- Space before parentheses: Always
- Space after C-style cast: true

## Configure Options

The `./configure` script supports various build configurations:

```bash
./configure -g          # Debug build (enables assertions, symbols, logging)
./configure -c          # Include assertion checking
./configure -l          # Include logging code
./configure -s          # Add debug symbols
./configure -p          # Pedantic compilation (-Werror -std=c99 --pedantic)
./configure --coverage  # Enable coverage instrumentation
./configure --profile   # Enable profiling (gprof)
./configure --quiet     # Disable messages
./configure --no-proofs # Disable proof support
```

## Code Style Guidelines

### File Organization

- Source files: `src/*.c` and `src/*.h`
- Test files: `test/test*.c` and `test/test*.h`
- Build output: `build/` directory

### Header Guards

Use `#ifndef` / `#define` / `#endif` pattern with `_filename_h_INCLUDED`:

```c
#ifndef _clause_h_INCLUDED
#define _clause_h_INCLUDED

// content

#endif
```

### Include Order

1. The corresponding header for the .c file (e.g., `analyze.h` first in `analyze.c`)
2. Other project headers in quotes
3. System headers in angle brackets

```c
#include "analyze.h"
#include "backtrack.h"
#include "inline.h"

#include <inttypes.h>
#include <stdbool.h>
```

### Naming Conventions

- **Functions**: lowercase with underscores, prefixed with `kissat_`
  ```c
  void kissat_add (kissat *solver, int lit);
  void kissat_backtrack (kissat *solver, unsigned level);
  ```

- **Types**: lowercase with underscores
  ```c
  typedef struct clause clause;
  typedef struct kissat kissat;
  ```

- **Macros**: UPPERCASE with underscores
  ```c
  #define INVALID_LIT UINT_MAX
  #define BEGIN_STACK(S) (S).begin
  ```

- **Variables**: lowercase with underscores
  ```c
  unsigned conflict_level;
  unsigned *lits;
  ```

- **Constants**: UPPERCASE or via enum
  ```c
  #define INVALID_LEVEL UINT_MAX
  ```

### Formatting Rules

- Maximum line width: 76 characters
- Space before function call parentheses
- Space after C-style cast
- Use `clang-format off` / `clang-format on` for macro definitions that need specific formatting

```c
// clang-format off

#define OPTIONS \
  OPTION (ands, 1, 0, 1, "extract and eliminate and gates") \
  OPTION (backbone, 1, 0, 2, "binary clause backbone")

// clang-format on
```

### Comments

- The codebase generally avoids inline comments
- Self-documenting code is preferred
- Comments are used sparingly, typically for section markers or complex logic explanations

### Error Handling

Use the provided error functions:

```c
// Non-fatal error (prints message, continues)
kissat_error ("format error in line %d", line);

// Fatal error (prints message, exits)
kissat_fatal ("out of memory allocating %zu bytes", size);
```

In test code, use the test framework macros:

```c
assert (condition);           // Test assertion (aborts on failure)
assume (condition);           // Test assumption (warns on failure)
FATAL ("message %d", value);  // Fatal error in test
```

### Memory Management

Use the custom allocation functions:

```c
void *kissat_malloc (kissat *solver, size_t bytes);
void *kissat_calloc (kissat *solver, size_t n, size_t size);
void kissat_free (kissat *solver, void *ptr, size_t bytes);

// Convenience macros
NALLOC (ptr, n);   // kissat_nalloc
CALLOC (ptr, n);   // kissat_calloc
DEALLOC (ptr, n);  // kissat_dealloc
```

### Data Structures

Common patterns used throughout:

```c
// Stack-based containers
typedef STACK (unsigned) unsigneds;
typedef STACK (watch) statches;

// Iteration macros
#define all_variables(IDX) \
  unsigned IDX = 0, IDX##_END = solver->vars; \
  IDX != IDX##_END; \
  ++IDX

#define all_literals_in_clause(LIT, C) \
  unsigned LIT, *LIT##_PTR = BEGIN_LITS (C), *const LIT##_END = END_LITS (C); \
  LIT##_PTR != LIT##_END && ((LIT = *LIT##_PTR), true); \
  ++LIT##_PTR
```

## Test Structure

Tests are defined in `test/test*.c` files. Each test file follows this pattern:

```c
#include "test.h"

static void test_feature (void) {
  // Test implementation
  kissat *solver = kissat_init ();
  // ... test code ...
  kissat_release (solver);
}

void tissat_schedule_feature (void) {
  SCHEDULE_FUNCTION (test_feature);
}
```

The `tissat_schedule_*` function registers the test with the test runner. Tests can be filtered by pattern matching on the function name.

## Debugging

```bash
# Build with debug symbols and logging
./configure -g
make

# Run with verbosity
./build/kissat -v input.cnf

# Run with logging (requires -l configure option)
./build/kissat -l input.cnf
```

## Important Files

- `src/kissat.h` - Public API header
- `src/internal.h` - Internal solver structure and includes
- `src/main.c` - Main entry point for the solver binary
- `test/test.c` - Test runner main file
- `test/test.h` - Test framework header
- `configure` - Build configuration script
- `makefile.in` - Makefile template
