# Swift Driver
Before exploring [swift-driver](https://github.com/apple/swift-driver), it's worth visiting the following links to get some more context 
and also be familiar with some terminology:
1. [Swift Binary Serialization Format](https://github.com/apple/swift/blob/main/docs/Serialization.md)
2. [Modules](https://github.com/apple/swift/blob/main/docs/Modules.rst)
3. [Lexico](https://github.com/apple/swift/blob/main/docs/Lexicon.md)
4. [Llbuild2](https://forums.swift.org/t/llbuild2/36896) 

## Why explore Swift-driver?
1. If you are interested to know more about what happens when you do “swift ...” or “swiftc..” Luckily [swift-driver](https://github.com/apple/swift-driver) is open-sourced and is definitely one of the best references to understand what goes behind the scenes.
2. A well-intended interop amongst swift-driver, llbuild2, swift compiler, swift package is enabling some really amazing features that I am looking forward to:
    1. [Explicit module support](https://forums.swift.org/t/explicit-module-builds-the-new-swift-driver-and-swiftpm/36990) - enabling the build system to better understand how to build the modules.
    2. Faster incremental builds, better diagnostics etc. 
    3. A lot of optimisations not only in improving build times but also enabling distributed builds on top of llbuild2 and remote artifacts caching support.
3. This also means that implicit module cache departure is near and so is the departure of the cons that it brings. [Reference](https://forums.swift.org/t/llbuild2/36896)
4. Most importantly, being all in Swift also makes understanding the internals very easy for any developer that understands Swift. 
5. It can also be integrated with other build systems. Once “planBuild()” is called on the Driver initialized with appropriate args, it returns the compilation jobs that can be added by the build system at the most appropriate place in the sequence, without actually executing the compilation jobs.
6. [The Swift Driver, Compilation Model, and Command-Line Experience](https://github.com/apple/swift/blob/main/docs/Driver.md)
7. This is a great doc that talks about the Driver design and internals:
    1. [Driver Stages](https://github.com/apple/swift/blob/main/docs/DriverInternals.md#driver-stages)
    2. [Parse: Option parsing](https://github.com/apple/swift/blob/main/docs/DriverInternals.md#parse-option-parsing)
    3. [Pipeline: Converting Args into Actions](https://github.com/apple/swift/blob/main/docs/DriverInternals.md#parse-option-parsing)
    4. [Build: Translating Actions into Jobs using a ToolChain](https://github.com/apple/swift/blob/main/docs/DriverInternals.md#build-translating-actions-into-jobs-using-a-toolchain)
    5. [Schedule: Ordering and skipping jobs by dependency analysis](https://github.com/apple/swift/blob/main/docs/DriverInternals.md#schedule-ordering-and-skipping-jobs-by-dependency-analysis)
    6. [Batch: Optionally combine similar jobs](https://github.com/apple/swift/blob/main/docs/DriverInternals.md#schedule-ordering-and-skipping-jobs-by-dependency-analysis)
    7. [Execute: Running the Jobs in a Compilation using a TaskQueue](https://github.com/apple/swift/blob/main/docs/DriverInternals.md#execute-running-the-jobs-in-a-compilation-using-a-taskqueue)
    8. [DriverParseableOutput.md](https://github.com/apple/swift/blob/main/docs/DriverParseableOutput.md)


## Notes
1. A triple is a struct constructed from a string that has the following canonical form:
ARCHITECTURE-VENDOR-OPERATING_SYSTEM or  ARCHITECTURE-VENDOR-OPERATING_SYSTEM-ENVIRONMENT
2. PredictableRandomNumberGenerator - a lot of focus towards predictability because any randomness might pollute the caching etc.
3. “-experimental-explicit-module-build” - This needs to be passed to enable the explicit module build feature:
4. Have a look at [Options.swift](https://github.com/apple/swift-driver/blob/main/Sources/SwiftOptions/Options.swift) to understand that all is accepted as args.
5. Enable these flags for local testing:
    1. “-driverPrintOutputFileMap” - Dump the contents of the output file map.
    2. "-driver-print-actions" - Dump list of actions to perform.
    3. “-driverPrintGraphviz” - Write the job graph as a graphviz file.
    4. “-driver-warn-unused-options" - Emit warnings for any provided options which are unused by the driver.
    5. “-emit-sil” - if you would like to see the canonical SIL.
    6. “-emit-ir” - "Emit LLVM IR file(s) after LLVM optimizations"
    7. “-driver-show-job-lifecycle" -> "Show every step in the life cycle of driver jobs"
    8. "-driver-show-incremental" - With -v, dump information about why files are being rebuilt.
    
## Let’s take a very simple example

Let’s say we have a few files that we want to bundle together and distribute as a dylib.

“swiftc a.swift b.swift c.swift -module-name MyModule -emit-library -target”.

As we execute the above command, the driver is invoked.
1. The driver based on the arguments and process environment variables available, sets its internal state.
    1. InvocationRunMode.
    2. DriverKind.
    3. Compiler Mode { standardCompile | bath | repl ...}
    4. Whether it should attempt incremental compilation or not.
    5. And based on the above information, it builds the right toolchain and determines target information.
    6. Multi threading state etc
    7. And a lot of validations (validate framework search paths etc.)
    8. Post this:
        1. Darwin toolchain is selected since we are working on the macOS platform. The toolchain is used to lookup the absolute path for the swift compiler, dynamic linker(“ld”), static linker(“libtool”), clang, lldb, dwarfdump etc.
        2. The compiler mode is “standard compilation” which is expected since we didn’t pass any bath mode flag, or involved the repl, or compilePCM or dumpPCM options.
2. Then we can call “driver.planBuild()”
    1. It returns the jobs based on the above query -> individual subprocess that should be involved during the compilation.
    2. Since it's a standard compilation, it returns 4 jobs. The first 3 jobs are to compile “a.swift”, “b.swift”, “c.swift” respectively and the fourth job is to link all of them to construct a dylib.
        1. Job 1 - “Compile a.swift” -> a-1.o
        2. Job 2 - “Compile b.swift” -> b-2.o
        3. Job 3 - “Compile c.swift” -> c3.o
        4. Job 4 - “Linking” -> libMyModule.dylib
3. Once you have the jobs, you can ask [SwiftDriverExecutor](https://github.com/apple/swift-driver/blob/bad19ac928dab85695461540d3174da2ff8cd1f2/Sources/SwiftDriverExecution/SwiftDriverExecutor.swift) or your external build system to execute the same.  [SwiftDriverExecutor](https://github.com/apple/swift-driver/blob/bad19ac928dab85695461540d3174da2ff8cd1f2/Sources/SwiftDriverExecution/SwiftDriverExecutor.swift) actually makes use of [llbuild](https://github.com/apple/swift-llbuild) to execute the jobs.

### Note:
1. We can also pass WMO { single or multi threaded} or even batch mode if that's what the configuration we would want to explore.
2. The output of jobs, their sequence, and even their lifecycle and count would differ based on the arguments that we pass in.
3. For example, if we pass a bath mode compilation: “-enable-match-mode” with “-driver-batch-count 2”, then 3 jobs would be constructed. The first job would be to compile all the files “a.swift”, “b.swift”; the second job will be about compiling “c.swift” and the last would be to link all of them to generate a dylib.
4. For example, if we pass a WMO mode compilation: “-whole-module-optimization”, then 2 jobs would be constructed. The first job would be to compile all the files “a.swift”, “b.swift”, “c.swift” and the last would be to link all of them.
5. Note: we can also pass “-emit-module” if we would like the Driver to generate the “.swiftmodule” file.


## Let’s take a different but a simple example that involves incremental builds
“swiftc -module-name MyModule -output-file-map <OFM> -driver-show-incremental -driver-show-job-lifecycle
-enable-batch-mode
-save-temps
-incremental
a.swift b.swift
”
Let’s assume that it's a clean build and there is no derived data present. Let’s also assume that the a.swift and b.swift don’t depend on each other.

1. When the Driver is involved for the first time, clearly there is no derived date, no Build Record present so the Driver doesn’t have an option to skip a compile level job. It generates a new Build Record as well as the dependency graph. 
2. It then produces 2 jobs. The first one is about  compiling the swift files - “a.swift”, “b.swift” that outputs “a.o”, “b.o”, “a.swiftdeps”, “b.swiftdeps” and the last one is about linking the generated object files to output MyModule.o
3. Now if you see your derived data folder, you will see the following files generated:
    1. a.o
    2. a.swiftdeps
    3. b.o
    4. b.swiftdeps
    5. MyModule-master.priors
    6. MyModule-master.swiftdeps
    7. MyModule.o
4. Now, if you don’t change any file contents and invoke the Driver or use the above swiftc command, the compilation sequence steps for “a.swift”, “b.swift” will be skipped. The Driver has reference to the old BuildRecord and the BuildRecord has the last modified time. Since the files haven’t been modified, the existing generated results are used - which is great because all those heavy steps are saved. You will see the following log statements that say the same:

```
Incremental compilation: Enabling incremental cross-module building
Incremental compilation: May skip current input:  {compile: a.o <= a.swift}
Incremental compilation: May skip current input:  {compile: b.o <= b.swift}
Incremental compilation: Skipping input:  {compile: a.o <= a.swift}
Incremental compilation: Skipping input:  {compile: b.o <= b.swift}
Incremental compilation: Skipping job: Linking MyModule; oldest output is current /var/../TemporaryDirectory.../derivedData/.../MyModule.o
``` 
5. Now, lets make some changes to “a.swift”. Clearly the file is modified. When we invoke Driver next, it has access to the BuildRecord, the dependency graph as well. It understands that because of changes in “a.swift”, “a.swift” definitely needs to be re-compiled. But since “b.swift” swiftdeps is not dependent on “a.swift” which the compiler extracts from the respective .swiftdeps files, also given the other deps criteria, it skips compiling “b.swift” and only recompiles “a.swift”.
6. Following would be good references to understand the other steps involved:
    1. [MultiJobExecutor.swift](https://github.com/apple/swift-driver/blob/main/Sources/SwiftDriverExecution/MultiJobExecutor.swift)
    2. [IncrementalCompilationState.swift](https://github.com/apple/swift-driver/blob/main/Sources/SwiftDriver/IncrementalCompilation/IncrementalCompilationState.swift)
    

## Some other amazing improvements
To mention a few:
1. Cross-Module Incremental Build Improvements:
    1. [Antisymmetry](https://github.com/apple/swift-driver/blob/bad19ac928dab85695461540d3174da2ff8cd1f2/Tests/IncrementalImportTests/Antisymmetry.swift)
    2. [Transitivity](https://github.com/apple/swift-driver/blob/bad19ac928dab85695461540d3174da2ff8cd1f2/Tests/IncrementalImportTests/Transitivity.swift)
2. "[Explicit Module Builds](https://forums.swift.org/t/explicit-module-builds-the-new-swift-driver-and-swiftpm/36990) are allowing us to diagnose a class of problems at build plan time that would otherwise be caught much later in the compilation process or even cause compiler hangs and crashes."


## Misc
If you are interested in knowing why specific source files were compiled in the incremental build phase or why your incremental builds seem to be slow, this [post](https://forums.swift.org/t/optimizing-and-debugging-incremental-build-time-in-swift-5-5/49379) has you covered - the compiler emits “remarks” explaining why it decided to compile specific files.

In "Other Swift Flags", add -driver-show-incremental and -driver-show-job-lifecycle. 
