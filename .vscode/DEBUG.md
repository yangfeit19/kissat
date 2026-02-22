# Kissat Debugging Guide

## Debug Configuration

This project has been configured for debugging in VSCode with the following setups:

### Debug Configurations

1. **Debug Kissat with Test Case**
   - Uses the default test case `test/cnf/and1.cnf`
   - Press F5 to start debugging

2. **Debug Kissat with Custom Test**
   - Prompts for a custom CNF file path
   - Select from the debug configurations dropdown

3. **Debug Tissat (Test Suite)**
   - Debugs the test suite program
   - Useful for testing internal functions

4. **Attach to Kissat Process**
   - Attaches to an already running Kissat process

### Build Tasks

- **build-kissat-debug**: Builds the debug version of Kissat
- **build-tissat-debug**: Builds the debug version of Tissat
- **build-kissat-release**: Builds the optimized release version
- **run-tests**: Runs the test suite
- **clean-build**: Cleans build artifacts

### Test Cases

Test cases are located in the `test/cnf/` directory:
- `and1.cnf`, `and2.cnf`, `and3.cnf` - AND gate tests
- `xor*.cnf` - XOR gate tests
- `prime*.cnf` - Prime number tests
- `unit*.cnf` - Unit clause tests

### Debugging Tips

1. To debug the AND gate detection code in `ands.c`:
   - Set a breakpoint at line 26 in `ands.c`
   - Use the "Debug Kissat with Test Case" configuration
   - The `and1.cnf` test case contains AND gates that will trigger this code

2. To view variable values during debugging:
   - Use the "Variables" panel in VSCode
   - Add expressions to the "Watch" panel

3. To step through the SAT solving process:
   - Set breakpoints in `search.c` for the main search loop
   - Set breakpoints in `analyze.c` for conflict analysis

### Common Issues

If you encounter issues with debugging:

1. Make sure the project is built with debug symbols:
   ```
   ./configure -g
   make
   ```

2. Ensure the executable files exist:
   - `build/kissat` for the main solver
   - `build/tissat` for the test suite

3. Check that GDB is installed on your system:
   ```
   which gdb
   ```

### Customizing Debug Configuration

To modify the debug configuration:
1. Edit `.vscode/launch.json`
2. Change the `args` array to modify command-line arguments
3. Update the `program` path if needed

For more information about Kissat, see the README.md file.