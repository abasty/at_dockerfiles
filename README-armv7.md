# Experiments with armv7

## Initial goal

The goal is to build Pi/armhf Dart self-contained executables from a Linux x64
host using multi-arch Docker, buildx, qemu.

The multi-arch Docker solution works well for the arm64 arch. armhf has some
problems.

To clone this file and the associated Dockerfile :

```
$ git clone https://github.com/abasty/at_dockerfiles.git
$ git checkout experiment/armv7
```

## Identified problems & possible solutions

* Certificates : install the right certificates before downloading Dart SDK
* Unrecognized ARM CPU architecture : fake Pi `/proc/cpuinfo` see
  <https://github.com/moby/moby/issues/16423>
* Filesystem inode 32 guest on 64 host (qemu): use `glibc` <= `2.27`
  <https://bugs.launchpad.net/qemu/+bug/1805913>. Debian Jessie has `2.19`

## Experiments

### Certificates problem

Seems that `debian/armv7` does not install the required certificates to download
the official Dart SDK packages.

To bypass this problem, we added the `-k` option to the `curl` command line :

```
curl -kfLO "$URL";
```

### "Unrecognized ARM CPU architecture" problem

From the top directory of this repo, build the armhf docker image :

```
$ git checkout experiment/armv7
$ docker buildx build --platform linux/arm/v7 -t atsigncompany/buildimage:dart-armv7 \
  -f at-buildimage/Dockerfile .
```

We can run an interactive container using this docker image :

```
$ docker run -ti atsigncompany/buildimage:dart-armv7
WARNING: The requested image's platform (linux/arm/v7) does not match the detected host platform (linux/amd64) and no specific platform was requested
root@96facecd1419:~# dart
../../runtime/vm/cpu_arm.cc: 151: error: Unrecognized ARM CPU architecture.
version=2.13.0 (stable) (Wed May 12 12:45:49 2021 +0200) on "linux_arm"
pid=17, thread=17, isolate_group=vm-isolate(0xfcf12000), isolate=vm-isolate(0xfcf28480)
isolate_instructions=ffa201c0, vm_instructions=ffa201c0
  pc 0xffbc15b5 fp 0xfe1e4a70 dart::Profiler::DumpStackTrace(void*)+0x4c
  pc 0xffa202af fp 0xfe1e4a80 dart::Assert::Fail(char const*, ...)+0x1e
  pc 0xffb1fe89 fp 0xfe1e4aa8 dart::HostCPUFeatures::Init()+0x17c
  pc 0xffb20419 fp 0xfe1e4bb8 dart::Dart::Init(unsigned char const*, unsigned char const*, _Dart_Isolate* (*)(char const*, char const*, char const*, char const*, Dart_IsolateFlags*, void*, char**), bool (*)(void**, char**), void (*)(void*, void*), void (*)(void*, void*), void (*)(void*), void (*)(), void* (*)(char const*, bool), void (*)(unsigned char**, int*, void*), void (*)(void const*, int, void*), void (*)(void*), bool (*)(unsigned char*, int), _Dart_Handle* (*)(), bool, Dart_CodeObserver*)+0x338
  pc 0xffec5525 fp 0xfe1e4c18 Dart_Initialize+0x54
  pc 0xffa0951b fp 0xfe1e4cd0 dart::bin::main(int, char**)+0x2be
  pc 0xffa0a193 fp 0xfe1e4ce0 main+0xa
-- End of DumpStackTrace
qemu: uncaught target signal 6 (Aborted) - core dumped
Aborted (core dumped)
```

In `runtime/vm/cpu_arm.cc` :

```cpp
  // Check for ARMv7, or aarch64.
  // It can be in either the Processor or Model information fields.
  if (CpuInfo::FieldContains(kCpuInfoProcessor, "aarch64") ||
      CpuInfo::FieldContains(kCpuInfoModel, "aarch64") ||
      CpuInfo::FieldContains(kCpuInfoArchitecture, "8") ||
      CpuInfo::FieldContains(kCpuInfoArchitecture, "AArch64")) {
    // pretend that this arm64 cpu is really an ARMv7
    is_arm64 = true;
  } else if (!CpuInfo::FieldContains(kCpuInfoProcessor, "ARMv7") &&
             !CpuInfo::FieldContains(kCpuInfoModel, "ARMv7") &&
             !CpuInfo::FieldContains(kCpuInfoArchitecture, "7")) {
#if !defined(DART_RUN_IN_QEMU_ARMv7)
    FATAL("Unrecognized ARM CPU architecture.");
#endif
  }
```

