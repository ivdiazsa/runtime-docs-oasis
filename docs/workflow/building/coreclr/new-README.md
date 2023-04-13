# Building CoreCLR

* [Introduction](#introduction)
* [Building the Runtime](#building-the-runtime)
  * [Linux Notes](#linux-notes)
  * [Additional Configuration Stuff](#additional-configuration-stuff)
    * [Drivers for the Build](#drivers-for-the-build)
    * [Extra Flags](#extra-flags)
  * [Build Results Layout](#build-results-layout)/docs/workflow/testing/coreclr/testing.md
  * [Cross-Compilation](#cross-compilation)
    * [Windows Native ARM64 (Experimental)](#windows-native-arm64-experimental)
    * [Intel and Apple Silicon for macOS](#intel-and-apple-silicon-for-macos)
    * [Linux Cross-Compilation](#linux-cross-compilation)
* [The Core\_Root and Testing](#the-core_root-and-testing)

## Introduction

Here is a brief overview on how to build the common form of CoreCLR in general.

First and foremost, ensure you have all the requirements satisfied for your specific platform. These are found in the [general workflow doc](/docs/workflow/README.md#build-requirements).

<!-- NOTES FOR LATER: Right now, it only specifies CoreCLR. We might probably need to add some information about libs, mono, host, etc. -->
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

### Linux Notes

<!-- Add link to Docker instructions. -->
To build for Linux, you can use Docker with our pre-built images, or you can use your own Linux environment.

For your own environment, besides having all the necessary [requirements](/docs/workflow/requirements/linux-requirements.md), you need to set the necessary number of file-handles, if it's not set already. To verify this, run the command `sysctl fs.file-max` in your terminal. If it's less than 100000, then add `fs.file-max = 100000` to `/etc/sysctl.conf`, and then run `sudo sysctl -p`.

### Additional Configuration Stuff

There are a few options you can use to tweak how the repo is built.

#### Drivers for the Build

If you want to use _Ninja_ to drive the native build instead of _Make_ on non-Windows platforms, you can pass the `-ninja` flag to the build script as follows:

```bash
./build.sh --subset clr --ninja
```

If you want to use Visual Studio's _MSBuild_ to drive the native build on Windows, you can pass the `-msbuild` flag to the build script similarly to the `-ninja` flag.

We recommend using _Ninja_ for building the project on Windows since it more efficiently uses the build machine's resources for the native runtime build in comparison to Visual Studio's _MSBuild_.

#### Extra Flags

To pass extra compiler/linker flags to the coreclr build, set the environment variables `EXTRA_CFLAGS`, `EXTRA_CXXFLAGS` and `EXTRA_LDFLAGS` as needed. Don't set `CFLAGS`/`CXXFLAGS`/`LDFLAGS` directly as that might lead to configure-time tests failing.

### Build Results Layout

After the build has completed, the produced output artifacts are organized in a very specific structure, which is detailed down below.

* The product binaries are placed in the `artifacts/bin/coreclr/<os>.<arch>.<configuration>` folder. The most important ones for the runtime are the following:
  * _`corerun.exe` on Windows/`corerun` on macOS and Linux_: The command line host. This program loads and starts the CoreCLR runtime, and processes the managed program (e.g. `program.dll`) you want to run with it.
  * _`coreclr.dll` on Windows/`libcoreclr.so` on Linux/`libcoreclr.dylib` on macOS_: The CoreCLR runtime itself.
  * _`System.Private.CoreLib.dll` (same on all OS's)_: The core managed library, containing the definitions of `Object` and base functionality.
* The _CoreCLR Nuget Package_ called `Microsoft.Dotnet.CoreCLR` is placed in the `artifacts/bin/coreclr/<os>.<arch>.<configuration>/.nuget` folder.
* The build logs are placed in the `artifacts/log` folder. Additionally, if you requested binary logs by passing the `-bl`/`--binaryLog` flag, they will be placed under this path as well. These are very useful when diagnosing build failures.
* The build requires some intermediate output while running. This is all placed in the `artifacts/obj/coreclr` folder.

If you want to force a full rebuild of the subsets you specified when calling the build script, pass the `--rebuild` flag to it, in addition to any other arguments you might require.

### Cross-Compilation

The repo also supports certain combinations to build targeting a different architecture than the host machine. These are explained in the following sections.

#### Windows Native ARM64 (Experimental)

Building natively on ARM64 requires you to have installed the appropriate ARM64 build tools and Windows SDK, as specified in the [Windows requirements doc](/docs/workflow/requirements/windows-requirements.md).

Once those requirements are satisfied, you have to specify you are doing an Arm64 build, and explicitly tell the build script you want to use `MSBuild`. `Ninja` is not yet supported on Arm64 platforms.

```cmd
build.cmd -s clr -c Release -arch arm64 -msbuild
```

Since this is still in an experimental phase, the recommended way for building ARM64 is cross-compiling from an x64 machine. Instructions on how to do this can be found at the [cross-building doc](/docs/workflow/building/coreclr/cross-building.md#cross-compiling-for-arm32-and-arm64).

#### Intel and Apple Silicon for macOS

It is possible to get a macOS ARM64 build using an Intel x64 Mac, and vice versa, an x64 one using an M1/M2 Mac. Instructions on how to do this are in the [cross-building doc](/docs/workflow/building/coreclr/cross-building.md#macos-cross-building).

#### Linux Cross-Compilation

Linux also supports doing cross-builds for ARM32 and ARM64. You can do it using the according Docker images following the instructions here, or you can use your own environment following the instructions [here](/docs/workflow/building/coreclr/cross-building.md#linux-cross-building).

## The Core\_Root and Testing

The Core_Root provides one of the main ways to test your build. Full instructions on how to build it in the [CoreCLR testing doc](/docs/workflow/testing/coreclr/testing.md), and we also have a detailed guide on how to use it for your own testing in [its own dedicated doc](/docs/workflow/testing/using-corerun-and-coreroot.md).

For general testing, the [testing docs](/docs/workflow/testing/coreclr/testing.md) have detailed instructions on how to do it.
