# Building CoreCLR

* [Introduction](#introduction)
* [Building the Runtime](#building-the-runtime)
  * [Drivers for the Build](#drivers-for-the-build)
  * [Extra Flags](#extra-flags)

## Introduction

Here is a brief overview on how to build the common form of CoreCLR in general.

To build the CoreCLR subset, you have to specify `clr` as part of the `--subset` flag when calling the build script at the root of the repo. Note that specifying the subset flag explicitly is not necessary when it is the first argument (i.e. `./build.sh --subset clr` and `./build.sh clr` are equivalent). However, if you pass any other argument beforehand, then you must specify it.

The command for just CoreCLR would look like the following examples:

For Linux and macOS:

```bash
./build.sh --subset clr <other args>
```

For Windows:

```cmd
.\build.cmd -subset clr <other args>
```

## Building the Runtime

By default, the script generates a _Debug_ build type, which is not optimized code and includes asserts. As its name suggests, this makes it easier and friendlier to walk through and debug the code. If you want to make performance measurements, you ought to build the _Release_ version instead, which doesn't have any asserts and has all code optimizations enabled. Likewise, if you plan on running tests, the _Release_ configuration is more suitable since it's considerably faster than the _Debug_ one.

To build this way, you tell the build script by adding the flag `-configuration` with the `release` value (or `-c release` for the shorthanded version). For example:

```bash
./build.sh --subset clr --configuration release
```

As mentioned before in the [general building document](/docs/workflow/README.md#configurations-and-subsets), CoreCLR also supports a _Checked_ build type which has asserts enabled like _Debug_, but is built with the native compiler optimizer enabled, so it runs much faster. This is the usual mode used for running tests in the CI system.

Now, it is also possible to select a different configuration for each subset when building them together. The `--configuration` flag applies universally to all subsets, but it can be overridden with any one or more of the following ones:

* `--runtimeConfiguration (-rc)`: Flag for the CLR build configuration.
* `--librariesConfiguration (-lc)`: Flag for the libraries build configuration.
* `--hostConfiguration (-hc)`: Flag for the host build configuration.

For example, a very common scenario used by developers and the repo's test scripts with default options, is to build the _clr_ in _Debug_ mode, and the _libraries_ in _Release_ mode. To achieve this, the command-line would look like the following:

```bash
./build.sh --subset clr+libs --configuration Release --runtimeConfiguration Debug
```

Or alternatively:

```bash
./build.sh --subset clr+libs --librariesConfiguration Release --runtimeConfiguration Debug
```

For more information about all the different options available, supply the argument `-help|-h` when calling the build script. On Unix-like systems, non-abbreviated arguments can be passed in with a single `-` or double hyphen `--`.

### Drivers for the Build

If you want to use _Ninja_ to drive the native build instead of _Make_ on non-Windows platforms, you can pass the `-ninja` flag to the build script as follows:

```bash
./build.sh --subset clr --ninja
```

If you want to use Visual Studio's _MSBuild_ to drive the native build on Windows, you can pass the `-msbuild` flag to the build script similarly to the `-ninja` flag.

We recommend using _Ninja_ for building the project on Windows since it more efficiently uses the build machine's resources for the native runtime build in comparison to Visual Studio's _MSBuild_.

### Extra Flags

To pass extra compiler/linker flags to the coreclr build, set the environment variables `EXTRA_CFLAGS`, `EXTRA_CXXFLAGS` and `EXTRA_LDFLAGS` as needed. Don't set `CFLAGS`/`CXXFLAGS`/`LDFLAGS` directly as that might lead to configure-time tests failing.

### Build Results Layout

After the build has completed, the produced output artifacts are organized in a very specific structure, which is detailed down below.

* The product binaries are placed in the `artifacts/bin/coreclr/<os>.<arch>.<configuration>` folder. The most important ones for the runtime are the following:
    * _`corerun.exe` on Windows/`corerun` on macOS and Linux_: The command line host. This program loads and starts the CoreCLR runtime, and processes the managed program (e.g. `program.dll`) you want to run with it.
    * _`coreclr.dll` on Windows/`libcoreclr.so` on Linux/`libcoreclr.dylib` on macOS_: The CoreCLR runtime itself.
    * _`System.Private.CoreLib.dll` (same on all OS's)_: The core managed library, containing the definitions of `Object` and base functionality.
* The _CoreCLR Nuget Package_ called `Microsoft.Dotnet.CoreCLR` is placed in the `artifacts/bin/coreclr/<os>.<arch>.<configuration>/.nuget folder.
* The build logs are placed in the `artifacts/log` folder. Additionally, if you requested binary logs by passing the `-bl`/`--binaryLog` flag, they will be placed under this path as well. These are very useful when diagnosing build failures.
* The build requires some intermediate output while running. This is all placed in the `artifacts/obj/coreclr` folder.

If you want to force a full rebuild of the subsets you specified when calling the build script, pass the `--rebuild` flag to it, in addition to any other arguments you might require.

