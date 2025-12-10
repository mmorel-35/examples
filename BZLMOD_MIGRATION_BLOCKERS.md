# Bzlmod Migration Blockers

This document tracks blockers and issues discovered while attempting to migrate envoy_examples to bzlmod using the branches:
- https://github.com/mmorel-35/envoy/tree/bzlmod-migration
- https://github.com/mmorel-35/toolshed/tree/bzlmod

## Configuration Applied

### Git Overrides Added to `wasm-cc/MODULE.bazel`

The following git_override directives have been added to support the bzlmod migration:

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

## Blockers

### 1. LLVM Extension Can Only Be Used by Root Module (CRITICAL)

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
```

However, when `envoy` is used as a dependency (not the root module), Bazel's bzlmod system does not allow non-root modules to use this extension. This causes the build to fail.

**Impact:** Complete blocker - cannot proceed with module resolution

**Potential Solutions:**
1. **Modify envoy MODULE.bazel:** Make the LLVM extension usage conditional - only use it when envoy is the root module. This could be done by:
   - Moving LLVM configuration to a separate extension that can be used by dependencies
   - Using module extension isolation features (if available)
   - Making LLVM configuration optional and allowing parent modules to configure it

2. **Use Bazel module extension tag classes:** Investigate if toolchains_llvm supports tag classes that allow non-root modules to contribute to the extension

3. **Root module override pattern:** Document that consuming modules must configure LLVM themselves with compatible settings

**Recommended Fix in envoy bzlmod-migration branch:**
The envoy MODULE.bazel should not directly use the llvm extension. Instead, it should:
- Declare the bazel_dep on toolchains_llvm
- Document the required LLVM version and configuration
- Allow the root module (consuming project) to configure LLVM

Example pattern:
```starlark
# In envoy MODULE.bazel - Remove this:
# llvm = use_extension("@toolchains_llvm//toolchain/extensions:llvm.bzl", "llvm")
# llvm.toolchain(...)

# Add documentation comment instead:
# Note: Consuming projects must configure toolchains_llvm@1.0.0 with:
# - llvm_version = "18.1.8"
# - cxx_standard = {"": "c++20"}
```

### 2. Rust Cargo Lockfile Out of Date

**Error:**
```
Error: Digests do not match: Current Digest("b864c94e442ea41673dcae0f7039f7afb9ef5c4287962b4464b406f670a8e6d7") != Expected Digest("7a9bbeeee5eb0baac08a5939158695e44af89cc79afe3b93b61944f78e3be539")

The current `lockfile` is out of date for 'dynamic_modules_rust_sdk_crate_index'. Please re-run bazel using `CARGO_BAZEL_REPIN=true`
```

**Description:**
The Rust dependencies in the envoy repository have an outdated Cargo.Bazel.lock file.

**Impact:** Build failure after resolving blocker #1

**Solution:**
Run the following in the envoy repository:
```bash
CARGO_BAZEL_REPIN=true bazel build //...
```
Then commit the updated lockfile.

### 3. Version Conflicts (Warnings)

**Warnings:**
```
WARNING: For repository 'rules_cc', the root module requires module version rules_cc@0.1.1, but got rules_cc@0.2.14 in the resolved dependency graph.
WARNING: For repository 'io_bazel_rules_go', the root module requires module version rules_go@0.53.0, but got rules_go@0.59.0 in the resolved dependency graph.
WARNING: For repository 'rules_python', the root module requires module version rules_python@1.4.1, but got rules_python@1.6.3 in the resolved dependency graph.
WARNING: For repository 'rules_rust', the root module requires module version rules_rust@0.56.0, but got rules_rust@0.67.0 in the resolved dependency graph.
```

**Description:**
The wasm-cc module requests older versions of some rules, but the dependency graph resolution chooses newer versions from envoy's dependencies.

**Impact:** Non-blocking warnings, but may cause compatibility issues

**Solution:**
Update the wasm-cc MODULE.bazel to use the same versions as envoy, or remove version specifications to accept whatever version is resolved:

```starlark
# Either update to match envoy:
bazel_dep(name = "rules_cc", version = "0.2.14")
bazel_dep(name = "rules_go", version = "0.59.0", repo_name = "io_bazel_rules_go")
bazel_dep(name = "rules_python", version = "1.6.3")
bazel_dep(name = "rules_rust", version = "0.67.0")

# Or remove version to accept resolved version:
bazel_dep(name = "rules_cc")
bazel_dep(name = "rules_go", repo_name = "io_bazel_rules_go")
bazel_dep(name = "rules_python")
bazel_dep(name = "rules_rust")
```

### 4. Maven Module Version Conflicts (Warning)

**Warning:**
```
WARNING: The following maven modules appear in multiple sub-modules with potentially different versions:
	com.google.code.gson:gson (versions: 2.10.1, 2.8.9)
	com.google.errorprone:error_prone_annotations (versions: 2.23.0, 2.5.1)
	com.google.guava:guava (versions: 32.0.1-jre, 33.0.0-jre)
```

**Description:**
Different modules in the dependency graph request different versions of Maven artifacts.

**Impact:** Non-blocking warning, but may cause runtime issues

**Solution:**
Add explicit version pins in the root MODULE.bazel or configure maven resolution preferences.

## Additional Notes

### Dependencies Structure

The envoy bzlmod-migration uses the following module structure:
- `envoy` - Main envoy module
- `envoy_api` - API definitions (located in `api/` subdirectory)
- `envoy_build_config` - Build configuration (located in `mobile/envoy_build_config/` subdirectory)  
- `envoy_mobile` - Mobile support (located in `mobile/` subdirectory)
- `envoy_toolshed` - Development tooling (separate repository)

All of these need to be available as git_override or bazel_dep for consuming projects.

### Testing Status

- ✅ Git overrides correctly configured
- ✅ Module dependency graph partially resolves
- ❌ Cannot complete module resolution due to LLVM extension blocker
- ⏸️ Build testing blocked until LLVM issue is resolved

## Recommendations for envoy_toolshed Repository

The `envoy_toolshed` bzlmod branch appears to work correctly with the git_override. No immediate blockers identified for this repository.

## Next Steps

1. **Priority 1:** Fix the LLVM extension issue in the envoy bzlmod-migration branch
2. **Priority 2:** Update Rust Cargo lockfiles in the envoy repository
3. **Priority 3:** Align version requirements between wasm-cc and envoy modules
4. **Priority 4:** Test actual build after resolving blockers
