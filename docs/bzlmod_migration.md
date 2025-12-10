# Bzlmod Migration Progress

This document tracks the progress of migrating envoy_examples to bzlmod using the work-in-progress branches:
- https://github.com/mmorel-35/envoy/tree/bzlmod-migration
- https://github.com/mmorel-35/toolshed/tree/bzlmod

## Status: 🔴 BLOCKED - Critical Issues Identified

## Configuration Applied

### Git Overrides Added to `wasm-cc/MODULE.bazel`

The following `git_override` directives have been successfully added:

```starlark
# Git overrides for bzlmod migration
git_override(
    module_name = "envoy",
    commit = "4fc5c5cd8a2aec2a51fd21462bbd648d92d0889e",
    remote = "https://github.com/mmorel-35/envoy",
)

git_override(
    module_name = "envoy_api",
    commit = "4fc5c5cd8a2aec2a51fd21462bbd648d92d0889e",
    remote = "https://github.com/mmorel-35/envoy",
    strip_prefix = "api",
)

git_override(
    module_name = "envoy_build_config",
    commit = "4fc5c5cd8a2aec2a51fd21462bbd648d92d0889e",
    remote = "https://github.com/mmorel-35/envoy",
    strip_prefix = "mobile/envoy_build_config",
)

git_override(
    module_name = "envoy_mobile",
    commit = "4fc5c5cd8a2aec2a51fd21462bbd648d92d0889e",
    remote = "https://github.com/mmorel-35/envoy",
    strip_prefix = "mobile",
)

git_override(
    module_name = "envoy_toolshed",
    commit = "d718b38e7d0bd7e41394ff48db046b15b20784d5",
    remote = "https://github.com/mmorel-35/toolshed",
    strip_prefix = "bazel",
)

git_override(
    module_name = "xds",
    commit = "8bfbf64dc13ee1a570be4fbdcfccbdd8532463f0",
    remote = "https://github.com/cncf/xds",
)

git_override(
    module_name = "toolchains_llvm",
    commit = "fb29f3d53757790dad17b90df0794cea41f1e183",
    remote = "https://github.com/bazel-contrib/toolchains_llvm",
)
```

### bazel_dep Declarations Added

```starlark
bazel_dep(name = "envoy")
bazel_dep(name = "envoy_api")
bazel_dep(name = "envoy_build_config")
bazel_dep(name = "envoy_mobile")
bazel_dep(name = "envoy_toolshed")
bazel_dep(name = "xds", repo_name = "com_github_cncf_xds")
```

## Critical Blockers

### 🔴 Blocker #1: Circular Dependency (envoy ↔ envoy_examples)

**Status:** Critical - Prevents migration

**Description:**
A circular dependency exists between `envoy` and `envoy_examples`:

```
envoy_examples (wasm-cc) 
    → depends on → envoy
        → depends on → envoy_examples (via envoy_dependencies_extension)
```

**Evidence:**
In `envoy/MODULE.bazel` (bzlmod-migration branch):
```starlark
envoy_deps = use_extension("//bazel:extensions.bzl", "envoy_dependencies_extension")
use_repo(
    envoy_deps,
    ...
    "envoy_examples",
    ...
)
```

In `envoy/bazel/repository_locations.bzl`:
```python
envoy_examples = dict(
    project_name = "envoy_examples",
    project_desc = "Envoy proxy examples",
    project_url = "https://github.com/envoyproxy/examples",
    version = "0.1.4",
    sha256 = "9bb7cd507eb8a090820c8de99f29d9650ce758a84d381a4c63531b5786ed3143",
    strip_prefix = "examples-{version}",
    urls = ["https://github.com/envoyproxy/examples/archive/v{version}.tar.gz"],
    use_category = ["test_only"],  # ← Only used for testing
    ...
)
```

**Impact:** 
- Bazel module resolution will fail due to circular dependency
- Cannot proceed with bzlmod migration until resolved

**Potential Solutions:**

1. **Mark envoy_examples as dev_dependency in envoy (RECOMMENDED)**
   
   Since `envoy_examples` is marked as `use_category = ["test_only"]`, it should be a dev dependency:
   
   ```starlark
   # In envoy MODULE.bazel - if envoy_examples is needed as a module
   bazel_dep(name = "envoy_examples", dev_dependency = True)
   ```
   
   This prevents envoy_examples from being included when envoy is used as a dependency.

