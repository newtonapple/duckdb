# DuckDB GitHub Actions Overview

This document provides a comprehensive overview of all GitHub Actions workflows in the DuckDB repository for developers new to the codebase.

## Table of Contents
1. [Workflow Categories](#workflow-categories)
2. [Fork Safety and S3 Uploads](#fork-safety-and-s3-uploads)
3. [Core CI/CD Workflows](#core-cicd-workflows)
4. [Extension Workflows](#extension-workflows)
5. [Test Workflows](#test-workflows)
6. [PR and Issue Management Workflows](#pr-and-issue-management-workflows)
7. [Specialized Workflows](#specialized-workflows)
8. [Release Process Flow](#release-process-flow)
9. [Quick Reference Tables](#quick-reference-tables)

---

## Workflow Categories

DuckDB's GitHub Actions are organized into 6 main categories:

```
┌────────────────────────────────────────────────────────────┐
│                    DuckDB GitHub Actions                   │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Core CI    │  │  Extensions  │  │    Tests     │      │
│  │   (5 flows)  │  │  (4 flows)   │  │  (9 flows)   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  PR/Issue    │  │ Specialized  │  │   Release    │      │
│  │  Management  │  │  Language    │  │   Process    │      │
│  │  (11 flows)  │  │  Bindings    │  │  (3 flows)   │      │
│  └──────────────┘  │  (6 flows)   │  └──────────────┘      │
│                    └──────────────┘                        │
└────────────────────────────────────────────────────────────┘
```

---

## Fork Safety and S3 Uploads

**Critical for Contributors**: All workflows in this repository include multiple safety layers to prevent unauthorized S3 uploads from forks.

### S3 Upload Protection Mechanisms

The `scripts/upload-assets-to-staging.sh` script implements several checks:

```bash
# 1. Repository owner check - exits immediately if not duckdb org
if [ "$GITHUB_REPOSITORY_OWNER" != "duckdb" ]; then
  echo "Repository is $GITHUB_REPOSITORY_OWNER (not duckdb)"
  exit 0  # Fork exits here - no upload attempt
fi

# 2. Repository name check - dry-run mode if not duckdb/duckdb
if [ "$GITHUB_REPOSITORY" != "duckdb/duckdb" ]; then
  DRY_RUN_PARAM="--dryrun"  # Simulates upload without actually uploading
fi

# 3. Branch check - dry-run if not on main
if [ "$GITHUB_REF" != "refs/heads/main" ]; then
  DRY_RUN_PARAM="--dryrun"
fi

# 4. Credentials check - dry-run if AWS keys missing
if [ -z "$AWS_ACCESS_KEY_ID" ]; then
  DRY_RUN_PARAM="--dryrun"
fi
```

### What Happens in Forks

**When you fork duckdb/duckdb:**

1. **S3 Uploads**: Script exits immediately (no upload attempt)
2. **Extension Uploads**: Missing secrets cause graceful failure
3. **Code Signing**:
   - Apple code signing (OSX.yml): Only runs if `GITHUB_REPOSITORY == 'duckdb/duckdb'`
   - Azure code signing (Windows.yml): Only runs if `github.repository == 'duckdb/duckdb' && github.event_name != 'pull_request'`
4. **Workflow Execution**: All other build and test steps run normally

**Result**: Forks can test builds locally without any risk of unauthorized uploads or deployments.

### S3 Bucket Structure

DuckDB uses two separate S3 buckets:

**1. Staging Bucket** (`s3://duckdb-staging/`)
- **Path**: `s3://duckdb-staging/{commit-sha}/[{version-tag}/]duckdb/duckdb/github_release/`
- **Purpose**: Temporary storage between build and release
- **Contains**: CLI binaries, libraries, headers, amalgamation files
- **Uploaded by**: LinuxRelease.yml, OSX.yml, Windows.yml, BundleStaticLibs.yml

**2. Extension Repository** (`s3://duckdb-core-extensions/`)
- **Path**: `s3://duckdb-core-extensions/{version}/{arch}/{extension}.duckdb_extension.gz`
- **Purpose**: Runtime extension auto-loading/auto-install
- **Contains**: All DuckDB extensions (signed .duckdb_extension files)
- **Uploaded by**: Extensions.yml
- **Accessed by**: DuckDB CLI/clients at runtime via `INSTALL` and `LOAD` commands

---

## Core CI/CD Workflows

These workflows build and test DuckDB across all major platforms.

### 1. Main.yml - Primary CI Testing

**Triggers:**
- Pull requests (opened, reopened, ready_for_review, converted_to_draft)
- Push to non-main/feature branches
- Merge queue
- Manual dispatch
- Skips: Markdown files, tools/**, test/configs/**, certain .github paths

**Purpose:**
The primary continuous integration workflow that runs comprehensive tests on PRs and feature branches.

**Key Jobs:**
- `linux-debug`: Debug build with sanitizers and assertions
- `linux-release`: Full test suite with jemalloc and core extensions
- `linux-configs`: Tests multiple storage configurations (encryption, WAL, block sizes)
- `no-string-inline`: Tests alternative memory configurations
- `vector-sizes`: Tests non-standard vector sizes
- `valgrind`: Memory leak detection
- `threadsan`: Thread sanitizer tests
- `amalgamation-tests`: Validates amalgamation builds

**Artifacts:** None (testing only)

**Flow Diagram:**
```
PR/Push → Main.yml
           │
           ├─→ linux-debug ─────────┐
           ├─→ linux-release ───────┤
           ├─→ linux-configs ───────┤
           ├─→ no-string-inline ────┤
           ├─→ vector-sizes ────────┤──→ All Pass? → ✓ Success
           ├─→ valgrind ────────────┤
           ├─→ threadsan ───────────┤
           └─→ amalgamation-tests ──┘
```

---

### 2. LinuxRelease.yml - Linux Binary Builds

**Triggers:**
- Pull requests (opened, reopened, ready_for_review)
- Push to non-main/feature branches
- Merge queue
- Manual dispatch (with skip_tests option)
- Called by other workflows
- Skips: Markdown files, test/configs/**, tools/** (except tools/shell/**)

**Purpose:**
Builds release-quality Linux binaries for amd64 and arm64 architectures using manylinux containers.

**Key Jobs:**
- `linux-release-cli`: Matrix build for amd64/arm64 with full test suite
- `upload-libduckdb-src`: Creates source amalgamation
- `symbol-leakage`: Validates exported symbols

**Artifacts:**
- `duckdb_cli-linux-{arch}.zip` - CLI binary
- `duckdb_cli-linux-{arch}.gz` - CLI binary (gzipped)
- `libduckdb-linux-{arch}.zip` - Library + headers + amalgamation
- `libduckdb-src.zip` - Source amalgamation

**Flow Diagram:**
```
PR/Push/Call → LinuxRelease.yml
                    │
                    ├─→ linux-release-cli [amd64] ──→ Build & Test ───┐
                    │                                                 │
                    ├─→ linux-release-cli [arm64] ───→ Build & Test ──┤
                    │                                                 ├─→ Upload to S3
                    ├─→ upload-libduckdb-src ────────→ Amalgamation ──┤
                    │                                                 │
                    └─→ symbol-leakage ───────────────→ Validate ─────┘
```

---

### 3. OSX.yml - macOS Universal Binaries

**Triggers:**
- Push to non-main/feature branches (only if workflow file changes)
- Manual dispatch (with skip_tests option)
- Called by other workflows
- Skips: Markdown files, test/configs/**, tools/** (except tools/shell/**)

**Purpose:**
Builds macOS universal binaries (x86_64 + arm64) with Apple code signing and notarization.

**Key Jobs:**
- `xcode-debug`: Debug build on Apple Silicon with assertions
- `xcode-release`: Universal binary with code signing, notarization, and full tests

**Artifacts:**
- `duckdb_cli-osx-universal.zip` - Universal CLI binary
- `duckdb_cli-osx-universal.gz` - Universal CLI (gzipped)
- `libduckdb-osx-universal.zip` - Universal library + headers + amalgamation

**Special Features:**
- Apple code signing with Developer ID
- Notarization via `xcrun notarytool`
- Tests embedded C/C++ examples

**Flow Diagram:**
```
Push/Call → OSX.yml
             │
             ├─→ xcode-debug ──→ Debug Build & Test ────┐
             │                                          │
             └─→ xcode-release ──→ Universal Build ──→ Sign ──→ Notarize ──→ Test ──→ Upload to S3
```

---

### 4. Windows.yml - Windows Multi-Architecture Builds

**Triggers:**
- Pull requests (opened, reopened, ready_for_review)
- Push to non-main/feature branches
- Merge queue
- Manual dispatch (with skip_tests, run_all options)
- Called by other workflows
- Skips: Markdown files, test/configs/**, tools/** (except tools/shell/**)

**Purpose:**
Builds Windows binaries for amd64, win32, and arm64 with Azure code signing.

**Key Jobs:**
- `win-release-64`: amd64 build with Azure signing and full tests
- `win-release-32`: Win32 build (only on main or with run_all)
- `win-release-arm64`: ARM64 build (only on main or with run_all)
- `mingw`: MinGW toolchain validation
- `win-packaged-upload`: Aggregates all Windows artifacts

**Artifacts:**
- `duckdb_cli-windows-{arch}.zip` - CLI executables
- `libduckdb-windows-{arch}.zip` - DLLs + libs + headers + amalgamation
- `duckdb-binaries-windows` - Combined package with all platforms

**Flow Diagram:**
```
PR/Push/Call → Windows.yml
                    │
                    ├─→ win-release-64 ──→ Build & Sign & Test ─┐
                    │                                           │
                    ├─→ win-release-32 ──→ Build & Sign ────────┤
                    │                                           ├─→ win-packaged-upload ──→ Combined Artifact
                    ├─→ win-release-arm64 ──→ Build & Sign ─────┤
                    │                                           │
                    └─→ mingw ──→ Validation Build ─────────────┘
```

---

### 5. Android.yml - Android Native Libraries

**Triggers:**
- Push to main/feature branches (only if workflow file changes)
- Pull requests on main/feature branches (only if workflow file changes)
- Manual dispatch
- Repository dispatch

**Purpose:**
Builds Android native libraries for ARM architectures using Android NDK.

**Key Jobs:**
- `android`: Matrix build for armeabi-v7a (32-bit) and arm64-v8a (64-bit)

**Artifacts:**
- `libduckdb-android_{arch}.zip` - Native libraries + headers for each architecture

**Special Features:**
- Uses Android NDK r27
- Static extension builds
- Flexible page sizes for Android 15+ compatibility
- Only runs on main/feature branches

**Flow Diagram:**
```
Tag/Push → Android.yml
            │
            ├─→ android [armeabi-v7a] ──→ Build Static Libs ───┐
            │                                                  ├─→ Upload to S3
            └─→ android [arm64-v8a] ─────→ Build Static Libs ──┘
```

---

## Extension Workflows

These workflows manage DuckDB's extension ecosystem.

### 1. Extensions.yml - Master Extension Orchestrator

**Triggers:**
- Pull requests (opened, reopened, ready_for_review, converted_to_draft)
- Push to non-main/feature branches
- Merge queue
- Manual dispatch (with extra_exclude_archs, skip_tests, run_all options)
- Called by other workflows
- Skips: Markdown files, tools/** (except tools/shell/**), duckdb-wasm patches, most workflows

**Purpose:**
Orchestrates building all DuckDB extensions across all platforms, merges them into a versioned repository, and uploads to S3.

**Key Jobs:**
1. `load-extension-configs`: Loads 3 config files (in-tree, out-of-tree, rust-based)
2. `main-extensions`: Builds in-tree + out-of-tree extensions
3. `rust-based-extensions`: Builds rust-based extensions with rust toolchain
4. `create-extension-repository`: Merges all extensions into single repository
5. `upload-extensions`: Deploys to S3 with signing
6. `autoload-tests`: Tests extension autoloading
7. `check-load-install-extensions`: Validates extension_entries.hpp

**Extension Categories:**
- **In-tree**: autocomplete, core_functions, icu, json, parquet, tpcds, tpch, demo_capi
- **Out-of-tree**: avro, aws, azure, ducklake, encodings, excel, fts, httpfs, iceberg, inet, mysql_scanner, postgres_scanner, spatial, sqlite_scanner, sqlsmith, vss
- **Rust-based**: delta

**Artifacts:**
- `main-extensions-{sha}*` - Main extensions per architecture (GitHub Actions artifacts)
- `rust-based-extensions-{sha}*` - Rust extensions per architecture (GitHub Actions artifacts)
- `extension-repository-{sha}` - Merged repository with all .duckdb_extension files (GitHub Actions artifacts)
- `extension_entries.hpp` - Updated extension entries header (GitHub Actions artifacts)
- Extensions uploaded to S3: `s3://duckdb-core-extensions/{version}/{arch}/{extension}.duckdb_extension.gz`

**S3 Upload Details:**

The `upload-extensions` job uploads to the extension repository bucket:
- **Bucket**: `s3://duckdb-core-extensions/`
- **Path**: `{version}/{arch}/{extension}.duckdb_extension.gz`
- **Signing**: Extensions are signed with `DUCKDB_EXTENSION_SIGNING_PK` before upload
- **Access**: Public read access for DuckDB clients at runtime
- **Usage**: When users run `INSTALL extension_name;`, DuckDB downloads from this bucket

**Fork Protection:**
- Requires `AWS_ENDPOINT_URL`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` secrets
- Forks without these secrets will skip upload (graceful failure)
- Environment variable `DUCKDB_DEPLOY_SCRIPT_MODE: for_real` required for actual uploads

**Flow Diagram:**
```
PR/Push → Extensions.yml
           │
           └─→ load-extension-configs
                    │
                    ├─→ main-extensions ─────────────────────┐
                    │   (calls _extension_distribution)      │
                    │                                        │
                    └─→ rust-based-extensions ───────────────┤
                        (calls _extension_distribution)      │
                                                             │
                                                             │
                        create-extension-repository ←────────┘
                            │   (merges all extension artifacts)
                            │
                            ├─→ upload-extensions ──→ Sign & Upload to S3
                            │
                            ├─→ autoload-tests ──→ Test Extension Loading
                            │
                            └─→ check-load-install-extensions ──→ Validate Headers
```

---

### 2. _extension_distribution.yml - Reusable Extension Builder

**Triggers:**
- Called by other workflows only (workflow_call)
- Inputs: artifact_prefix, extension_config, exclude_archs, extra_toolchains, duckdb_ref, override_tag, skip_tests, save_cache

**Purpose:**
Reusable workflow that wraps the external `duckdb/extension-ci-tools` workflow to build extensions.

**Key Jobs:**
- `build`: Delegates to `duckdb/extension-ci-tools/.github/workflows/_extension_distribution.yml`

**Artifacts:**
- Named based on `artifact_prefix` input (e.g., `main-extensions-{sha}`, `rust-based-extensions-{sha}`)
- Contains built .duckdb_extension files

**Special Note:**
Acts as a configuration layer between Extensions.yml and the external CI tools repository.

---

### 3. _extension_client_tests.yml - Extension Client Testing

**Triggers:**
- Called by other workflows only (workflow_call)
- Inputs: duckdb_version (required)

**Purpose:**
Reusable workflow for testing extensions with DuckDB client libraries (Python).

**Key Jobs:**
- `python`: Builds DuckDB Python client and runs pytest tests

**Artifacts:** None (testing only)

---

### 4. NotifyExternalRepositories.yml - Cross-Repository Notifications

**Triggers:**
- Called by other workflows (workflow_call) - primarily InvokeCI.yml
- Manual dispatch
- Inputs: duckdb-sha, target-branch, triggering-event, should-publish, is-success, override-git-describe

**Purpose:**
Notifies external DuckDB repositories (ODBC, JDBC, Python, build-status) to trigger their workflows. These external repos then vendor the DuckDB source code and build language-specific bindings/packages.

**Key Jobs:**

1. **`notify-odbc-run`**: Triggers duckdb-odbc's Vendor.yml
   - **What it does**: duckdb-odbc vendors DuckDB source and builds ODBC drivers
   - **Condition**: Only if `is-success == true` and NOT on release tags
   - **Output**: ODBC drivers for various platforms

2. **`notify-jdbc-run`**: Triggers duckdb-java's Vendor.yml
   - **What it does**: duckdb-java vendors DuckDB source and builds JDBC drivers
   - **Condition**: Only if `is-success == true` and NOT on release tags
   - **Output**: JDBC jars published to Maven Central

3. **`notify-python-nightly`**: Triggers duckdb-python's release.yml
   - **What it does**: duckdb-python vendors DuckDB source and builds Python wheels
   - **Condition**: Always runs (even on release tags)
   - **Output**: Python wheels published to PyPI (for nightly builds or releases)
   - **Parameters**: Passes `duckdb-sha` and `pypi-index` (test vs production)

4. **`notify-nightly-build-status`**: Triggers duckdb-build-status's NightlyBuildsCheck.yml
   - **What it does**: Updates nightly build status dashboard
   - **Condition**: Always runs
   - **Output**: Status updates on build success/failure

**Important Notes:**
- **ODBC and JDBC**: Only trigger on nightly/main builds, NOT on releases
- **Python**: Triggers on both nightly builds AND releases
- **Build Status**: Always runs to track all builds

**Artifacts:** None (only triggers external workflows via GitHub API)

**External Repository Outputs:**

After notification, external repos produce:
- **duckdb-python** → PyPI packages (`pip install duckdb`)
- **duckdb-java** → Maven Central artifacts (JDBC)
- **duckdb-odbc** → ODBC drivers for Windows, Linux, macOS

**Flow Diagram:**
```
Caller (InvokeCI.yml) → NotifyExternalRepositories.yml
                         │
                         ├─→ notify-odbc-run ──────────→ duckdb-odbc (Vendor.yml)
                         │                               └─→ Builds ODBC drivers
                         │
                         ├─→ notify-jdbc-run ──────────→ duckdb-java (Vendor.yml)
                         │                               └─→ Builds JDBC jars → Maven
                         │
                         ├─→ notify-python-nightly ────→ duckdb-python (release.yml)
                         │                               └─→ Builds Python wheels → PyPI
                         │
                         └─→ notify-nightly-build-status → duckdb-build-status (NightlyBuildsCheck.yml)
                                                          └─→ Updates build dashboard
```

---

## Test Workflows

These workflows provide comprehensive testing coverage.

### 1. NightlyTests.yml - Comprehensive Nightly Testing

**Triggers:**
- Manual dispatch
- Repository dispatch
- Push (only when workflow file or duckdb-wasm patches change)
- Pull requests (only when workflow file or duckdb-wasm patches change)

**Purpose:**
Runs extensive, time-consuming tests with various build configurations that are too slow for regular CI.

**Key Jobs:**
- `linux-memory-leaks`: Python test suite for memory leaks
- `release-assert`: Linux release with assertions and slow verifiers
- `release-assert-osx`: macOS with assertions
- `release-assert-osx-storage`: macOS with forced storage
- `smaller-binary`: Tests SMALLER_BINARY flag
- `release-assert-clang`: Release assertions with Clang
- `sqllogic`: SQL logic tests
- `storage-initialization`: Zero initialization verification
- `extension-updating`: Extension update mechanisms with Minio S3
- `force-blocking-sink-source`: Async behavior testing
- `regression-test-memory-safety`: Safe vs unsafe builds comparison
- `vector-and-block-sizes`: Non-standard sizes (512-element vectors, 16kB blocks)
- `linux-debug-configs`: Various debug configurations
- `linux-wasm-experimental`: WebAssembly builds (currently disabled)
- `hash-zero`: HASH_ZERO flag testing
- `codecov`: Code coverage report generation

**Artifacts:**
- `coverage.zip` - Coverage HTML report

**Duration:** Several hours

---

### 2. ExtendedTests.yml - Compiler Performance Benchmarks

**Triggers:**
- Manual dispatch
- Push (only when workflow file changes)
- Pull requests (only when workflow file changes)

**Purpose:**
Performance regression testing comparing different compiler configurations.

**Key Jobs:**
- `regression-lto-benchmark-runner`: LTO=full vs non-LTO (macOS)
- `regression-clang16-vs-clang14-benchmark-runner`: Clang 16 vs Clang 14 (macOS)
- `regression-clang-benchmark-runner`: Clang vs GCC (Linux)
- `regression-flto-gcc-benchmark-runner`: GCC full LTO vs standard GCC

**Benchmark Suites:** Micro, TPCH, TPCH-PARQUET, TPCDS, H2OAI, IMDB

**Artifacts:** None

**Duration:** Several hours

---

### 3. ExtraTests.yml - Full Release Regression

**Triggers:**
- Manual dispatch only

**Purpose:**
Runs all regression tests against the last tagged release.

**Key Jobs:**
- `regression-test-all`: Compares current branch against last git tag across all benchmarks

**Artifacts:** None

**Duration:** Several hours

---

### 4. CrossVersion.yml - Database Compatibility Testing

**Triggers:**
- Called by other workflows (workflow_call)
- Manual dispatch
- Repository dispatch
- Push (only when workflow file changes)

**Purpose:**
Tests database file compatibility across multiple DuckDB versions (v1.0.0, v1.1.3, v1.2.2, v1.3-ossivalis, main).

**Key Jobs:**
- `osx-step-1` / `linux-step-1`: Builds each version, creates database files
- `osx-step-2` / `linux-step-2`: Attempts to open all database files with all versions

**Artifacts:**
- `files-osx-{version}` - Database files per version (macOS)
- `files-linux-{version}` - Database files per version (Linux)

**Flow Diagram:**
```
CrossVersion.yml
     │
     ├─→ osx-step-1 [v1.0.0] ──→ Create DB ──┐
     ├─→ osx-step-1 [v1.1.3] ──→ Create DB ──┤
     ├─→ osx-step-1 [v1.2.2] ──→ Create DB ──┤
     ├─→ osx-step-1 [v1.3] ────→ Create DB ──┤
     ├─→ osx-step-1 [main] ────→ Create DB ──┤
     │                                       │
     └─→ osx-step-2 ←────────────────────────┘
         │
         └─→ Test all versions can open all DBs
```

---

### 5. DockerTests.yml - Docker Image Validation

**Triggers:**
- Called by other workflows (workflow_call)
- Manual dispatch
- Repository dispatch
- Push (only when workflow file or test_docker_images.sh changes)
- Pull requests (only when workflow file or test_docker_images.sh changes)

**Purpose:**
Tests Docker image builds.

**Key Jobs:**
- `linux-x64-docker`: Runs `./scripts/test_docker_images.sh`

**Artifacts:** None

**Duration:** ~15 minutes

---

### 6. Regression.yml - Core Performance Regression

**Triggers:**
- Pull requests (opened, reopened, ready_for_review, converted_to_draft)
- Push to non-main/feature branches
- Merge queue
- Called by other workflows (workflow_call)
- Manual dispatch
- Repository dispatch
- Skips: Markdown, test/configs/**, tools/**, most workflows, wasm patches

**Purpose:**
Core regression testing that compares performance, storage size, binary size, and query plan costs.

**Key Jobs:**
- `regression-test-benchmark-runner`: Performance comparison (includes Fivetran benchmarks for main branch)
- `regression-test-storage`: Storage size and format compatibility
- `regression-test-binary-size`: Extension binary size
- `regression-test-plan-cost`: Join order plan cost (IMDB, TPCH)

**Artifacts:** None

**Comparison Strategy:**
- PRs: Compare against base branch
- Main: Compare against last successful Regression run

**Duration:** ~1 hour

---

### 7. cifuzz.yml - Continuous Fuzzing

**Triggers:**
- Manual dispatch
- Repository dispatch
- Push to non-main/feature branches
- Skips: Markdown, tools/**, wasm patches, most workflows

**Purpose:**
Continuous fuzzing using OSS-Fuzz infrastructure (only on duckdb/duckdb repo).

**Key Jobs:**
- `Fuzzing`: Matrix with 3 sanitizers (address, undefined, memory), runs for 1 hour each

**Artifacts:**
- `artifacts-{sanitizer}` - Crash artifacts if issues found

**Duration:** 1+ hour per sanitizer

---

### 8. CodeQuality.yml - Code Quality Checks

**Triggers:**
- Pull requests (opened, reopened, ready_for_review, converted_to_draft)
- Push to non-main/feature branches
- Merge queue
- Manual dispatch (with explicit_checks option)
- Repository dispatch
- Skips: Markdown, test/configs/**, wasm patches, most workflows, out_of_tree_extensions.cmake

**Purpose:**
Code quality validation including formatting, C enum integrity, and clang-tidy analysis.

**Key Jobs:**
- `format-check`: clang-format, black, cmake-format validation
- `enum-check`: C API enum integrity verification
- `tidy-check`: clang-tidy static analysis (full check on main/feature, diff check on other branches)

**Artifacts:** None

**Duration:** ~30 minutes

---

### 9. coverity.yml - Coverity Static Analysis

**Triggers:**
- Repository dispatch (for daily schedule)
- Manual dispatch

**Purpose:**
Static analysis using Coverity Scan (limited to 1 build/day for projects >1M LOC).

**Key Jobs:**
- `coverity`: Builds with cov-build wrapper, uploads to Coverity Scan service

**Artifacts:** None (uploaded to Coverity Scan)

**Duration:** ~1 hour

---

## PR and Issue Management Workflows

These workflows automate PR and issue management tasks.

### Auto-Draft PR System (3 workflows)

**Purpose:** Automatically converts PRs to draft when new commits are pushed, with cancellation support.

#### 1. DraftMe.yml - Draft on Synchronize

**Triggers:** PR synchronize (new commits pushed)

**What it does:** Saves PR node ID as artifact when non-draft PR receives new commits.

#### 2. DraftMeNot.yml - Cancel Auto Draft

**Triggers:** PR ready_for_review

**What it does:** Cancels pending auto-draft by joining same concurrency group.

#### 3. DraftPR.yml - Move PR to Draft

**Triggers:** Workflow run completion of DraftMe.yml

**What it does:** Uses GraphQL API to convert PR to draft status.

**Flow Diagram:**
```
New Commit Pushed to PR → DraftMe.yml ──→ Save PR ID ──→ DraftPR.yml ──→ Convert to Draft
                              │
                              │ (cancellation path)
                              │
PR Marked Ready ─────────→ DraftMeNot.yml ──→ Cancel DraftMe.yml
```

---

### 4. InvokeCI.yml - Master CI Orchestrator

**Triggers:**
- Repository dispatch (external API calls)
- Manual dispatch (GitHub UI)

**Purpose:**
Master workflow that invokes all major CI pipelines in parallel and notifies external repos. **This is NOT triggered automatically** by pushes, PRs, or tags - it must be manually invoked or triggered via API.

**Key Inputs:**
- `git_ref` (string, optional): Which commit/branch/tag to build
  - If empty: Builds from the workflow's triggering commit
  - If provided: Checks out and builds from the specified ref
- `override_git_describe` (string, optional): Version string for binaries and S3 path
- `skip_tests` (string, optional): Skip test execution
- `run_all` (string, optional): Build all architectures (including win32/arm64)
- `twine_upload` (string, optional): Upload Python packages to PyPI

**Commit Selection Logic:**

All downstream workflows receive both inputs and use them like this:
```yaml
- uses: actions/checkout@v4
  with:
    ref: ${{ inputs.git_ref }}  # Empty = triggering commit, else specified ref
```

**Important Notes:**
- Artifact names use `github.sha` (the triggering commit's SHA)
- Builds use the code from `git_ref` (which may be different)
- S3 paths use the **actual built commit's SHA** (from `git log -1`)

**Typical Usage Scenarios:**

1. **Pre-release builds** (most common):
   ```
   User triggers InvokeCI with:
     git_ref: "main"
     override_git_describe: "v1.2.3"

   Result:
     - Checks out current HEAD of main branch (e.g., commit def456)
     - Builds from def456
     - Version in binaries shows "v1.2.3"
     - S3 path: s3://duckdb-staging/def456/v1.2.3/...
   ```

2. **Testing specific commit**:
   ```
   User triggers InvokeCI with:
     git_ref: "abc123def"

   Result:
     - Checks out commit abc123def
     - Builds from abc123def
     - S3 path: s3://duckdb-staging/abc123def/...
   ```

3. **Nightly builds** (via external scheduler):
   ```
   External system triggers via repository_dispatch:
     git_ref: "main"

   Result:
     - Builds latest main branch
     - Notifies external repos
   ```

**What it calls:**
- Extensions.yml
- OSX.yml
- LinuxRelease.yml
- Windows.yml
- BundleStaticLibs.yml
- NotifyExternalRepositories.yml (always runs, even if builds fail)

**Flow Diagram:**
```
InvokeCI.yml (manual/API trigger with git_ref + override_git_describe)
     │
     ├─→ Extensions.yml ──────────┐
     ├─→ OSX.yml ─────────────────┤
     ├─→ LinuxRelease.yml ────────┤
     ├─→ Windows.yml ─────────────┼─→ Collect Results
     └─→ BundleStaticLibs.yml ────┘
              │
              └─→ NotifyExternalRepositories.yml (always runs)
```

**Artifact Output:**
- Uploads to S3 staging: `s3://duckdb-staging/{built-commit-sha}/[{version-tag}/]duckdb/duckdb/github_release/`
- Uploads to extension repo: `s3://duckdb-core-extensions/{version}/{arch}/*.duckdb_extension.gz`

---

### 5. IssuesCloseStale.yml - Stale Issue Management

**Triggers:**
- Repository dispatch
- Manual dispatch

**Purpose:**
Automatically manages stale issues and PRs using `actions/stale@v9`.

**Configuration:**
- Marks stale after: 180 days of inactivity
- Closes after: 30 additional days
- Exempt label: `no stale`
- Processes: Up to 500 issues/PRs per run

---

### 6. CheckIssueForCodeFormatting.yml - Issue Formatting Validation

**Triggers:**
- Issue opened

**Purpose:**
Checks new issues for unformatted code and posts helpful comment.

**Detection criteria:**
- >2 SQL keywords
- No backticked code blocks
- No indented code
- No inline code snippets

**Action:** Posts comment with formatting guidance

---

### Issue/PR Mirroring System (5 workflows)

**Purpose:** Creates internal mirrors of public issues/PRs/discussions for tracking.

#### 7. NeedsDocumentation.yml

**Triggers:** Label `Needs Documentation` on issues/PRs/discussions

**What it does:** Creates mirror issue in `duckdb/duckdb-web` repo for documentation tracking.

#### 8. PRNeedsMaintainerApproval.yml

**Triggers:** Label `needs maintainer approval` on PRs

**What it does:** Creates mirror issue in `duckdblabs/duckdb-internal` with "external action required" label.

#### 9. InternalIssuesCreateMirror.yml

**Triggers:** Multiple labels on issues

**What it does:**
- `PR submitted` / `fixed on nightly`: Removes "needs triage"
- `needs reproducible example`: Posts guidance comment
- `reproduced`: Creates/updates internal mirror with "reproduced" label
- `under review`: Creates/updates internal mirror with "under review" label

#### 10. InternalIssuesUpdateMirror.yml

**Triggers:** Issue/discussion closed or reopened

**What it does:**
- Synchronizes status between public and internal mirrors
- Adds "public closed" or "public reopened" labels
- Posts comments to internal mirrors

#### 11. MirrorDiscussions.yml

**Triggers:** Label `under review` on discussions

**What it does:** Creates mirror issue in `duckdblabs/duckdb-internal` with "discussion" label.

**Mirror System Diagram:**
```
Public Repository (duckdb/duckdb)          Internal/Doc Repositories
─────────────────────────────────          ─────────────────────────

Issue/PR/Discussion
      │
      ├─ [Needs Documentation] ──────────→ duckdb-web
      │                                     (Documentation tracking)
      │
      ├─ [needs maintainer approval] ────→ duckdb-internal
      │                                     (Approval tracking)
      │
      ├─ [reproduced] ───────────────────→ duckdb-internal
      │                                     (Issue tracking)
      │
      ├─ [under review] ─────────────────→ duckdb-internal
      │                                     (Review tracking)
      │
      └─ [closed/reopened] ──────────────→ Update mirrors
                                           (Status sync)
```

---

## Specialized Workflows

These workflows handle language-specific bindings and release processes.

### 1. Julia.yml - Julia Bindings Testing

**Triggers:**
- Pull requests (opened, reopened, ready_for_review, converted_to_draft)
- Push to non-main/feature branches
- Merge queue
- Manual dispatch
- Repository dispatch
- Skips: Markdown, examples, test, tools/** (except tools/juliapkg), wasm patches, most workflows

**Purpose:**
Tests Julia bindings across multiple Julia versions.

**Key Jobs:**
- `format_check`: Julia code formatting with JuliaFormatter (Julia 1.7)
- `main_julia`: Tests Julia 1.10 and latest, builds with TPCH and ICU extensions

**Artifacts:** None

---

### 2. Swift.yml - Swift Multi-Platform Testing

**Triggers:**
- Pull requests (opened, reopened, ready_for_review, converted_to_draft)
- Push to non-main/feature branches
- Merge queue
- Manual dispatch
- Repository dispatch
- Skips: Markdown, examples, test, tools/** (except tools/swift), wasm patches, most workflows

**Purpose:**
Tests Swift bindings across Apple platforms (macOS, iOS, tvOS).

**Key Jobs:**
- `test-apple-platforms`: Matrix testing across macOS (always), iOS Simulator (main only), tvOS Simulator (main only)

**Artifacts:** None

**Platform Matrix:**
```
Swift.yml
    │
    ├─→ macOS (all branches)
    │
    ├─→ iOS Simulator (main only)
    │
    └─→ tvOS Simulator (main only)
```

---

### 3. SwiftRelease.yml - Swift Package Release Automation

**Triggers:**
- Push on tags (any tag)
- Manual dispatch
- Repository dispatch

**Purpose:**
Automates Swift package release by syncing code to `duckdb/duckdb-swift` and creating tagged releases.

**Key Jobs:**
- `update`: Generates Swift package, commits to duckdb-swift, creates and pushes tag

**Artifacts:** None (commits directly to duckdb-swift)

---

### 4. BundleStaticLibs.yml - Static Library Bundling

**Triggers:**
- Called by other workflows (workflow_call)
- Manual dispatch
- Push (only when workflow file changes)
- Pull requests (only when workflow file changes)

**Purpose:**
Builds and bundles static libraries for multiple platforms with bundled extensions.

**Key Jobs:**
- `bundle-osx-static-libs`: Matrix for macOS Intel (amd64) and ARM (arm64)
- `bundle-mingw-static-lib`: Windows MinGW with Rtools 42
- `bundle-linux-static-libs`: Matrix for Linux amd64 and arm64 with manylinux containers

**Artifacts:**
- `duckdb-static-libs-osx-amd64`
- `duckdb-static-libs-osx-arm64`
- `duckdb-static-libs-windows-mingw`
- `duckdb-static-libs-linux-amd64`
- `duckdb-static-libs-linux-arm64`

**Platform Matrix:**
```
BundleStaticLibs.yml
         │
         ├─→ macOS [amd64] ───→ Bundle Static Libs ──┐
         ├─→ macOS [arm64] ───→ Bundle Static Libs ──┤
         │                                           ├─→ Upload to S3
         ├─→ Windows [MinGW] ─→ Bundle Static Libs ──┤
         │                                           │
         ├─→ Linux [amd64] ───→ Bundle Static Libs ──┤
         └─→ Linux [arm64] ───→ Bundle Static Libs ──┘
```

---

### 5. OnTag.yml - Release Trigger

**Triggers:**
- Push on version tags matching `v[0-9]+.[0-9]+.[0-9]+` (e.g., v1.0.0)
- Manual dispatch

**Purpose:**
Entry point for release process. **IMPORTANT**: This workflow does NOT build anything - it only downloads pre-existing artifacts from S3 staging and publishes them to GitHub Releases.

**Prerequisites:**
- Artifacts must already exist in S3 staging bucket at:
  `s3://duckdb-staging/{commit-sha}/{version-tag}/duckdb/duckdb/github_release/`
- Typically created by running InvokeCI.yml BEFORE creating the tag

**Key Jobs:**
- `staged_upload`: Calls StagedUpload.yml with target version from tag name

**Typical Flow:**
```bash
# 1. Maintainer runs InvokeCI to build artifacts (this step creates the artifacts)
# Trigger manually with git_ref="main" and override_git_describe="v1.2.3"

# 2. Wait for builds to complete and upload to S3 staging

# 3. Create and push the version tag (this triggers OnTag.yml)
git tag v1.2.3
git push origin v1.2.3

# 4. OnTag.yml triggers and publishes pre-built artifacts to GitHub Release
```

---

### 6. StagedUpload.yml - Artifact Publishing

**Triggers:**
- Called by other workflows (workflow_call) - primarily OnTag.yml
- Manual dispatch

**Purpose:**
Downloads pre-built release artifacts from S3 staging bucket and publishes them to GitHub releases. **Does NOT build anything** - only downloads and publishes.

**Key Jobs:**
- `staged-upload`:
  1. Downloads from S3 staging using `aws s3 cp --recursive`
  2. Publishes to GitHub releases using `scripts/asset-upload-gha.py`

**S3 Download Path:**
```
s3://duckdb-staging/
    └── {commit_hash}/              # SHA of the commit that was built
        └── {target_git_describe}/   # Version tag (e.g., v1.2.3)
            └── duckdb/duckdb/
                └── github_release/
                    ├── duckdb_cli-linux-amd64.zip
                    ├── duckdb_cli-osx-universal.zip
                    ├── duckdb_cli-windows-amd64.zip
                    ├── libduckdb-linux-amd64.zip
                    ├── libduckdb-osx-universal.zip
                    ├── libduckdb-windows-amd64.zip
                    └── ... (all other platform artifacts)
```

**GitHub Release Upload:**
- Uses `asset-upload-gha.py` script
- Only runs if `GITHUB_REPOSITORY == 'duckdb/duckdb'` (fork protection)
- Only runs on tag events (not on branches)
- Uploads all downloaded artifacts as release assets

**Important Notes:**
- If artifacts don't exist in S3, the workflow will fail
- The commit SHA and version tag must match what was used during build
- This is a "publish-only" step - no compilation happens here

---

## Release Process Flow

**CRITICAL UNDERSTANDING**: DuckDB uses a **two-stage release process**:
1. **BUILD stage** (before tagging) - Creates and stages artifacts
2. **PUBLISH stage** (after tagging) - Downloads and publishes artifacts

This separation allows artifact validation before public release.

---

### Complete Release Process

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TWO-STAGE RELEASE PROCESS                        │
└─────────────────────────────────────────────────────────────────────┘

╔═══════════════════════════════════════════════════════════════════╗
║ STAGE 1: BUILD (BEFORE TAGGING) - Typically 1-2 hours            ║
╚═══════════════════════════════════════════════════════════════════╝

Step 1: Maintainer manually triggers InvokeCI.yml via GitHub UI
        Inputs: git_ref="main", override_git_describe="v1.2.3"
   │
   └─→ InvokeCI.yml
        │
        ├─→ LinuxRelease.yml
        │   ├─→ Build: linux-amd64, linux-arm64
        │   ├─→ Create: CLI binaries + libraries + amalgamation
        │   └─→ Upload to: s3://duckdb-staging/def456/v1.2.3/.../github_release/
        │
        ├─→ OSX.yml
        │   ├─→ Build: macOS universal (Intel + ARM)
        │   ├─→ Code sign with Apple Developer ID
        │   ├─→ Notarize with Apple
        │   └─→ Upload to: s3://duckdb-staging/def456/v1.2.3/.../github_release/
        │
        ├─→ Windows.yml
        │   ├─→ Build: windows-amd64, windows-arm64
        │   ├─→ Code sign with Azure Trusted Signing
        │   └─→ Upload to: s3://duckdb-staging/def456/v1.2.3/.../github_release/
        │
        ├─→ Extensions.yml
        │   ├─→ Build: ALL extensions for ALL platforms
        │   ├─→ Sign extensions with DUCKDB_EXTENSION_SIGNING_PK
        │   └─→ Upload to: s3://duckdb-core-extensions/v1.2.3/{arch}/...
        │
        ├─→ BundleStaticLibs.yml
        │   ├─→ Build: Static libraries with bundled extensions
        │   └─→ Upload to: s3://duckdb-staging/def456/v1.2.3/.../github_release/
        │
        └─→ NotifyExternalRepositories.yml
            ├─→ Trigger: duckdb-python (builds Python wheels)
            ├─→ Trigger: duckdb-odbc (vendors DuckDB source)
            └─→ Trigger: duckdb-java (builds JDBC drivers)

Result: All artifacts staged at s3://duckdb-staging/def456/v1.2.3/...

╔═══════════════════════════════════════════════════════════════════╗
║ STAGE 2: PUBLISH (AFTER TAGGING) - ~5-10 minutes                 ║
╚═══════════════════════════════════════════════════════════════════╝

Step 2: Maintainer creates and pushes version tag
        $ git tag v1.2.3 def456
        $ git push origin v1.2.3
   │
   ├─→ OnTag.yml (triggered by tag matching v[0-9]+.[0-9]+.[0-9]+)
   │   │
   │   └─→ StagedUpload.yml
   │       ├─→ Download from: s3://duckdb-staging/def456/v1.2.3/.../github_release/*
   │       └─→ Publish to: GitHub Release (github.com/duckdb/duckdb/releases/tag/v1.2.3)
   │           ├─→ duckdb_cli-linux-amd64.zip
   │           ├─→ duckdb_cli-osx-universal.zip
   │           ├─→ duckdb_cli-windows-amd64.zip
   │           ├─→ libduckdb-linux-amd64.zip
   │           ├─→ libduckdb-osx-universal.zip
   │           ├─→ libduckdb-windows-amd64.zip
   │           └─→ ... (all platform artifacts)
   │
   └─→ SwiftRelease.yml (triggered by any tag)
       └─→ Sync code to duckdb/duckdb-swift with tag v1.2.3

Result: Public GitHub Release with all artifacts available for download
```

---

### What Gets Published Where

**GitHub Releases** (`github.com/duckdb/duckdb/releases`)
- **Contains**: CLI binaries, libraries, headers, amalgamation files
- **Consumers**:
  - Manual downloads by users
  - Homebrew formulas (pulls from GitHub Releases)
  - Docker images (pulls from GitHub Releases)
  - CI systems requiring specific versions

**Extension Repository** (`s3://duckdb-core-extensions/`)
- **Contains**: Signed .duckdb_extension.gz files for all platforms
- **Accessed by**: DuckDB CLI/clients at runtime
- **Usage**: `INSTALL extension_name;` and `LOAD extension_name;` commands

**PyPI** (`pypi.org/project/duckdb/`)
- **Built by**: duckdb-python repository (triggered by NotifyExternalRepositories)
- **Contains**: Python wheels for all platforms
- **Usage**: `pip install duckdb`

**Maven Central** (JDBC)
- **Built by**: duckdb-java repository (triggered by NotifyExternalRepositories)
- **Contains**: JDBC drivers
- **Usage**: Maven/Gradle dependencies

**Swift Package Manager**
- **Built by**: SwiftRelease.yml → duckdb/duckdb-swift
- **Contains**: Swift package with DuckDB source
- **Usage**: Swift Package Manager dependencies

**Not Automated in Main Repo:**
- Homebrew (separate tap repository)
- apt/deb packages (likely separate infrastructure)
- Conda packages (external conda-forge process)
- Julia packages (external process)

---

### Validation Between Stages

**Why Two Stages?**

The two-stage process allows maintainers to:
1. **Verify builds succeed** across all platforms before tagging
2. **Test artifacts** from staging bucket before public release
3. **Coordinate timing** with external repository releases (Python, JDBC, etc.)
4. **Cancel/retry** if issues found, without polluting git tags

**Example Timeline for v1.2.3 Release:**

```
Day 1, 10:00 AM - Trigger InvokeCI with git_ref="main", override_git_describe="v1.2.3"
Day 1, 11:30 AM - All builds complete, artifacts in S3 staging
Day 1, 12:00 PM - QA team tests artifacts from staging
Day 1,  2:00 PM - External repos (Python, JDBC) start their builds
Day 1,  4:00 PM - Maintainer validates everything looks good
Day 1,  4:30 PM - Create and push tag v1.2.3
Day 1,  4:35 PM - OnTag publishes to GitHub Release
Day 1,  5:00 PM - Users can download v1.2.3 from GitHub
Day 2, 10:00 AM - Python wheels available on PyPI
Day 2, 12:00 PM - JDBC drivers available on Maven
```

### Typical Development Flow

```
Developer Push/PR
      │
      ├─→ Main.yml (CI tests) ───────────────────┐
      │                                          │
      ├─→ CodeQuality.yml (format/tidy checks) ──┤
      │                                          │
      ├─→ Regression.yml (performance tests) ────┤
      │                                          ├─→ PR Approval
      ├─→ LinuxRelease.yml (build artifacts) ────┤
      │                                          │
      ├─→ Windows.yml (build artifacts) ─────────┤
      │                                          │
      └─→ Extensions.yml (build extensions) ─────┘
            │
            └─→ (After merge to main)
                  │
                  ├─→ NightlyTests.yml (scheduled)
                  │
                  ├─→ CrossVersion.yml (compatibility)
                  │
                  └─→ NotifyExternalRepositories.yml
                        │
                        ├─→ duckdb-odbc
                        ├─→ duckdb-java
                        └─→ duckdb-python
```

---

## Quick Reference Tables

### Workflows by Trigger Type

#### Pull Request Triggers (Most Common)
| Workflow | Draft Check | Platform | Duration |
|----------|------------|----------|----------|
| Main.yml | Yes | Linux | ~1 hour |
| LinuxRelease.yml | Yes | Linux | ~1 hour |
| Windows.yml | Yes | Windows | ~1-2 hours |
| Extensions.yml | Yes | Multi-platform | ~1-2 hours |
| Regression.yml | Yes | Linux | ~1 hour |
| CodeQuality.yml | Yes | Linux | ~30 min |
| Julia.yml | Yes | Linux/macOS | ~30 min |
| Swift.yml | Yes | macOS | ~30 min |

#### Schedule/Manual Triggers (Nightly/On-Demand)
| Workflow | Typical Frequency | Purpose |
|----------|------------------|---------|
| NightlyTests.yml | Nightly | Comprehensive testing |
| ExtendedTests.yml | Manual | Compiler benchmarks |
| ExtraTests.yml | Manual | Release regression |
| IssuesCloseStale.yml | Daily | Issue management |
| InvokeCI.yml | Manual | Full CI orchestration |
| coverity.yml | Daily | Static analysis |

#### Tag Triggers (Release)
| Workflow | Tag Pattern | Purpose |
|----------|------------|---------|
| OnTag.yml | v[0-9]+.[0-9]+.[0-9]+ | Release orchestration |
| SwiftRelease.yml | Any tag | Swift package release |
| Android.yml | Any tag | Android builds |

#### Label Triggers (Automation)
| Workflow | Label | Action |
|----------|-------|--------|
| NeedsDocumentation.yml | Needs Documentation | Create doc issue |
| PRNeedsMaintainerApproval.yml | needs maintainer approval | Create internal issue |
| InternalIssuesCreateMirror.yml | reproduced, under review | Create/update mirror |
| MirrorDiscussions.yml | under review | Mirror discussion |

---

### Artifacts by Workflow

#### Binary Artifacts
| Workflow | Platform | Artifact Names |
|----------|----------|----------------|
| LinuxRelease.yml | Linux amd64/arm64 | duckdb_cli-linux-{arch}.zip, libduckdb-linux-{arch}.zip |
| OSX.yml | macOS Universal | duckdb_cli-osx-universal.zip, libduckdb-osx-universal.zip |
| Windows.yml | Windows amd64/win32/arm64 | duckdb_cli-windows-{arch}.zip, libduckdb-windows-{arch}.zip |
| Android.yml | Android arm | libduckdb-android_{arch}.zip |
| BundleStaticLibs.yml | All platforms | duckdb-static-libs-{platform}-{arch} |

#### Extension Artifacts
| Workflow | Artifact Type |
|----------|--------------|
| Extensions.yml | main-extensions-{sha}*, rust-based-extensions-{sha}*, extension-repository-{sha} |

#### Test/Analysis Artifacts
| Workflow | Artifact Type |
|----------|--------------|
| NightlyTests.yml | coverage.zip |
| CrossVersion.yml | files-{platform}-{version} |
| cifuzz.yml | artifacts-{sanitizer} (on failure) |

---

### Reusable Workflows (Called by Others)

| Workflow | Purpose | Called By |
|----------|---------|-----------|
| _extension_distribution.yml | Build extensions | Extensions.yml |
| _extension_client_tests.yml | Test extension clients | Extension repos |
| NotifyExternalRepositories.yml | Notify external repos | InvokeCI.yml, other CI workflows |
| LinuxRelease.yml | Build Linux binaries | InvokeCI.yml, OnTag.yml |
| OSX.yml | Build macOS binaries | InvokeCI.yml, OnTag.yml |
| Windows.yml | Build Windows binaries | InvokeCI.yml, OnTag.yml |
| BundleStaticLibs.yml | Bundle static libraries | InvokeCI.yml |
| StagedUpload.yml | Publish releases | OnTag.yml |
| CrossVersion.yml | Compatibility testing | Other workflows |
| DockerTests.yml | Docker validation | Other workflows |
| Regression.yml | Performance testing | Other workflows |
| Extensions.yml | Build extensions | InvokeCI.yml |

---

### Critical Path Workflows (Must Pass for PRs)

1. **Main.yml** - Core CI testing across configurations
2. **CodeQuality.yml** - Format and static analysis
3. **Regression.yml** - Performance and size regression
4. **LinuxRelease.yml** - Linux binary builds
5. **Windows.yml** - Windows binary builds (amd64)
6. **Extensions.yml** - Extension builds (subset on PRs)

---

### Architecture Coverage Matrix

| Workflow | Linux amd64 | Linux arm64 | macOS Intel | macOS ARM | Windows amd64 | Windows arm64 | Android ARM |
|----------|------------|-------------|-------------|-----------|---------------|---------------|-------------|
| Main.yml | ✓ | | | | | | |
| LinuxRelease.yml | ✓ | ✓ | | | | | |
| OSX.yml | | | ✓ | ✓ | | | |
| Windows.yml | | | | | ✓ | ✓ | |
| Android.yml | | | | | | | ✓ |
| BundleStaticLibs.yml | ✓ | ✓ | ✓ | ✓ | ✓ (MinGW) | | |

---

## Summary

DuckDB's GitHub Actions are organized into a comprehensive CI/CD system with 34 workflows:

**Core CI (5 workflows):** Build and test across all major platforms
- Main.yml, LinuxRelease.yml, OSX.yml, Windows.yml, Android.yml

**Extensions (4 workflows):** Manage DuckDB's extension ecosystem
- Extensions.yml, _extension_distribution.yml, _extension_client_tests.yml, NotifyExternalRepositories.yml

**Tests (9 workflows):** Multiple layers of validation
- NightlyTests.yml, ExtendedTests.yml, ExtraTests.yml, CrossVersion.yml, DockerTests.yml, Regression.yml, cifuzz.yml, CodeQuality.yml, coverity.yml

**PR/Issue Management (11 workflows):** Automate repository management
- DraftMe.yml, DraftMeNot.yml, DraftPR.yml, InvokeCI.yml, IssuesCloseStale.yml, CheckIssueForCodeFormatting.yml, NeedsDocumentation.yml, PRNeedsMaintainerApproval.yml, InternalIssuesCreateMirror.yml, InternalIssuesUpdateMirror.yml, MirrorDiscussions.yml

**Specialized (6 workflows):** Language bindings and release automation
- Julia.yml, Swift.yml, SwiftRelease.yml, BundleStaticLibs.yml, OnTag.yml, StagedUpload.yml

### Key Design Principles

1. **Parallel Execution:** Multiple workflows run simultaneously on the same trigger
2. **Platform Independence:** Each platform has its own workflow
3. **Staged Deployment:** Build → S3 Staging → GitHub Releases
4. **Flexibility:** Workflows support both automatic and manual triggers
5. **Extensibility:** Reusable workflows via `workflow_call`

### For Beginners

Most PRs trigger these critical workflows:
- **Main.yml** - Core testing
- **CodeQuality.yml** - Format checks
- **Regression.yml** - Performance tests
- **LinuxRelease.yml** - Linux builds
- **Windows.yml** - Windows builds
- **Extensions.yml** - Extension builds

These must all pass before your PR can be merged. Additional workflows like NightlyTests and CrossVersion provide deeper safety nets that run less frequently.

The release process is fully automated: when a version tag (e.g., v1.0.0) is pushed, OnTag.yml triggers StagedUpload.yml which downloads pre-built artifacts from S3 and publishes them to GitHub Releases.
