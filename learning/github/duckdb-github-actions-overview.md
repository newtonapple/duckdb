# DuckDB GitHub Actions Overview

This document provides a comprehensive overview of all GitHub Actions workflows in the DuckDB repository for developers new to the codebase.

## Table of Contents
1. [Workflow Categories](#workflow-categories)
2. [Core CI/CD Workflows](#core-cicd-workflows)
3. [Extension Workflows](#extension-workflows)
4. [Test Workflows](#test-workflows)
5. [PR and Issue Management Workflows](#pr-and-issue-management-workflows)
6. [Specialized Workflows](#specialized-workflows)
7. [Release Process Flow](#release-process-flow)
8. [Quick Reference Tables](#quick-reference-tables)

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
- `main-extensions-{sha}*` - Main extensions per architecture
- `rust-based-extensions-{sha}*` - Rust extensions per architecture
- `extension-repository-{sha}` - Merged repository with all .duckdb_extension files
- `extension_entries.hpp` - Updated extension entries header

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
- Called by other workflows (workflow_call)
- Manual dispatch
- Inputs: duckdb-sha, target-branch, triggering-event, should-publish, is-success, override-git-describe

**Purpose:**
Notifies external DuckDB repositories (ODBC, JDBC, Python, build-status) to trigger their workflows.

**Key Jobs:**
- `notify-odbc-run`: Triggers duckdb-odbc's Vendor.yml
- `notify-jdbc-run`: Triggers duckdb-java's Vendor.yml
- `notify-nightly-build-status`: Triggers duckdb-build-status's NightlyBuildsCheck.yml
- `notify-python-nightly`: Triggers duckdb-python's release.yml

**Artifacts:** None (only triggers external workflows)

**Flow Diagram:**
```
Caller → NotifyExternalRepositories.yml
          │
          ├─→ notify-odbc-run ──────────→ duckdb-odbc (Vendor.yml)
          │
          ├─→ notify-jdbc-run ──────────→ duckdb-java (Vendor.yml)
          │
          ├─→ notify-nightly-build-status → duckdb-build-status (NightlyBuildsCheck.yml)
          │
          └─→ notify-python-nightly ─────→ duckdb-python (release.yml)
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
- Repository dispatch
- Manual dispatch

**Purpose:**
Master workflow that invokes all major CI pipelines in parallel and notifies external repos.

**What it calls:**
- Extensions.yml
- OSX.yml
- LinuxRelease.yml
- Windows.yml
- BundleStaticLibs.yml
- NotifyExternalRepositories.yml (always runs)

**Flow Diagram:**
```
InvokeCI.yml
     │
     ├─→ Extensions.yml ──────────┐
     ├─→ OSX.yml ─────────────────┤
     ├─→ LinuxRelease.yml ────────┤
     ├─→ Windows.yml ─────────────┼─→ Collect Results
     └─→ BundleStaticLibs.yml ────┘
              │
              └─→ NotifyExternalRepositories.yml
```

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
      ├─ [Needs Documentation] ───────────→ duckdb-web
      │                                     (Documentation tracking)
      │
      ├─ [needs maintainer approval] ─────→ duckdb-internal
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
         ├─→ macOS [amd64] ──→ Bundle Static Libs ──┐
         ├─→ macOS [arm64] ──→ Bundle Static Libs ──┤
         │                                           ├─→ Upload to S3
         ├─→ Windows [MinGW] ─→ Bundle Static Libs ─┤
         │                                           │
         ├─→ Linux [amd64] ───→ Bundle Static Libs ─┤
         └─→ Linux [arm64] ───→ Bundle Static Libs ─┘
```

---

### 5. OnTag.yml - Release Trigger

**Triggers:**
- Push on version tags matching `v[0-9]+.[0-9]+.[0-9]+` (e.g., v1.0.0)
- Manual dispatch

**Purpose:**
Entry point for release process, immediately calls StagedUpload workflow.

**Key Jobs:**
- `staged_upload`: Calls StagedUpload.yml with target version

---

### 6. StagedUpload.yml - Artifact Publishing

**Triggers:**
- Called by other workflows (workflow_call)
- Manual dispatch

**Purpose:**
Downloads pre-built release artifacts from S3 staging bucket and uploads to GitHub releases.

**Key Jobs:**
- `staged-upload`: Downloads from S3 and publishes to GitHub releases using asset-upload-gha.py

**S3 Structure:**
```
s3://duckdb-staging/
    └── {commit_hash}/
        └── {target_git_describe}/
            └── {repo}/
                └── github_release/
                    └── [artifacts]
```

---

## Release Process Flow

Here's how DuckDB releases work:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         RELEASE PROCESS                             │
└─────────────────────────────────────────────────────────────────────┘

1. Tag Created (v1.0.0)
   │
   └─→ OnTag.yml
         │
         └─→ StagedUpload.yml
               │
               ├─→ Download from S3 staging bucket
               │   (artifacts built by previous CI runs)
               │
               └─→ Upload to GitHub Release

Parallel Processes for Tag:
   │
   ├─→ SwiftRelease.yml ──→ Update duckdb-swift repo & create tag
   │
   └─→ InvokeCI.yml (if manually triggered)
         │
         ├─→ LinuxRelease.yml ──→ Build Linux binaries ───┐
         ├─→ OSX.yml ──────────→ Build macOS binaries ────┤
         ├─→ Windows.yml ───────→ Build Windows binaries ─┤
         ├─→ Extensions.yml ────→ Build extensions ───────┼─→ Upload to S3 staging
         └─→ BundleStaticLibs ──→ Build static libs ──────┘
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