2. **Remove envoy_examples dependency from envoy**
   
   If envoy doesn't actually need envoy_examples for its core functionality (only for testing), remove it from the main dependency graph and load it only in CI/test environments.

3. **Split test dependencies into separate extension**
   
   Create a separate module extension for test-only dependencies that is only used when building envoy itself, not when envoy is used as a dependency.

4. **Use archive_override in envoy_examples instead of depending on envoy directly**
   
   Instead of depending on the published envoy module, use archive_override or git_override to get a specific version that doesn't include envoy_examples as a dependency. However, this doesn't solve the fundamental circular dependency issue.

**Recommended Action for envoy bzlmod-migration branch:**
```starlark
# If envoy_examples must be loaded, make it dev-only:
bazel_dep(name = "envoy_examples", dev_dependency = True)

# OR remove it entirely from the extension if it's only for testing:
# - Remove "envoy_examples" from use_repo(envoy_deps, ...)
# - Load it separately in test-only contexts
```

### 🔴 Blocker #2: LLVM Extension Can Only Be Used by Root Module

**Status:** Critical - Prevents module resolution

**Error:**
```
ERROR: Only the root module can use the 'llvm' extension
```

**Description:**
The `envoy` module (from https://github.com/mmorel-35/envoy/tree/bzlmod-migration) uses the LLVM toolchain extension in its MODULE.bazel:

```starlark
llvm = use_extension("@toolchains_llvm//toolchain/extensions:llvm.bzl", "llvm")
llvm.toolchain(
    name = "llvm_toolchain",
    llvm_version = "18.1.8",
    cxx_standard = {"": "c++20"},
)
use_repo(llvm, "llvm_toolchain", "llvm_toolchain_llvm")
```

When `envoy` is used as a dependency (not the root module), Bazel's bzlmod system does not allow non-root modules to use this extension.

**Impact:** 
- Module resolution fails completely
- Cannot test any builds with envoy as a dependency

**Potential Solutions:**

1. **Make LLVM configuration conditional (RECOMMENDED)**
   
   Use `module_ctx.is_root` to only configure LLVM when envoy is the root module:
   
   ```starlark
   # In envoy MODULE.bazel - wrap the extension usage
   # This requires a custom extension wrapper or checking if there's a way
   # to conditionally use extensions
   ```

2. **Remove LLVM extension from envoy MODULE.bazel**
   
   Instead, document that consuming modules must configure toolchains_llvm themselves:
   
   ```starlark
   # In envoy MODULE.bazel - Remove:
   # llvm = use_extension(...)
   
   # Add documentation comment:
   # NOTE: Consuming projects must configure toolchains_llvm with:
   # - llvm_version = "18.1.8"
   # - cxx_standard = {"": "c++20"}
   ```

3. **Use a different toolchain configuration mechanism**
   
   Investigate if toolchains_llvm has alternative configuration methods that work for non-root modules.

**Recommended Action for envoy bzlmod-migration branch:**
The envoy MODULE.bazel should not directly configure the llvm extension. Instead:
- Document the required LLVM configuration
- Allow consuming modules to configure it
- Potentially provide a helper extension that can be used by root modules

### 🟡 Blocker #3: Rust Cargo Lockfile Out of Date

**Status:** Build-time blocker (after resolving #1 and #2)

**Error:**
```
Error: Digests do not match: Current Digest("b864c94e442ea41673dcae0f7039f7afb9ef5c4287962b4464b406f670a8e6d7") != Expected Digest("7a9bbeeee5eb0baac08a5939158695e44af89cc79afe3b93b61944f78e3be539")

The current `lockfile` is out of date for 'dynamic_modules_rust_sdk_crate_index'. Please re-run bazel using `CARGO_BAZEL_REPIN=true`
```

**Solution:**
In the envoy repository:
```bash
CARGO_BAZEL_REPIN=true bazel build //...
git add -A
git commit -m "Update Rust Cargo lockfiles"
```

## Warnings (Non-blocking)

### Version Conflicts

Several dependency version mismatches were detected:

```
WARNING: For repository 'rules_cc', the root module requires module version rules_cc@0.1.1, but got rules_cc@0.2.14 in the resolved dependency graph.
WARNING: For repository 'io_bazel_rules_go', the root module requires module version rules_go@0.53.0, but got rules_go@0.59.0 in the resolved dependency graph.
WARNING: For repository 'rules_python', the root module requires module version rules_python@1.4.1, but got rules_python@1.6.3 in the resolved dependency graph.
WARNING: For repository 'rules_rust', the root module requires module version rules_rust@0.56.0, but got rules_rust@0.67.0 in the resolved dependency graph.
```

**Solution:**
Update `wasm-cc/MODULE.bazel` to use compatible versions or remove version constraints:

```starlark
# Update to match envoy's requirements:
bazel_dep(name = "rules_cc", version = "0.2.14")
bazel_dep(name = "rules_go", version = "0.59.0", repo_name = "io_bazel_rules_go")
bazel_dep(name = "rules_python", version = "1.6.3")
bazel_dep(name = "rules_rust", version = "0.67.0")
```

### Maven Module Version Conflicts

```
WARNING: The following maven modules appear in multiple sub-modules with potentially different versions:
	com.google.code.gson:gson (versions: 2.10.1, 2.8.9)
	com.google.errorprone:error_prone_annotations (versions: 2.23.0, 2.5.1)
	com.google.guava:guava (versions: 32.0.1-jre, 33.0.0-jre)
```

Can be addressed by adding explicit version pins in the root MODULE.bazel if needed.

## Dependencies Structure

The envoy bzlmod-migration uses the following module structure:

| Module | Location | Description |
|--------|----------|-------------|
| `envoy` | Root | Main envoy module |
| `envoy_api` | `api/` subdirectory | API definitions (protobuf) |
| `envoy_build_config` | `mobile/envoy_build_config/` | Build configuration for mobile |
| `envoy_mobile` | `mobile/` subdirectory | Mobile platform support |
| `envoy_toolshed` | Separate repo | Development and CI tooling |
| `xds` | External (cncf/xds) | xDS protocol definitions |

## Testing Progress

- ✅ Git overrides correctly configured
- ✅ bazel_dep declarations added
- ✅ Commit hashes verified for git_override
- 🔴 Module dependency graph resolution - **BLOCKED** (circular dependency)
- 🔴 LLVM extension issue - **BLOCKED** (non-root module restriction)
- ⏸️ Build testing - **BLOCKED** (waiting for above issues)

## Next Steps

### For envoy bzlmod-migration branch:

1. **[CRITICAL] Fix circular dependency**
   - Option A: Make envoy_examples a dev_dependency in envoy
   - Option B: Remove envoy_examples from envoy's dependency graph entirely
   - Option C: Restructure so envoy_examples doesn't depend on envoy core

2. **[CRITICAL] Fix LLVM extension usage**
   - Remove llvm extension configuration from envoy MODULE.bazel
   - Document required LLVM configuration for consuming modules
   - Consider providing a helper/wrapper extension

3. **[REQUIRED] Update Rust lockfiles**
   - Run `CARGO_BAZEL_REPIN=true bazel build //...`
   - Commit updated lockfiles

### For envoy_examples:

4. **Update dependency versions** (after blockers resolved)
   - Align rules_cc, rules_go, rules_python, rules_rust versions with envoy

5. **Test builds** (after blockers resolved)
   - Test `bazel build //wasm-cc:envoy_filter_http_wasm_example.wasm`
   - Test other example builds
   - Verify CI compatibility

6. **Update documentation**
   - Document bzlmod usage for contributors
   - Update build instructions

## Recommendations for envoy_toolshed

✅ The `envoy_toolshed` bzlmod branch (https://github.com/mmorel-35/toolshed/tree/bzlmod) appears to work correctly with git_override. No blockers identified for this repository.

## References

- Envoy bzlmod migration branch: https://github.com/mmorel-35/envoy/tree/bzlmod-migration
- Toolshed bzlmod branch: https://github.com/mmorel-35/toolshed/tree/bzlmod
- Bazel bzlmod documentation: https://bazel.build/external/module
- Module extensions: https://bazel.build/external/extension
