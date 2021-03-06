# rules_foreign_cc
[![Build status](https://badge.buildkite.com/c28afbf846e2077715c753dda1f4b820cdcc46cc6cde16503c.svg)](https://buildkite.com/bazel/rules-foreign-cc)

Rules for building C/C++ projects using foreign build systems inside Bazel projects.

* <span style="color:red">**Experimental** - API will most definitely change.</span>
* This is not an officially supported Google product
(meaning, support and/or new releases may be limited.)

**NOTE**: this requires Bazel version starting from 0.17.1.

It also requires passing Bazel the following flag:
(**please pay attention @foreign_cc_impl was added** due to adoption to Starlark API changes that are goning to happen in Bazel 0.21)
```
--experimental_cc_skylark_api_enabled_packages=@rules_foreign_cc//tools/build_defs,tools/build_defs,@foreign_cc_impl
```
Where ```rules_foreign_cc``` is the name of this repository in your WORKSPACE file.

## Building CMake projects:

- Build libraries/binaries with CMake from sources using cmake_external rule
- Use cmake_external targets in cc_library, cc_binary targets as dependency
- Bazel cc_toolchain parameters are used inside cmake_external build
- See full list of cmake_external arguments below 'example'
- cmake_external is defined in ./tools/build_defs
- Works on Ubuntu, Mac OS and Windows(* see special notes below in Windows section) operating systems

**Example:**
(Please see full examples in ./examples)
<br/>The example for **Windows** is below, in the section 'Usage on Windows'.

* In `WORKSPACE`, we use a `http_archive` to download tarballs with the libraries we use.
* In `BUILD`, we instantiate a `cmake_external` rule which behaves similarly to a `cc_library`, which can then be used in a C++ rule (`cc_binary` in this case).

In `WORKSPACE`, put

```python
workspace(name = "rules_foreign_cc_usage_example")

load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

# Group the sources of the library so that CMake rule have access to it
all_content = """filegroup(name = "all", srcs = glob(["**"]), visibility = ["//visibility:public"])"""

# Rule repository
http_archive(
   name = "rules_foreign_cc",
   strip_prefix = "rules_foreign_cc-master",
   url = "https://github.com/bazelbuild/rules_foreign_cc/archive/master.zip",
)

load("@rules_foreign_cc//:workspace_definitions.bzl", "rules_foreign_cc_dependencies")

# Workspace initialization function; includes repositories needed by rules_foreign_cc,
# and creates some utilities for the host operating system
rules_foreign_cc_dependencies()

# OpenBLAS source code repository
http_archive(
   name = "openblas",
   build_file_content = all_content,
   strip_prefix = "OpenBLAS-0.3.2",
   urls = ["https://github.com/xianyi/OpenBLAS/archive/v0.3.2.tar.gz"],
)

# Eigen source code repository
http_archive(
   name = "eigen",
   build_file_content = all_content,
   strip_prefix = "eigen-git-mirror-3.3.5",
   urls = ["https://github.com/eigenteam/eigen-git-mirror/archive/3.3.5.tar.gz"],
)
```

and in `BUILD`, put

```python
load("@rules_foreign_cc//tools/build_defs:cmake.bzl", "cmake_external")

cmake_external(
   name = "openblas",
   # Values to be passed as -Dkey=value on the CMake command line;
   # here are serving to provide some CMake script configuration options
   cache_entries = {
       "NOFORTRAN": "on",
       "BUILD_WITHOUT_LAPACK": "no",
   },
   lib_source = "@openblas//:all",

   # We are selecting the resulting static library to be passed in C/C++ provider
   # as the result of the build;
   # However, the cmake_external dependants could use other artefacts provided by the build,
   # according to their CMake script
   static_libraries = ["libopenblas.a"],
)

cmake_external(
   name = "eigen",
   # These options help CMake to find prebuilt OpenBLAS, which will be copied into
   # $EXT_BUILD_DEPS/openblas by the cmake_external script
   cache_entries = {
       "BLA_VENDOR": "OpenBLAS",
       "BLAS_LIBRARIES": "$EXT_BUILD_DEPS/openblas/lib/libopenblas.a",
   },
   headers_only = True,
   lib_source = "@eigen//:all",
   # Dependency on other cmake_external rule; can also depend on cc_import, cc_library rules
   deps = [":openblas"],
)
```

then build as usual:

```bash
$ devbazel build //examples/cmake_pcl:eigen
```

**Usage on Windows**

When using on Windows, you should start Bazel in MSYS2 shell, as the shell script inside cmake_external assumes this.
Also, you should explicitly specify **make commands and option to generate CMake crosstool file**.<br/>
The default generator for CMake will be detected automatically, or you can specify it explicitly.
<br/>**The tested generators:** Visual Studio 15, Ninja and NMake.
The extension '.lib' is assumed for the static libraries by default.   

Example usage (see full example in ./examples/cmake_hello_world_lib):
Example assumes that MS Visual Studio and Ninja are installed on the host machine, and Ninja bin directory is added to PATH.

```python
cmake_external(
    # expect to find ./lib/hello.lib as the result of the build
    name = "hello",
    # This option can be omitted
    cmake_options = ["-G \"Visual Studio 15 2017 Win64\""],
    generate_crosstool_file = True,
    lib_source = ":srcs",
    # .vcxproj or .sln file must be specified argument, as multiple files are generated by CMake
    make_commands = ["MSBuild.exe INSTALL.vcxproj"],
)

cmake_external(
    name = "hello_ninja",
    # expect to find ./lib/hello.lib as the result of the build
    lib_name = "hello",
    # explicitly specify the generator
    cmake_options = ["-GNinja"],
    generate_crosstool_file = True,
    lib_source = ":srcs",
    # specify to call ninja after configuring
    make_commands = [
        "ninja",
        "ninja install",
    ],
)

cmake_external(
    name = "hello_nmake",
    # explicitly specify the generator
    cmake_options = ["-G \"NMake Makefiles\""],
    generate_crosstool_file = True,
    lib_source = ":srcs",
    # specify to call nmake after configuring
    make_commands = [
        "nmake",
        "nmake install",
    ],
    # expect to find ./lib/hello.lib as the result of the build
    static_libraries = ["hello.lib"]
)
```

**cmake_external arguments:**

Mandatory arguments:
```name, lib_source```

```python
attrs: {
    # CMake only:
    #
    # Relative install prefix to be passed to CMake in -DCMAKE_INSTALL_PREFIX
    "install_prefix": attr.string(mandatory = False),
    # CMake cache entries to initialize (they will be passed with -Dkey=value)
    # Values, defined by the toolchain, will be joined with the values, passed here.
    # (Toolchain values come first)
    "cache_entries": attr.string_dict(mandatory = False, default = {}),
    # CMake environment variable values to join with toolchain-defined.
    # For example, additional CXXFLAGS.
    "env_vars": attr.string_dict(mandatory = False, default = {}),
    # Other CMake options
    "cmake_options": attr.string_list(mandatory = False, default = []),
    # When True, CMake crosstool file will be generated from the toolchain values,
    # provided cache-entries and env_vars (some values will still be passed as -Dkey=value
    # and environment variables).
    # If CMAKE_TOOLCHAIN_FILE cache entry is passed, specified crosstool file will be used
    # When using this option, it makes sense to specify CMAKE_SYSTEM_NAME in the
    # cache_entries - the rule makes only a poor guess about the target system,
    # it is better to specify it manually.
    "generate_crosstool_file": attr.bool(mandatory = False, default = False),
    #
    # From framework.bzl:
    # 
    # Library name. Defines the name of the install directory and the name of the static library,
    # if no output files parameters are defined (any of static_libraries, shared_libraries,
    # interface_libraries, binaries_names)
    # Optional. If not defined, defaults to the target's name.
    "lib_name": attr.string(mandatory = False),
    # Label with source code to build. Typically a filegroup for the source of remote repository.
    # Mandatory.
    "lib_source": attr.label(mandatory = True, allow_files = True),
    # Optional compilation definitions to be passed to the dependencies of this library.
    # They are NOT passed to the compiler, you should duplicate them in the configuration options.
    "defines": attr.string_list(mandatory = False, default = []),
    #
    # Optional additional inputs to be declared as needed for the shell script action.
    # Not used by the shell script part in cc_external_rule_impl.
    "additional_inputs": attr.label_list(mandatory = False, allow_files = True, default = []),
    # Optional additional tools needed for the building.
    # Not used by the shell script part in cc_external_rule_impl.
    "additional_tools": attr.label_list(mandatory = False, allow_files = True, default = []),
    #
    # Optional part of the shell script to be added after the make commands
    "postfix_script": attr.string(mandatory = False),
    # Optinal make commands, defaults to ["make", "make install"]
    "make_commands": attr.string_list(mandatory = False, default = ["make", "make install"]),
    #
    # Optional dependencies to be copied into the directory structure.
    # Typically those directly required for the external building of the library/binaries.
    # (i.e. those that the external buidl system will be looking for and paths to which are
    # provided by the calling rule)
    "deps": attr.label_list(mandatory = False, allow_files = True, default = []),
    # Optional tools to be copied into the directory structure.
    # Similar to deps, those directly required for the external building of the library/binaries.
    "tools_deps": attr.label_list(mandatory = False, allow_files = True, default = []),
    #
    # Optional name of the output subdirectory with the header files, defaults to 'include'.
    "out_include_dir": attr.string(mandatory = False, default = "include"),
    # Optional name of the output subdirectory with the library files, defaults to 'lib'.
    "out_lib_dir": attr.string(mandatory = False, default = "lib"),
    # Optional name of the output subdirectory with the binary files, defaults to 'bin'.
    "out_bin_dir": attr.string(mandatory = False, default = "bin"),
    #
    # Optional. if true, link all the object files from the static library,
    # even if they are not used.
    "alwayslink": attr.bool(mandatory = False, default = False),
    # Optional link options to be passed up to the dependencies of this library
    "linkopts": attr.string_list(mandatory = False, default = []),
    #
    # Output files names parameters. If any of them is defined, only these files are passed to
    # Bazel providers.
    # if no of them is defined, default lib_name.a/lib_name.lib static library is assumed.
    #
    # Optional names of the resulting static libraries.
    "static_libraries": attr.string_list(mandatory = False),
    # Optional names of the resulting shared libraries.
    "shared_libraries": attr.string_list(mandatory = False),
    # Optional names of the resulting interface libraries.
    "interface_libraries": attr.string_list(mandatory = False),
    # Optional names of the resulting binaries.
    "binaries": attr.string_list(mandatory = False),
    # Flag variable to indicate that the library produces only headers
    "headers_only": attr.bool(mandatory = False, default = False),
  }
```

## Design document:

[External C/C++ libraries rules](https://docs.google.com/document/d/1Gv452Vtki8edo_Dj9VTNJt5DA_lKTcSMwrwjJOkLaoU/edit?usp=sharing) 