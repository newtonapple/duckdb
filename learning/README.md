# DuckDB Learning Directory

Welcome to your DuckDB learning repository! This directory contains documentation and resources to help you learn and master DuckDB development.

## Purpose

This directory serves as a centralized knowledge base for:
- Understanding DuckDB architecture and design
- Learning C++ development practices specific to DuckDB
- Collecting useful code patterns and examples
- Documenting debugging techniques and troubleshooting tips
- Building expertise in specific subsystems

## Getting Started

Choose your learning path based on your role:

### Path 1: DuckDB User (Start Here if you're using DuckDB)
📊 **For end users, data analysts, and application developers:**

1. **[usage-overview.md](usage-overview.md)** - Get started with DuckDB CLI and basic SQL (30 min)
2. **[usage-sql-features.md](usage-sql-features.md)** - Learn DuckDB's SQL dialect and features (45 min)
3. **[usage-data-import-export.md](usage-data-import-export.md)** - Load and export data (30 min)
4. **[usage-client-apis.md](usage-client-apis.md)** - Use DuckDB from your favorite language (45 min)
5. **[usage-advanced-features.md](usage-advanced-features.md)** - Master advanced SQL (60 min)

**Total time**: ~3-4 hours

---

### Path 2: DuckDB Developer (Start Here if you're contributing code)
🔧 **For developers contributing to DuckDB:**

1. **[overview.md](overview.md)** - Project overview for developers (30-45 min)
   - What DuckDB is and how it works
   - Setting up your development environment
   - Build commands and testing
   - Architecture and code structure
   - C++ guidelines and conventions
   - Development workflow

2. **[architecture-deep-dive.md](architecture-deep-dive.md)** - Understand DuckDB's internals (60 min)
3. **[testing-guide.md](testing-guide.md)** - Learn testing practices (30 min)
4. **[debugging-tips.md](debugging-tips.md)** - Master debugging techniques (30 min)

**Total time**: ~2-3 hours to get started

---

### 2. Build and Test
After reading the overview, get hands-on:

```bash
# Navigate to the repository root
cd ..

# Build DuckDB in debug mode
make debug

# Run the fast unit tests
make unit

# Try the DuckDB shell
./build/debug/duckdb
```

### 3. Explore the Codebase
Once you've successfully built and tested, start exploring:

- Browse `src/` directory to understand the structure
- Look at simple functions in `src/function/scalar/`
- Read test files in `test/sql/` to see SQL logic tests
- Check out `CLAUDE.md` in the repository root for detailed guidelines

## Learning Resources

### DuckDB Usage Documentation (Start Here for Users!)

| Document | Description | Status |
|----------|-------------|--------|
| [usage-overview.md](usage-overview.md) | Getting started with DuckDB CLI and basic operations | ✅ Available |
| [usage-sql-features.md](usage-sql-features.md) | DuckDB SQL dialect and unique features reference | ✅ Available |
| [usage-data-import-export.md](usage-data-import-export.md) | Loading and exporting data (CSV, Parquet, JSON) | ✅ Available |
| [usage-client-apis.md](usage-client-apis.md) | Using DuckDB from Python, R, Java, Node.js, C/C++ | ✅ Available |
| [usage-advanced-features.md](usage-advanced-features.md) | Advanced SQL features (CTEs, window functions, nested types) | ✅ Available |

### Core Documentation (Essential for Developers)

| Document | Description | Status |
|----------|-------------|--------|
| [overview.md](overview.md) | Comprehensive project overview for new developers | ✅ Available |
| [architecture-deep-dive.md](architecture-deep-dive.md) | Detailed architecture and design patterns | ✅ Available |
| [testing-guide.md](testing-guide.md) | Complete guide to writing and running tests | ✅ Available |
| [debugging-tips.md](debugging-tips.md) | Debugging techniques and common issues | ✅ Available |
| [vectorized-execution.md](vectorized-execution.md) | Understanding DuckDB's vectorized engine | ✅ Available |

### Subsystem Guides (Deep Dives)

| Document | Description | Status |
|----------|-------------|--------|
| [parser-guide.md](parser-guide.md) | How the parser works and how to extend it | ✅ Available |
| [optimizer-guide.md](optimizer-guide.md) | Query optimization techniques and rules | ✅ Available |
| [storage-guide.md](storage-guide.md) | Storage layer and data structures | ✅ Available |
| [function-development.md](function-development.md) | How to add new SQL functions | ✅ Available |
| [extension-development.md](extension-development.md) | Creating and building extensions | ✅ Available |