### Building a new SDK

I removed this fatal line and built a new Dart arm SDK **on the x64 host**,
following instructions on :

* <https://github.com/dart-lang/sdk/wiki/Building>
* <https://github.com/dart-lang/sdk/wiki/Building-Dart-SDK-for-ARM-processors>

Downloading sources :

```
$ sudo apt-get install g++-multilib git python3 curl
$ sudo apt-get install g++-arm-linux-gnueabihf
$ git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
$ export PATH="$PATH:$PWD/depot_tools"
$ mkdir dart-sdk
$ cd dart-sdk
```

You should have a quick connection here or launch the process and go to beach :

```
$ fetch dart
Running: gclient root
...
Receiving objects:   8% (102826/1145300)
...
Syncing projects: 100% (110/110), done.

________ running 'python3 sdk/build/linux/sysroot_scripts/install-sysroot.py --arch i386' in '/home/alain/Projects/articles/travail/dart-sdk'
Installing Debian Jessie i386 root image: /home/alain/Projects/articles/travail/dart-sdk/sdk/build/linux/debian_jessie_i386-sysroot
Downloading https://commondatastorage.googleapis.com/chrome-linux-sysroot/toolchain/7031a828c5dcedc937bbf375c42daab08ca6162f/debian_jessie_i386_sysroot.tgz
Hook 'python3 sdk/build/linux/sysroot_scripts/install-sysroot.py --arch i386' took 65.02 secs
Running hooks:  33% ( 2/ 6) sysroot_amd64
________ running 'python3 sdk/build/linux/sysroot_scripts/install-sysroot.py --arch amd64' in '/home/alain/Projects/articles/travail/dart-sdk'
Installing Debian Jessie amd64 root image: /home/alain/Projects/articles/travail/dart-sdk/sdk/build/linux/debian_jessie_amd64-sysroot
Downloading https://commondatastorage.googleapis.com/chrome-linux-sysroot/toolchain/7031a828c5dcedc937bbf375c42daab08ca6162f/debian_jessie_amd64_sysroot.tgz
Hook 'python3 sdk/build/linux/sysroot_scripts/install-sysroot.py --arch amd64' took 59.61 secs
Running hooks:  50% ( 3/ 6) sysroot_amd64
________ running 'python3 sdk/build/linux/sysroot_scripts/install-sysroot.py --arch arm' in '/home/alain/Projects/articles/travail/dart-sdk'
Installing Debian Jessie arm root image: /home/alain/Projects/articles/travail/dart-sdk/sdk/build/linux/debian_jessie_arm-sysroot
Downloading https://commondatastorage.googleapis.com/chrome-linux-sysroot/toolchain/7031a828c5dcedc937bbf375c42daab08ca6162f/debian_jessie_arm_sysroot.tgz
Hook 'python3 sdk/build/linux/sysroot_scripts/install-sysroot.py --arch arm' took 50.02 secs
Running hooks:  66% ( 4/ 6) sysroot_amd64
________ running 'python3 sdk/build/linux/sysroot_scripts/install-sysroot.py --arch arm64' in '/home/alain/Projects/articles/travail/dart-sdk'
Installing Debian Jessie arm64 root image: /home/alain/Projects/articles/travail/dart-sdk/sdk/build/linux/debian_jessie_arm64-sysroot
Downloading https://commondatastorage.googleapis.com/chrome-linux-sysroot/toolchain/7031a828c5dcedc937bbf375c42daab08ca6162f/debian_jessie_arm64_sysroot.tgz
Hook 'python3 sdk/build/linux/sysroot_scripts/install-sysroot.py --arch arm64' took 49.14 secs
Running hooks: 100% (6/6), done.
Running: git submodule foreach 'git config -f $toplevel/.git/config submodule.$name.ignore all'
Running: git config --add remote.origin.fetch '+refs/tags/*:refs/tags/*'
Running: git config diff.ignoreSubmodules all
```

