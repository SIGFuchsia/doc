### Run Component

- **How to create a component?**

  Just follow the official tutorial is okay.

- **How to run an example component?**

  - On `Term 1`, use below commands.

    ```bash
    fx set workstation.x64 --with //examples/hello_world
    fx build
    fx list-packages					# to check if hello-world is included
    fx vdl start -N --software-gpu		# start the FEMU with network and software GPU
    ```

    Then, `Term 1` will output the following. The last line tells us how to communicate to this device (emulator) using `fx` and `ffx` tools provided by Google.

    ```bash
    [fvdl] Using fuchsia build dir: "/home/fuchsia/fuchsia/fuchsia/out/default"
    [fvdl] Found Fuchsia root directory "/home/fuchsia/fuchsia/fuchsia"
    2021/09/05 12:51:13 [info] FVD Properties:
    device_spec:  {
    horizontal_resolution:  1280
    vertical_resolution:  800
    vm_heap:  192
    ram:  8192
    cache:  32
    screen_density:  240
    }
    fvm_size:  "2G"
    gpu:  "swiftshader_indirect"
    2021/09/05 12:51:13 [info] images: using fvm tool from /home/fuchsia/fuchsia/fuchsia/out/default/host_x64/fvm
    2021/09/05 12:51:13 [info] images: using zbi tool from /home/fuchsia/fuchsia/fuchsia/out/default/host_x64/zbi
    2021/09/05 12:51:14 [info] Resizing /tmp/launcher1019998335/femu_fvm file to 2G
    2021/09/05 12:51:14 [info] found fuchsia board architecture: x64
    2021/09/05 12:51:14 [info] running on host architecture: amd64
    2021/09/05 12:51:14 [info] device: DISPLAY=:0
    2021/09/05 12:51:14 /home/fuchsia/fuchsia/fuchsia/prebuilt/third_party/aemu/linux-x64/emulator -feature VirtioInput,GLDirectMem,Vulkan,RefCountPipe,KVM -metrics-collection -no-hidpi-scaling -grpc 55477 -rtcfps 30 -gpu swiftshader_indirect -window-size 1280x800 -no-location-ui -fuchsia -kernel /tmp/vdl_staging_hxnUxq/femu_kernel -initrd /tmp/launcher1019998335/femu_zircona-ed25519 -m 8192M -vga none -device virtio-keyboard-pci -smp 4,threads=2 -drive file=/tmp/launcher1019998335/femu_fvm,format=raw,if=none,id=vdisk -device virtio-blk-pci,drive=vdisk -serial stdio -machine q35 -device isa-debug-exit,iobase=0xf4,iosize=0x04 -enable-kvm -cpu host,migratable=no,+invtsc -device virtio_input_multi_touch_pci_1 -soundhw hda -netdev type=tap,ifname=qemu,id=net0,script=no,downscript=no -device virtio-net-pci,vectors=8,netdev=net0,mac=52:54:00:63:5e:7a -append "verbose kernel.serial=legacy TERM=xterm-256color kernel.entropy-mixin=42ac2452e99c1c979ebfca03bce0cbb14126e4021a6199ccfeca217999c0aaa0 kernel.halt-on-panic=true zircon.nodename=fuchsia-5254-0063-5e7a kernel.lockup-detector.critical-section-fatal-threshold-ms=0 kernel.lockup-detector.critical-section-threshold-ms=5000 kernel.lockup-detector.heartbeat-age-fatal-threshold-ms=0"
    2021/09/05 12:51:14 Starting emulator
    2021/09/05 12:51:16 [info] Waiting for emulator to start...
    2021/09/05 12:51:19 [info] Waiting for emulator to start...
    2021/09/05 12:51:22 [info] Waiting for emulator to start...
    2021/09/05 12:51:25 [info] Waiting for emulator to start...
    2021/09/05 12:51:28 [info] Waiting for emulator to start...
    2021/09/05 12:51:31 [info] Waiting for emulator to start...
    2021/09/05 12:51:32 [info] Found device address: fe80::53d0:df2:3acd:baf6%brqemu
    2021/09/05 12:51:32 Fuchsia Image version: 2021-09-03T03:58:43+00:00
    2021/09/05 12:51:32 Finished starting up and the device can be reached via ssh.
    2021/09/05 12:51:32  From Start | Duration | Event Name
    2021/09/05 12:51:32      19.28  |   19.28  | Fuchsiastart
    To support fx tools on emulator, please run "fx set-device fuchsia-5254-0063-5e7a"
    $ 
    ```

  - On `Term 2`.

    ```bash
    fx set-device fuchsia-5254-0063-5e7a
    fx serve-updates
    ```

    Then, the output.

    ```bash
    2021-09-05 12:55:10 [serve-updates] Discovery...
    2021-09-05 12:55:11 [serve-updates] Device up
    2021-09-05 12:55:11 [serve-updates] Registering devhost as update source
    2021-09-05 12:55:11 [serve-updates] Ready to push packages!
    2021-09-05 12:55:12 [serve-updates] Target uptime: 236
    2021-09-05 12:55:37 [pm auto] adding client: [fe80::53d0:df2:3acd:baf6%brqemu]:46623
    2021-09-05 12:55:37 [pm auto] client count: 1
    ```

  - On `Term 3`, use `ffx` tool to further control and manage all devices, including NUC and FEMU.

    This command lists the components.

    ```bash
    fx ffx --target fuchsia-5254-0063-5e7a component list
    ```

    This runs the example component using URL.

    ```bash
    ffx --target fuchsia-5254-0063-5e7a component run fuchsia-pkg://fuchsia.com/hello-world#meta/hello-world-cpp.cm
    ```

    And if we want to see the log using `fx`, we can run `fx log --only hello-world` on `Term 4`.

    ```bash
    [00262.641762][25256][25258][pkg-resolver] INFO: updated local TUF metadata for "fuchsia-pkg://devhost" to version RepoVersions { root: 7, timestamp: Some(1630674370), snapshot: Some(1630674370), targets: Some(1630674370) } while getting merkle for TargetPath("hello-world/0")
    [00262.665920][25256][25258][pkg-resolver] INFO: Fetching blobs for fuchsia-pkg://devhost/hello-world: [
        3d6bb0e691e7c2591c37674c79e33eab6e5625f5756043463497b3e24f72c089,
        4090c9eaf6476287a7904efc56443406cef53399bf205ac3155b4672359299f7,
        4d36db65b5b9e8c4bf0d0af5a1b45a94da8f1ab004779b5ff1d9eee804dded38,
        4e99f044c370ce1fbd42fb20725a5ae53c5cd8aea56309f5f1a35eb67a2c93ff,
        5a758135fc9ba99df2b2bb539d4bbab38a43a69249d31e61730ece2ed7a189b7,
        990d5a92889176cfd4564564e830201e37e4ab738d0286a3183b1e65fde871a0,
    ]
    [00262.717444][25256][25258][pkg-resolver] INFO: resolved fuchsia-pkg://fuchsia.com/hello-world as fuchsia-pkg://devhost/hello-world to b09363cb388e98b85459ca091338e00ac66f523d9e2efb8754e0ac3f6ba8aac6 with TUF
    [00262.729115][1186][1298][ffx-laboratory:hello-world-cpp] INFO: Hello, World!
    ```