### Advanced Topics (Internals)

| Document | Description | Status |
|----------|-------------|--------|
| [performance-optimization.md](performance-optimization.md) | Profiling and optimizing queries | ✅ Available |
| [transaction-internals.md](transaction-internals.md) | MVCC and transaction management | ✅ Available |
| [parallel-execution.md](parallel-execution.md) | Understanding parallel query execution | ✅ Available |

### Code Examples (Hands-On Walkthroughs)

| Document | Description | Status |
|----------|-------------|--------|
| [example-scalar-function.md](example-scalar-function.md) | Complete example of adding a scalar function | ✅ Available |
| [example-aggregate-function.md](example-aggregate-function.md) | Complete example of adding an aggregate function | ✅ Available |
| [example-operator.md](example-operator.md) | Complete example of adding a new operator | ✅ Available |

## How to Use This Directory

### For Learning
1. **Sequential learning**: Follow the documents in order, starting with overview.md
2. **Topic-based learning**: Jump to specific subsystem guides as needed
3. **Example-driven**: Look at code examples for practical patterns

### For Reference
- Keep this directory open while coding
- Refer back to guidelines when uncertain
- Use as a quick reference for build commands and conventions

### For Contributing
As you learn and discover useful information:
1. Create new markdown files for topics not yet covered
2. Add code examples with explanations
3. Document common pitfalls and solutions
4. Update the tables above to mark documents as available

## Recommended Learning Paths

### For Users: 2-Week Learning Journey

**Week 1: Core Usage**
- [ ] Day 1-2: Read [usage-overview.md](usage-overview.md) and try the CLI
- [ ] Day 3-4: Study [usage-sql-features.md](usage-sql-features.md) and practice queries
- [ ] Day 5: Work through [usage-data-import-export.md](usage-data-import-export.md) with your own data

**Week 2: Advanced Usage**
- [ ] Day 1-2: Explore [usage-client-apis.md](usage-client-apis.md) in your language
- [ ] Day 3-5: Master [usage-advanced-features.md](usage-advanced-features.md)
- [ ] Build a real project using DuckDB!

---

### For Developers: 4-Week Learning Journey

**Week 1: Foundations**
- [ ] Read [overview.md](overview.md)
- [ ] Set up development environment
- [ ] Build DuckDB in debug mode
- [ ] Run and understand the test suite
- [ ] Read through `CLAUDE.md` in repository root
- [ ] Study [architecture-deep-dive.md](architecture-deep-dive.md)

**Week 2: Development Skills**
- [ ] Master [testing-guide.md](testing-guide.md) - write your first test
- [ ] Learn [debugging-tips.md](debugging-tips.md) - debug a simple issue
- [ ] Understand [vectorized-execution.md](vectorized-execution.md)
- [ ] Explore the `src/` directory structure
- [ ] Read simple scalar functions in `src/function/scalar/string/`

**Week 3: First Contribution**
- [ ] Follow [example-scalar-function.md](example-scalar-function.md) - add a function!
- [ ] Or follow [example-aggregate-function.md](example-aggregate-function.md)
- [ ] Find a "good first issue" on GitHub
- [ ] Make a small code change
- [ ] Format and test your code
- [ ] Create your first PR

**Week 4+: Specialization**
- [ ] Pick a subsystem: parser, optimizer, storage, or execution
- [ ] Read the corresponding guide ([parser-guide.md](parser-guide.md), [optimizer-guide.md](optimizer-guide.md), [storage-guide.md](storage-guide.md))
- [ ] Study [performance-optimization.md](performance-optimization.md)
- [ ] Explore advanced topics ([transaction-internals.md](transaction-internals.md), [parallel-execution.md](parallel-execution.md))
- [ ] Build an extension following [extension-development.md](extension-development.md)
- [ ] Take on more complex issues

## Quick Reference

### Essential Commands
```bash
# Build
make debug                    # Debug build with sanitizers
make release                  # Optimized release build
GEN=ninja make               # Faster parallel build

# Test
make unit                    # Fast unit tests (~1 min)
make allunit                 # All unit tests (~1 hour)

# Format (ALWAYS before committing!)
make format-fix              # Format all code

# Clean
make clean                   # Remove build directory
```

