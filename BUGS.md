  Looking at the recent 5 commits (CMakeLists.txt:255-270 through nightly-macos.yml:1-131), this
  series introduces Darwin (macOS) as a host platform for building and testing ld.eld.
  From the perspective of a lead senior compiler and linker engineer, several architectural shortcuts,
  symbol visibility leaks, test assertion regressions, and CI resource risks were introduced. Below is
  a structured architectural and in-depth review.
  ──────
  ### 1. Symbol Visibility & Dual-Linkage ODR Hazard in libLW.dylib
  Commits: CMakeLists.txt:16-25, AddNoVersionedSymbolExports.cmake:1-10

  #### What Happened
  On Linux/ELF, AddNoVersionedSymbolExports.cmake:10-29 synthesizes a GNU version script from
  LW.exports:1-46 with local: *;. This gives hidden visibility to all static LLVM libraries linked
  into libLW.so.
  On Darwin (Apple ld64), --version-script is unsupported. Commit
  AddNoVersionedSymbolExports.cmake:3-7 handles this with an early return:
    if(NOT LLVM_HAVE_LINK_VERSION_SCRIPT)
      return()
    endif()

  Because no symbol export control was provided for Apple ld64, all static LLVM symbols linked into
  libLW.dylib are exported as public dynamic symbols in Mach-O's dynamic symbol table.

  When unit tests linked against both libLW and LLVM static archives, llvm::cl::opt static
  registrations fired in both images, causing a runtime crash on startup: "Option 'basic' already
  exists".
  #### The Questionable Workaround
  Instead of restricting exports from libLW.dylib, commit CMakeLists.txt:16-25 hacked around the crash
  by stripping LLVM libraries from the unit test link lines on Apple:

    if(APPLE)
      set(LLVM_LINK_COMPONENTS Support)
    endif()
  and in CMakeLists.txt:3-6, CMakeLists.txt:3-6, and CMakeLists.txt:11-14:
    set(_lto_lib)
    if(NOT APPLE)
      set(_lto_lib LLVMLTO)
    endif()

  #### Architectural Risks
  1. Accidental Pseudo-LLVM dylib: libLW.dylib now re-exports arbitrary LLVM internals with Mach-O
  default visibility. Any plugin or external consumer loading libLW has its namespace polluted with
  duplicate LLVM symbols, inviting subtle ODR violations and crash bugs.
  2. Hidden Test Breakages: Any future unit test added under CMakeLists.txt that needs non-Support
  LLVM components (e.g. MC, Object, TargetParser) without linking LW will fail to link on Darwin.
  3. Proper Solution: Darwin's ld64 supports symbol export filtering via -Wl,-exported_symbols_list,
  <file> or -Wl,-unexported_symbols_list,<file>. LW.exports can be translated into a Mach-O export
  list (or compiled with -fvisibility=hidden on libLW), keeping LLVM internals strictly hidden.
  ──────
  ### 2. Test Assertion Weakening in DynamicGOTPLT

  Commit: DynamicGOTPLT_AArch64.test:14-40

  #### What Happened
  In all five target tests (DynamicGOTPLT_AArch64.test, DynamicGOTPLT_ARM.test,
  DynamicGOTPLT_Hexagon.test, DynamicGOTPLT_RISCV32.test, and DynamicGOTPLT_RISCV64.test), the test
  previously verified that the linker resolved PLTGOT to the actual address of .plt:

    -CHECK-DAG: [{{[ 0-9]+}}] .plt PROGBITS [[PLT:[[:xdigit:]]+]] {{[[:xdigit:]]+}}
    ...
    -CHECK: {{0*}}0 {{0*}}0 {{0*}}0 [[PLT]]
  Because BSD od on macOS does not support -w64, the test was rewritten to use llvm-readelf -x. But
  because readelf -x dumps formatted little-endian byte words, matching [[PLT]] failed. The assertion
  was changed to:

    +CHECK: Hex dump of section '.got.plt':
    +CHECK-NEXT: 0x{{0*}}70000 00000000 00000000 00000000 00000000
    +CHECK-NEXT: 0x{{0*}}70010 00000000 00000000 {{[[:xdigit:]]+}} 00000000
  #### Architectural Concern

  • Neutered Assertions: Replacing [[PLT]] with a generic wildcard regex {{[[:xdigit:]]+}} means the
  test passes even if the linker writes invalid garbage into the GOT slot, as long as it is non-zero.
  The entire verification objective of the test was dropped to make the runner green.
  • Fix: Use llvm-readobj --relocs / llvm-objdump -d or verify the exact byte layout of the PLT
  address rather than wildcarding it.
  ──────
  ### 3. Mach-O Runtime Semantics & Path Handling

  Commit: SearchDirs.cpp:245-315
  #### A. @loader_path vs @executable_path

  In SearchDirs.cpp:248-260:

    eld::string::ReplaceString(CRPath, "$ORIGIN", SearchDirs::MainExecutablePath);
    eld::string::ReplaceString(CRPath, "@loader_path", SearchDirs::MainExecutablePath);
    eld::string::ReplaceString(CRPath, "@executable_path", SearchDirs::MainExecutablePath);

  In Mach-O dyld semantics:

  • @executable_path resolves to the main process binary (ld.eld).
  • @loader_path resolves to the Mach-O binary containing the load command.

  If an external caller, driver, or plugin residing in another directory references @loader_path,
  substituting it with MainExecutablePath is incorrect.

  #### B. Empty String Path Token Leading to Root / Traversal
  In SearchDirs.cpp:292-314:

    #ifdef __APPLE__
      Path = llvm::sys::Process::GetEnv("DYLD_LIBRARY_PATH");
      if (Path && !Path->empty())
        Combined = *Path;
    #endif
      Path = llvm::sys::Process::GetEnv("LD_LIBRARY_PATH");
      if (Path && !Path->empty()) {
        if (!Combined.empty())
          Combined += ":";
        Combined += *Path;
      }
    ...
      PathSplit = eld::string::split(Combined, ':');
      for (auto R : PathSplit) {
        const std::string FileName = R + "/" + LibraryName.str();
  If DYLD_LIBRARY_PATH or LD_LIBRARY_PATH has a trailing or leading colon (e.g. foo: or :foo), or two
  colons :::
  1. StringUtils.cpp:25-35 yields an empty string "".
  2. FileName = R + "/" + LibraryName.str() resolves to "/libfoo.dylib" (the root directory!).
  In standard POSIX and dyld semantics, an empty path element denotes the current directory (.), never
  /.
  #### C. Header-Emitted Static Variable in PluginBase.h.inc
  In PluginBase.h.inc:38-46:

    #elif defined(__APPLE__)
    __attribute__((
        used,
        section("__DATA,__eld_plugin"))) static const char eld_plugin_built_with[] =
        "Plugin built with tools @LLVM_VENDOR_VERSION@";
  1. Because this is in a common header included by dozens of files and marked static +
  __attribute__((used)), every translation unit gets a duplicate copy in Mach-O's __DATA segment that
  the linker cannot dead-strip.
  2. In ELF, .comment uses "MS",@progbits,1 which instructs the linker to merge strings. Mach-O has no
  direct equivalent in __DATA.
  3. Read-only metadata in Mach-O should be placed in __TEXT,__const or __DATA_CONST,__const, not
  writable __DATA.
  ──────
  ### 4. CMake Anti-Patterns & Commit Slicing
  Commits: CMakeLists.txt:255-270, CMakeLists.txt:198-220

  #### A. Global link_directories for Homebrew

  In CMakeLists.txt:260-268:

    if(APPLE)
      foreach(_eld_brew_lib IN ITEMS /opt/homebrew/lib /usr/local/lib)
        if(EXISTS "${_eld_brew_lib}")
          link_directories("${_eld_brew_lib}")
        endif()
      endforeach()
    endif()

  • Modern CMake Violation: link_directories injects -L flags globally across all targets.
  • Architecture Clash: /opt/homebrew/lib is for ARM64, /usr/local/lib is for x86_64. On dual-arch or
  Rosetta-enabled machines, adding both unconditionally can pull in mismatched architecture binaries.
  Use find_library or imported targets for zstd.
  #### B. CMAKE_BUILD_WITH_INSTALL_RPATH TRUE Breaks Standalone Unit Tests
  In CMakeLists.txt:205-210:

    if(APPLE)
      set(CMAKE_MACOSX_RPATH ON)
      set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
      set(CMAKE_INSTALL_RPATH "@loader_path/../lib")

  By forcing CMAKE_BUILD_WITH_INSTALL_RPATH TRUE, CMake does not generate build-tree RPATHs.
  While bin/ld.eld can find ../lib, unit tests created by add_unittest live deep inside subdirectories
  (e.g. tools/eld/test/UnitTests/LTOPreserveListTests/).
  As a result, running any unit test binary directly outside of lit crashes immediately with dyld:
  Library not loaded: @rpath/libLW.dylib. That is why lit.cfg:50-56 had to inject DYLD_LIBRARY_PATH.

  #### C. Dead Configuration Variable ELD_ON_APPLE

  In Config.h.cmake:22:

    #cmakedefine ELD_ON_APPLE "${ELD_ON_APPLE}"
  CMakeLists.txt:175-195 was defined in CMake, but is never referenced in any C++ source file; all
  code checks #ifdef __APPLE__.

  #### D. Commit Slicing Leak in CMakeLists.txt:255-270

  Commit 87ad232c is titled:

  │ [CMake] Use keyword form of target_link_libraries after llvm_add_library

  Yet it secretly contains:

  1. Homebrew link_directories
  2. Disabling version scripts for Apple in CMakeLists.txt:102-105 (which was then redundantly handled
  again in f153f7d1)
  3. Unit test LLVM_LINK_COMPONENTS changes for Apple
  4. Unit test LLVMLTO exclusions for Apple

  These should have been grouped cleanly into commit CMakeLists.txt:170-225.
  ──────
  ### 5. GitHub Actions Nightly CI Bottlenecks

  Commit: nightly-macos.yml:1-131

  #### A. Full LLVM Monolithic Build on macos-latest

  In nightly-macos.yml:96-105:

    -DLLVM_PARALLEL_LINK_JOBS=1
    ...
    run: ninja

  • Running a naked ninja compiles all of LLVM + Clang + ELD (Release with Assertions) across 5
  backends.
  • On a standard GitHub-hosted macOS runner (3-4 vCPUs) with serialized linking
  (LLVM_PARALLEL_LINK_JOBS=1), this will take 2.5 to 3+ hours and easily risk hitting the 180-minute
  timeout or running out of disk space.
  • Target ninja check-eld already brings in all prerequisites defined in CMakeLists.txt:59-74. If
  tools like clang or opt are needed by tests, they should be added to TEST_DEPENDS rather than
  compiling the entire LLVM repository.

  #### B. Cache Thrashing & Python 3.14

  • ccache blowup: GitHub Actions enforces a strict 10 GB cache limit per repository. Saving a new 5
  GB ccache snapshot nightly via key: macos-eld-ccache-${{ runner.arch }}-${{ github.run_id }} will
  quickly evict PR CI caches.
  • python-version: '3.14': Python 3.14 is currently pre-release/alpha. Pinning nightly CI to it
  invites runner setup failures when pre-built binaries are absent.
  ──────
  ### Summary Checklist for Cleanup

   Priority | Issue                          | Location              | Recommended Remedy
  ----------|--------------------------------|-----------------------|--------------------------------
   High     | libLW.dylib re-exports all     | CMakeLists.txt:16,    | Add -Wl,-exported_symbols_list
            | LLVM symbols; unit tests       | AddNoVersionedSymbolExports.cmake:3 | or hide visibility
on libLW,
            | stripped of LLVM libs          |                       | then restore standard
            |                                |                       | LLVM_LINK_COMPONENTS to unit
            |                                |                       | tests.
   High     | DynamicGOTPLT test assertions  | DynamicGOTPLT_AArch64.test:23-40 | Restore verification
that the
            | gutted with {{[[:xdigit:]]+}}  |                       | GOT entry actually points to
            |                                |                       | .plt address using endian-
            |                                |                       | aware byte checks or llvm-
            |                                |                       | objdump.
   Medium   | Global link_directories for    | CMakeLists.txt:260    | Use find_package(zstd) or
            | Homebrew                       |                       | target-specific linking
            |                                |                       | instead of global directory
            |                                |                       | search.
   Medium   | CMAKE_BUILD_WITH_INSTALL_RPATH | CMakeLists.txt:205    | Allow CMake to maintain build-
            | TRUE breaks standalone unit    |                       | tree RPATHs so unit tests run
            | tests                          |                       | out-of-tree without
            |                                |                       | DYLD_LIBRARY_PATH.
   Medium   | Empty path tokens resolve to   | SearchDirs.cpp:308    | Handle empty token R.empty()
            | root /                         |                       | by mapping to . rather than
            |                                |                       | concatenating with /.
   Medium   | ninja builds entire LLVM       | nightly-macos.yml:105 | Target ninja check-eld (or
            | monorepo on macOS runner       |                       | ninja ld.eld clang check-eld)
            |                                |                       | to stay within runner time &
            |                                |                       | disk limits.
   Low      | Static eld_plugin_built_with   | PluginBase.h.inc:39   | Place in read-only section or
            | replicated in __DATA           |                       | avoid header emission of
            |                                |                       | static used buffers.
   Low      | Unused ELD_ON_APPLE define     | Config.h.cmake:22     | Remove or use consistently
            |                                |                       | instead of bare __APPLE__.