Then, do some changes in the source code and build the arm SDK :

```
$ nano sdk/runtime/vm/cpu_arm.cc
$ cd sdk
$ ./tools/build.py --no-goma -m release -a arm create_sdk
Done. Made 385 targets from 93 files in 257ms
ninja -C out/ReleaseXARM create_sdk
ninja: Entering directory `out/ReleaseXARM'
[4615/4615] STAMP obj/create_sdk.stamp
The build took 739.001 seconds
$ (cd out/ReleaseXARM && zip -r dartsdk-linux-armhf dart-sdk)
```

You can now copy this new SDK to the armv7 running container :

```
$ docker ps
CONTAINER ID   IMAGE
96facecd1419   atsigncompany/buildimage:dart-armv7
$ docker cp out/ReleaseXARM/dartsdk-linux-armhf.zip 96facecd1419:/root
```

Now back in the container :

```
# unzip dartsdk-linux-armhf.zip
# export PATH=/root/dart-sdk/bin:$PATH
# dart
  ╔════════════════════════════════════════════════════════════════════════════╗
  ║ The Dart tool uses Google Analytics to report feature usage statistics     ║
  ║ and to send basic crash reports. This data is used to help improve the     ║
  ║ Dart platform and tools over time.                                         ║
  ║                                                                            ║
  ║ To disable reporting of analytics, run:                                    ║
  ║                                                                            ║
  ║   dart --disable-analytics                                                 ║
  ║                                                                            ║
  ╚════════════════════════════════════════════════════════════════════════════╝

A command-line utility for Dart development.

Usage: dart <command|dart-file> [arguments]

Global options:
-h, --help                 Print this usage information.
-v, --verbose              Show additional command output.
    --version              Print the Dart SDK version.
    --enable-analytics     Enable analytics.
    --disable-analytics    Disable analytics.

Available commands:
  analyze   Analyze Dart code in a directory.
  compile   Compile Dart to various formats.
  create    Create a new Dart project.
  fix       Apply automated fixes to Dart source code.
  format    Idiomatically format Dart source code.
  migrate   Perform null safety migration on a project.
  pub       Work with packages.
  run       Run a Dart program.
  test      Run tests for a project.

Run "dart help <command>" for more information about a command.
See https://dart.dev/tools/dart-tool for detailed documentation.
```

### Return of the certificates problem

There is a small Dart example in the 'restserver' folder :

```
# cd restserver
# dart pub get
Resolving dependencies... (37.5s)
It looks like pub.dartlang.org is having some trouble.
Pub will wait for a while before trying to connect again.
```

The problem here is the TLS communication with <https://pub.dev> cannot be
trusted.

Very dirty workaround, copy `/etc/ssl/certs` from the host into the running
container :

```
$ docker cp /etc/ssl/certs/ 96facecd1419:/etc/ssl
```

And back to the container :

```
# dart pub get
...
Downloading test 1.17.4...
Downloading http 0.13.3...
Downloading pedantic 1.11.0...
Downloading mongo_dart 0.7.0+1...
Downloading sse 4.0.0...
Downloading args 2.1.0...
Downloading shelf_router 1.0.0...
Downloading shelf 1.1.4...
...
```

### The "Deletion failed, path = 'xxx' (OS Error: Value too large for defined data type, errno = 75)" problem

Deletion failed on error, but try again :

```
# dart pub get
Resolving dependencies... (4.7s)
  _fe_analyzer_shared 21.0.0 (22.0.0 available)
  analyzer 1.5.0 (1.7.1 available)
...
Got dependencies!
```

We can even compile the Dart source :

```
# dart compile exe bin/courses_server.dart -o ./coursesd
Info: Compiling with sound null safety
Generated: /root/restserver/coursesd
Error: AOT compilation failed
FileSystemException: Deletion failed, path = '/tmp/JHJYUL' (OS Error: Value too large for defined data type, errno = 75)
```

And run the Dart executable :

```
./coursesd
Loaded data from file
Registered Courses REST API
Listen SSE clients
Server launched on http://0.0.0.0:8067
```

If we look at the `/tmp` directory, it's full of Dart temporary files that have
not been deleted...

```
# rm -rf /tmp/pub_* /tmp/JHJYUL ...
```

PLEASE HELP !!! :D