### Key Directories
```
src/parser/      - SQL parsing
src/planner/     - Query planning and binding
src/optimizer/   - Query optimization
src/execution/   - Query execution engine
src/storage/     - Data storage layer
src/function/    - Built-in SQL functions
test/sql/        - SQL logic tests
extension/       - Extensions (Parquet, JSON, etc.)
```

### Getting Help
- **Documentation**: https://duckdb.org/docs/
- **GitHub Issues**: https://github.com/duckdb/duckdb/issues
- **Community Discord**: Ask questions and connect with developers
- **CLAUDE.md**: Comprehensive project guidelines in repository root

## Tips for Success

1. **Start small**: Don't try to understand everything at once
2. **Read code**: The best way to learn is by reading existing implementations
3. **Write tests**: Tests help you understand how components work
4. **Ask questions**: The DuckDB community is helpful and welcoming
5. **Document your learning**: Write notes in this directory as you learn
6. **Be patient**: It's a large codebase - learning takes time

## Contributing to This Directory

This is YOUR learning space. Feel free to:
- Add new documents as you learn
- Create cheat sheets and quick references
- Add code snippets and examples
- Document debugging sessions and solutions
- Share useful external resources

**Format**: Use markdown (.md) files for consistency and readability.

## Document Cross-References

To help you navigate between related topics, here are key connections:

### Function Development Chain
1. [function-development.md](function-development.md) - Overview of all function types
2. [example-scalar-function.md](example-scalar-function.md) - Hands-on scalar function
3. [example-aggregate-function.md](example-aggregate-function.md) - Hands-on aggregate function
4. [vectorized-execution.md](vectorized-execution.md) - Understanding the execution model
5. [testing-guide.md](testing-guide.md) - Testing your functions

### Architecture Understanding Chain
1. [overview.md](overview.md) - High-level architecture
2. [architecture-deep-dive.md](architecture-deep-dive.md) - Detailed internals
3. [parser-guide.md](parser-guide.md) → [optimizer-guide.md](optimizer-guide.md) → [storage-guide.md](storage-guide.md) - Subsystems
4. [vectorized-execution.md](vectorized-execution.md) - Execution engine
5. [parallel-execution.md](parallel-execution.md) - Parallelism

### Performance Optimization Chain
1. [performance-optimization.md](performance-optimization.md) - Profiling and optimization
2. [optimizer-guide.md](optimizer-guide.md) - Query optimization
3. [vectorized-execution.md](vectorized-execution.md) - Execution efficiency
4. [parallel-execution.md](parallel-execution.md) - Parallel performance
5. [storage-guide.md](storage-guide.md) - Storage performance

### Extension Development Chain
1. [extension-development.md](extension-development.md) - Complete extension guide
2. [function-development.md](function-development.md) - Adding functions to extensions
3. [testing-guide.md](testing-guide.md) - Testing extensions
4. [debugging-tips.md](debugging-tips.md) - Debugging extensions

### Usage to Development Bridge
1. [usage-overview.md](usage-overview.md) - Using DuckDB
2. [usage-sql-features.md](usage-sql-features.md) - SQL features
3. [function-development.md](function-development.md) - Implementing those features
4. [parser-guide.md](parser-guide.md) - Adding new SQL syntax

## Quick Navigation by Topic

**I want to...**
- **Use DuckDB**: Start with [usage-overview.md](usage-overview.md)
- **Learn SQL features**: Read [usage-sql-features.md](usage-sql-features.md)
- **Load data**: See [usage-data-import-export.md](usage-data-import-export.md)
- **Use Python/R/Java**: Check [usage-client-apis.md](usage-client-apis.md)
- **Contribute code**: Begin with [overview.md](overview.md)
- **Add a function**: Follow [example-scalar-function.md](example-scalar-function.md)
- **Build an extension**: Read [extension-development.md](extension-development.md)
- **Debug an issue**: Reference [debugging-tips.md](debugging-tips.md)
- **Optimize queries**: Study [performance-optimization.md](performance-optimization.md)
- **Understand architecture**: Read [architecture-deep-dive.md](architecture-deep-dive.md)
- **Write tests**: See [testing-guide.md](testing-guide.md)

## Next Steps

**For Users:** Start with [usage-overview.md](usage-overview.md) and begin using DuckDB!

**For Developers:** Open [overview.md](overview.md) and begin your DuckDB development journey!

---

*Last updated: 2025-10-14*
*21 comprehensive guides available covering usage, development, and internals*
*Maintained by: Your learning journey*
