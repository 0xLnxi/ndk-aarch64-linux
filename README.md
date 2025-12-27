# How to Build on Aarch64 Linux

[![Build NDK for aarch64-linux](https://github.com/0xLnxi/ndk-aarch64-linux/actions/workflows/build-ndk.yml/badge.svg)](https://github.com/0xLnxi/ndk-aarch64-linux/actions/workflows/build-ndk.yml)

## Automated Builds with GitHub Actions

This repository includes a GitHub Actions workflow that automatically downloads and packages the Android NDK for aarch64-linux platforms.

### Available NDK Versions

The workflow supports two NDK versions:

#### NDK r27d (LTS)
- **Version**: 27.3.13750724
- **Download**: android-ndk-r27d-linux.zip (663,956,036 bytes)
- **SHA1**: 22105e410cf29afcf163760cc95522b9fb981121

#### NDK r29 (Stable)
- **Version**: 29.0.14206865
- **Download**: android-ndk-r29-linux.zip (783,549,481 bytes)
- **SHA1**: 87e2bb7e9be5d6a1c6cdf5ec40dd4e0c6d07c30

### Using Pre-built NDK Packages

You can download pre-built NDK packages from the [GitHub Actions artifacts](https://github.com/0xLnxi/ndk-aarch64-linux/actions) or [Releases](https://github.com/0xLnxi/ndk-aarch64-linux/releases) page.

To use a pre-built package:
1. Download the `ndk-r27d-aarch64-linux.tar.gz` or `ndk-r29-aarch64-linux.tar.gz` file
2. Extract it: `tar xzf ndk-<version>-aarch64-linux.tar.gz`
3. Follow the manual build instructions below to complete the setup

## Manual Build Instructions

## Clone Source Code

```shell
cd ${SOURCE_ROOT}
repo init -u https://android.googlesource.com/platform/manifest -b llvm-toolchain
repo sync -c
```
## Prepare Environment

Install Required Packages

```shell
apt install libc++-dev bison ninja cmake
```
Create Symbolic Links
```shell
cd ${SOURCE_ROOT}/prebuilts/build-tools/linux-arm64/bin
ln -sf /bin/bison .
ln -sf /bin/ninja .

cd ${SOURCE_ROOT}/prebuilts/clang/host
ln -s /usr linux-arm64

cd ${SOURCE_ROOT}/prebuilts/cmake
ln -s /usr linux-arm64

cd ${SOURCE_ROOT}/prebuilts/python
ln -s /usr linux-arm64
```

## Modify Build Scripts
${SOURCE_ROOT}/toolchain/llvm_android/do_build.py
### Disable stage1 and set stage2 toolchain to system toolchain
Comment out the following lines and add 'set_default_toolchain(toolchains.Toolchain(Path("/usr"),Path("/usr")))'
```python
# if not args.bootstrap_use_prebuilt and not args.bootstrap_use:
#     stage1 = builders.Stage1Builder(host_configs)
#     stage1.build_name = 'stage1'
#     stage1.svn_revision = android_version.get_svn_revision()
#     # Build lldb for lldb-tblgen. It will be used to build lldb-server and windows lldb.
#     stage1.build_lldb = build_lldb
#     stage1.enable_mlgo = mlgo
#     stage1.build_extra_tools = args.run_tests_stage1
#     # In a debug or instrumented build, stage1 toolchain builds the stage2
#     # runtimes.  We need cross runtimes in stage1 so CMake can find musl libc++
#     # for stage2 musl runtimes.
#     stage1.build_cross_runtimes = hosts.build_host().is_linux and (args.debug or instrumented)
#     stage1.libzstd = libzstd_builder
#     stage1.build()
#     if hosts.build_host().is_linux:
#         add_header_links('stage1', host_config=configs.host_config(musl))
#     # stage1 test is off by default, turned on by --run-tests-stage1,
#     # and suppressed by --skip-tests.
#     if not args.skip_tests and args.run_tests_stage1:
#         stage1.test()
#     set_default_toolchain(stage1.installed_toolchain)
# if args.bootstrap_use:
#     # Remove previous install directories, since the bootstrap compiler
#     # will overwrite install directories.
#     if (paths.OUT_DIR / 'stage1-install').exists():
#         shutil.rmtree(paths.OUT_DIR / 'stage1-install')
#     if (paths.OUT_DIR / 'stage2-install').exists():
#         shutil.rmtree(paths.OUT_DIR / 'stage2-install')
#     with timer.Timer('extract_bootstrap'):
#         utils.extract_tarball(paths.OUT_DIR, args.bootstrap_use)
#     # If we were to use the full build as bootstrap, we need to rename it to stage-install.
#     if (paths.OUT_DIR / 'stage2-install').exists():
#         (paths.OUT_DIR / 'stage2-install').rename(paths.OUT_DIR / 'stage1-install')
#     set_default_toolchain(toolchains.Toolchain(paths.OUT_DIR / 'stage1-install', paths.OUT_DIR / 'stage1'))
# if args.bootstrap_build_only:
#     with timer.Timer('package_bootstrap'):
#         utils.create_tarball(paths.OUT_DIR, ['stage1', 'stage1-install'], paths.DIST_DIR / 'stage1-install.tar.xz')
#     return
set_default_toolchain(toolchains.Toolchain(Path("/usr"),Path("/usr")))
```
### Disable building libxml2 and xz
Comment out the following lines:
```python

     if build_lldb:
         # Swig is needed for both host and windows lldb.
@@ -1142,7 +1143,7 @@ def main():
         stage2.libzstd = libzstd_builder
 
         libxml2_builder = builders.LibXml2Builder(host_configs)
-        libxml2_builder.build()
+        # libxml2_builder.build()
         stage2.libxml2 = libxml2_builder
 
         stage2.build_lldb = build_lldb
@@ -1150,7 +1151,7 @@ def main():
             stage2.swig_executable = swig_builder.install_dir / 'bin' / 'swig'
 
             xz_builder = builders.XzBuilder(host_configs)
-            xz_builder.build()
+            # xz_builder.build()
             stage2.liblzma = xz_builder
 
             libncurses = builders.LibNcursesBuilder(host_configs)
```
### Modify host tag
${SOURCE_ROOT}/toolchain/llvm_android/py3_utils.py

Change the host tag from linux-x86to linux-arm64:
```python
 THIS_DIR = os.path.realpath(os.path.dirname(__file__))
 def get_host_tag():
     if sys.platform.startswith('linux'):
-        return 'linux-x86'
+        return 'linux-arm64'
     if sys.platform.startswith('darwin'):
         return 'darwin-x86'
     raise RuntimeError('Unsupported host: {}'.format(sys.platform))
```

### Add CMake build type configuration
${SOURCE_ROOT}/toolchain/llvm_android/src/llvm_android/builders.py

Add the release build type configuration:
```python
class Stage2Builder(base_builders.LLVMBuilder):
 
         if self.debug_build:
             defines['CMAKE_BUILD_TYPE'] = 'Debug'
+        else:
+            defines['CMAKE_BUILD_TYPE'] = 'Release'
 
         if self.build_instrumented:
             defines['LLVM_BUILD_INSTRUMENTED'] = 'ON'
```

### Update Linux configuration for ARM64
${SOURCE_ROOT}/toolchain/llvm_android/src/llvm_android/configs.py
Change the configuration to use system toolchain and set ARM64 triple:
```python
class LinuxConfig(_GccConfig):
     """Configuration for Linux targets."""
 
     target_os: hosts.Host = hosts.Host.Linux
-    sysroot: Optional[Path] = paths.GCC_ROOT / 'host' / 'x86_64-linux-glibc2.17-4.8' / 'sysroot'
-    gcc_root: Path = paths.GCC_ROOT / 'host' / 'x86_64-linux-glibc2.17-4.8'
-    gcc_triple: str = 'x86_64-linux'
+    sysroot: Optional[Path] = Path('/')
+    gcc_root: Path = Path('/')
+    gcc_triple: str = 'aarch64-linux'
     gcc_ver: str = '4.8.3'
     is_cross_compiling: bool = False
     is_musl: bool = False
 
     @property
     def llvm_triple(self) -> str:
-        return 'i386-unknown-linux-gnu' if self.is_32_bit else 'x86_64-unknown-linux-gnu'
+        return 'i386-unknown-linux-gnu' if self.is_32_bit else 'aarch64-unknown-linux-gnu'
 
     @property
     def cflags(self) -> List[str]:
```
### Update host OS tag mapping
${SOURCE_ROOT}/toolchain/llvm_android/src/llvm_android/hosts.py
Change Linux host tag to linux-arm64:

```python
class Host(enum.Enum):
         """Returns the os tag of current Host."""
         return {
             Host.Darwin: 'darwin-x86',
-            Host.Linux: 'linux-x86',
+            Host.Linux: 'linux-arm64',
             Host.Windows: 'windows-x86',
         }[self]
```
### Update paths for ARM64
${SOURCE_ROOT}/toolchain/llvm_android/src/llvm_android/paths.py

Modify the paths to use the system toolchain:
```python
@@ -35,8 +35,7 @@ TOOLCHAIN_DIR: Path = ANDROID_DIR / 'toolchain'
 TOOLCHAIN_UTILS_DIR: Path = EXTERNAL_DIR / 'toolchain-utils'
 TOOLCHAIN_LLVM_PATH: Path = TOOLCHAIN_DIR / 'llvm-project'
 
-CLANG_PREBUILT_DIR: Path = (PREBUILTS_DIR / 'clang' / 'host' / hosts.build_host().os_tag
-                            / constants.CLANG_PREBUILT_VERSION)
+CLANG_PREBUILT_DIR: Path = (PREBUILTS_DIR / 'clang' / 'host' / hosts.build_host().os_tag)
 CLANG_PREBUILT_LIBCXX_HEADERS: Path = CLANG_PREBUILT_DIR / 'include' / 'c++' / 'v1'
 WINDOWS_CLANG_PREBUILT_DIR: Path = (PREBUILTS_DIR / 'clang' / 'host' / 'windows-x86'
                                     / constants.CLANG_PREBUILT_VERSION)
@@ -55,7 +54,7 @@ M4_BIN_PATH: Path = BUILD_TOOLS_DIR / hosts.build_host().os_tag / 'bin' / 'm4'
 MAKE_BIN_PATH: Path = BUILD_TOOLS_DIR / hosts.build_host().os_tag / 'bin' / 'make'
 # Use the musl version of ninja on Linux, it is statically linked and avoids
 # problems with LD_LIBRARY_PATH causing ninja to use the wrong libc++.so.
-NINJA_BIN_PATH: Path = BUILD_TOOLS_DIR / hosts.build_host().os_tag_musl / 'bin' / 'ninja'
+NINJA_BIN_PATH: Path = BUILD_TOOLS_DIR / hosts.build_host().os_tag / 'bin' / 'ninja'
 
 LIBEDIT_SRC_DIR: Path = EXTERNAL_DIR / 'libedit'
 LIBNCURSES_SRC_DIR: Path = EXTERNAL_DIR / 'libncurses'
```

## Build the Toolchain
```shell
cd ${SOURCE_ROOT}
python3 toolchain/llvm_android/build.py --no-build windows --skip-tests --no-musl
```

## View Complete Changes
All modifications are available in the `diff.txt` file.

## Modify android-ndk-r29-linux.zip
```shell
cd ${R29}/prebuilt/linux-x86_64/bin
ln -sf /bin/make .
ln -sf /bin/yasm .
ln -sf /bin/ytasm .

cd ${R29}/toolchains/llvm/prebuilt/linux-x86_64/bin
cp -f ${SOURCE_ROOT}/out/stage2/bin/* .

cd ${R29}/toolchains/llvm/prebuilt/linux-x86_64/lib/clang
cp -r ${SOURCE_ROOT}/out/stage2/lib/clang/22 .
cd 22/lib
cp ${R29}/toolchains/llvm/prebuilt/linux-x86_64/lib/clang/21/lib/linux .

nano ${R29}/build/tools/ndk_bin_common.sh
# add aarch64 to case
HOST_ARCH=$(uname -m)
case $HOST_ARCH in
  arm64) HOST_ARCH=arm64;;
  i?86) HOST_ARCH=x86;;
  x86_64|amd64|aarch64) HOST_ARCH=x86_64;;
  *) echo "ERROR: Unknown host CPU architecture: $HOST_ARCH"
     exit 1
esac

cd ${R29}/toolchains/llvm/prebuilt/linux-x86_64/python3/bin
ln -sf /bin/python3 pyhton3.11
```