- **How to add a third party package to `/bin` or `/boot/bin`?**

  To build our third party packages and add one or more packages to the shell, which is convenient to run by bypassing the `fx` server and `ffx` device manager, we can follow the steps. Take the `//third-party/benchmark`, a Google's benchmark framework, as an example.

  - Modify the `GN` build system.

    Modify `//third-party/benchmark/BUILD.gn`.

    ```diff
    diff --git a/BUILD.gn b/BUILD.gn
    index 4cb9f78..028fae1 100644
    --- a/BUILD.gn
    +++ b/BUILD.gn
    @@ -2,14 +2,17 @@
     # Use of this source code is governed by a BSD-style license that can be
     # found in the LICENSE file.
     
    +import("//build/components.gni")
    +
     config("benchmark_config") {
       visibility = [ ":*" ]
       include_dirs = [ "include" ]
     }
     
    -static_library("benchmark") {
    -  testonly = true
    +executable("benchmark_test_bin") {
    +  output_name = "benchmark_test"
       sources = [
    +    "test/benchmark_test.cc",
         "src/benchmark.cc",
         "src/benchmark_register.cc",
         "src/colorprint.cc",
    @@ -31,3 +34,7 @@ static_library("benchmark") {
       ]
       configs += [ "//build/config:Wno-conversion" ]
     }
    +
    +fuchsia_shell_package("benchmark") {
    +  deps = [ ":benchmark_test_bin" ]
    +}
    ```

    Modify `//build/config/BUILD.gn`.

    ```diff
    diff --git a/build/config/BUILD.gn b/build/config/BUILD.gn
    index ff00c465695..e432a8cefcf 100644
    --- a/build/config/BUILD.gn
    +++ b/build/config/BUILD.gn
    @@ -983,6 +983,7 @@ config("Wno-conversion") {
         "//third_party/opus/*",
         "//third_party/rust_crates/compat/brotli:*",
         "//third_party/zstd:*",
    +    "//third_party/benchmark:*",
         "//tools/bootserver_old:*",
         "//tools/fidlcat:*",
         "//tools/fidlcat/interception_tests:*",
    ```

    Modify `garnet/packages/prod/BUILD.gn`. Here, I have no idea what should be in the `garnet` code area and what third party code should be configured in `garnet`. I see the `sbase` package is in this file so I just follow.

    ```diff
    diff --git a/garnet/packages/prod/BUILD.gn b/garnet/packages/prod/BUILD.gn
    index e96345106df..19f7abf9877 100644
    --- a/garnet/packages/prod/BUILD.gn
    +++ b/garnet/packages/prod/BUILD.gn
    @@ -39,6 +39,7 @@ group("all") {
         "//garnet/packages/prod:network-speed-test",
         "//garnet/packages/prod:run",
         "//garnet/packages/prod:sbase",
    +    "//garnet/packages/prod:benchmark",
         "//garnet/packages/prod:scenic",
         "//garnet/packages/prod:sched",
         "//garnet/packages/prod:setui_client",
    @@ -164,6 +165,7 @@ group("cmdutils") {
         "//garnet/bin/uname",
         "//garnet/packages/prod:hwstress",
         "//garnet/packages/prod:sbase",
    +    "//garnet/packages/prod:benchmark",
         "//garnet/packages/prod:sched",
       ]
     }
    @@ -258,6 +260,11 @@ group("run") {
       public_deps = [ "//garnet/bin/run" ]
     }
     
    +group("benchmark") {
    +  public_deps = [ "//third_party/benchmark:benchmark" ]
    +  deps = [ "//build/validate:non_production_tag" ]
    +}
    +
     group("sbase") {
       public_deps = [
         "//third_party/sbase:basename",
    ```

  - Fix the bug in the example of benchmark provided by Google.

    ```diff
    diff --git a/test/benchmark_test.cc b/test/benchmark_test.cc
    index d832f81..5ab2191 100644
    --- a/test/benchmark_test.cc
    +++ b/test/benchmark_test.cc
    @@ -57,7 +57,7 @@ static void BM_Factorial(benchmark::State& state) {
       // Prevent compiler optimizations
       std::stringstream ss;
       ss << fac_42;
    -  state.SetLabel(ss.str());
    +  state.SetLabel(ss.str().c_str());
     }
     BENCHMARK(BM_Factorial);
     BENCHMARK(BM_Factorial)->UseRealTime();
    @@ -67,7 +67,7 @@ static void BM_CalculatePiRange(benchmark::State& state) {
       while (state.KeepRunning()) pi = CalculatePi(state.range(0));
       std::stringstream ss;
       ss << pi;
    -  state.SetLabel(ss.str());
    +  state.SetLabel(ss.str().c_str());
     }
     BENCHMARK_RANGE(BM_CalculatePiRange, 1, 1024 * 1024);
    ```

  - Run `benchmark_test`, the name of the executable, in emulator shell.

    ```shell
    $ benchmark_test
    Run on (1 X 2592.14 MHz CPU )
    2021-09-06 11:41:17
    ***WARNING*** Library was built as DEBUG. Timings may be affected.
    Benchmark                                         Time           CPU Iterations
    -------------------------------------------------------------------------------
    BM_Factorial                                     20 ns         20 ns   33472229 40320
    BM_Factorial/real_time                           20 ns         20 ns   35920865 40320
    BM_CalculatePiRange/1                            13 ns         13 ns   54041014 0
    BM_CalculatePiRange/8                            41 ns         41 ns   17306448 3.28374
    BM_CalculatePiRange/64                          259 ns        259 ns    2660336 3.15746
    BM_CalculatePiRange/512                        2016 ns       2016 ns     353513 3.14355
    BM_CalculatePiRange/4k                        15967 ns      15967 ns      44053 3.14184
    BM_CalculatePiRange/32k                      126546 ns     126546 ns       5487 3.14162
    BM_CalculatePiRange/256k                    1012777 ns    1012781 ns        689 3.1416
    BM_CalculatePiRange/1024k                   4095950 ns    4095971 ns        139 3.14159
    BM_CalculatePi/threads:8                        789 ns       3954 ns     176368
    BM_CalculatePi/threads:1                       3975 ns       3975 ns     173389
    BM_CalculatePi/threads:2                       2001 ns       4003 ns     175916
    BM_CalculatePi/threads:4                        995 ns       3906 ns     177236
    BM_CalculatePi/threads:8                        753 ns       3841 ns     174008
    BM_CalculatePi/threads:16                       657 ns       4052 ns     166176
    BM_CalculatePi/threads:32                       256 ns       3899 ns     160064
    BM_CalculatePi/threads:1                       3941 ns       3941 ns     178150
    BM_SetInsert/1024/1                           62785 ns      62206 ns      11298   62.7956kB/s   15.6989k items/s
    BM_SetInsert/4k/1                            252802 ns     251994 ns       2774   15.5013kB/s   3.87534k items/s
    BM_SetInsert/8k/1                            529103 ns     526835 ns       1285   7.41456kB/s   1.85364k items/s
    BM_SetInsert/1024/8                           68558 ns      68193 ns      10335   458.256kB/s   114.564k items/s
    BM_SetInsert/4k/8                            268487 ns     266804 ns       2635   117.127kB/s   29.2818k items/s
    BM_SetInsert/8k/8                            535696 ns     533019 ns       1323   58.6283kB/s   14.6571k items/s
    BM_SetInsert/1024/10                          70453 ns      69789 ns       9930   559.722kB/s   139.931k items/s
    BM_SetInsert/4k/10                           271270 ns     271624 ns       2553   143.811kB/s   35.9528k items/s
    BM_SetInsert/8k/10                           535577 ns     533078 ns       1292   73.2773kB/s   18.3193k items/s
    BM_Sequential<std::vector<int>,int>/1            62 ns         62 ns   11171256   61.4717MB/s   15.3679M items/s
    BM_Sequential<std::vector<int>,int>/8          1510 ns       1510 ns     459005   20.2076MB/s   5.05191M items/s
    BM_Sequential<std::vector<int>,int>/64         4796 ns       4795 ns     146760   50.9115MB/s   12.7279M items/s
    BM_Sequential<std::vector<int>,int>/512       24337 ns      24337 ns      28851   80.2538MB/s   20.0635M items/s
    BM_Sequential<std::vector<int>,int>/1024      45693 ns      45693 ns      15478   85.4887MB/s   21.3722M items/s
    BM_Sequential<std::list<int>>/1                  45 ns         45 ns   15432027   85.3898MB/s   21.3474M items/s
    BM_Sequential<std::list<int>>/8                1323 ns       1323 ns     495244   23.0739MB/s   5.76847M items/s
    BM_Sequential<std::list<int>>/64              11765 ns      11765 ns      60333   20.7515MB/s   5.18787M items/s
    BM_Sequential<std::list<int>>/512             94780 ns      94781 ns       7280   20.6068MB/s    5.1517M items/s
    BM_Sequential<std::list<int>>/1024           191727 ns     191728 ns       3681   20.3739MB/s   5.09349M items/s
    BM_Sequential<std::vector<int>, int>/512      24178 ns      24178 ns      29185   80.7815MB/s   20.1954M items/s
    BM_StringCompare/1                               79 ns         79 ns    8741679
    BM_StringCompare/8                               83 ns         83 ns    7820085
    BM_StringCompare/64                             107 ns        107 ns    6417652
    BM_StringCompare/512                            323 ns        323 ns    2179937
    ```

  - Run our own component in `/boot/bin`.

    Copy the `//example/hello_world/cpp` to `//src/bringup/bin/hello_world`. Add the `//src/bringup/bin/hello-world:bin` to the dependencies of the `zedboot` group to `//src/bringup/bundles/BUILD.gn`. Then, we can even run the executable even with the `bringup` configuration.

  - Write some micro-benches using the framework.

    <span style='color:red'>TODO</span>

  - Run the same tests on both Linux and Zircon on Intel NUC.

    <span style='color:red'>TODO</span>

