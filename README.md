# ROCm Profiler November 2016 Release

## Overview
This release of the ROCm Profiler is compatible with the ROCm 1.3
release that appears in the following GitHub organization:

https://github.com/RadeonOpenCompute

It is also backward-compatible with the ROCm 1.2 release.

This build of the profiler is supported on Carrizo, Fiji, Hawaii and Polaris (Baffin and Ellesmere).
Kaveri support has been deprecated, and while the profiler may still work, there may be some
issues when using it on Kaveri.

The ROCm Profiler is integrated into the CodeXL GPU Profiler Backend (aka
"CodeXLGpuProfiler"). CodeXL is the official delivery vehicle for the ROCm
Profiler, but it is being made available here as well for those who want to
use the profiler independently of CodeXL.

The ROCm Profiler is also installed by default when you follow the ROCm installation
instructions provided at https://radeonopencompute.github.io/install.html

Information contained here is specific to CodeXLGpuProfiler version 4.0.6036. This build
is newer than the version of the profiler included in [CodeXL 2.2](https://github.com/GPUOpen-Tools/CodeXL).
It is also slightly newer than the version of the profiler included in ROCm 1.3.

## Table of Contents
* [What's New](#whats-new)
* [Previous Release Notes](#previous-release-notes)
* [Collecting an Application Trace](#collecting-an-application-trace)
* [Collecting GPU Performance Counters](#collecting-gpu-performance-counters)
* [System Setup](#system-setup)
* [Sample Usage](#sample-usage)
* [Using with CodeXL 2.2](#using-this-build-with-codexl22)
* [Building the ROCm Profiler](#building-the-rocm-profiler)
* [Related Links](#related-links)
* [Known Issues](#known-issues)
* [License](LICENSE)

## What's New
* Support for collecting timing data for data transfers initiated by a call to hsa_amd_memory_async_copy
* Support for collecting an AQL Packet trace (using --hsaaqlpackettrace command-line switch). This requires the use of ROCm 1.3
* Support for reporting how many passes are required for a given set of performance counters (using --numberofpass switch)
* Support for generating default counter files (using --list --outputfile switches)
* Support for generating single-pass counter files (using --maxpassperfile in conjunction with --list and --outputfile)
* Support for whole-application replay, where the profiler will execute the application more than once with a different set of perform counters with each execution. This can make it easier to collect a full set of performance counters for a given application (works best when the application runs deterministically from one run to the next). This mode is initiated by passing in more than one --counterfile switch. After the application is executed the specified number of times, the profiler will combine the performance counter results into a single file.
* Support for better control over which parts of an application are profiled (using --startdelaty and --profileduration switches)
* When installing the rocm-profiler package (.deb or .rpm), a set of single-pass counter files will be automatically generated into /opt/rocm/profiler/counterfiles. These counter files can be used for whole-application replay (see [Known Issues](#KnownIssues) for a known problem with this)

## Previous Release Notes
* August 2016 Release
 * Support for ROCm 1.2
 * Support for Hawaii GPUs
 * Support for tracing AMD extension APIs
 * Support for CodeXL 2.2
 * The ROCm-Profiler is now installed by default with all of ROCm
 * Many bug fixes to improve stability and functionality
* April 2016 Release
 * The profiler executable has been renamed from "sprofile" to "CodeXLGpuProfiler"
 * Support for ROCm 1.0
 * Support for CodeXL 2.0
 * CodeXL is now open-sourced. As the ROCm Profiler is a component of CodeXL, the source code of the profiler is part of the [CodeXL repository](https://github.com/GPUOpen-Tools/CodeXL)
 * Many bug fixes to improve stability and functionality

## Collecting an Application Trace

To collect an application trace with kernel timestamps:

   `./CodeXLGpuProfiler --hsatrace AppToProfile`

Executing CodeXLGpuProfiler with the `--hsatrace` switch will launch the specified
"AppToProfile" and allow the profiler to trace all HSA APIs called by the
application. In addition, the profiler will also gather 2 sets of timestamp
information:
* Host-side CPU timestamps for all HSA APIs called by the application
* Device-side GPU timestamps for all HSA kernels dispatched by the application as well as all data transfers initiated by a call to hsa_amd_memory_async_copy

If you want to include a full trace of all AQL packets submitted to a queue by the application, you can use the --hsaaqlpackettrace switch instead.

The results from the profiler will be written to an .atp file that is very
similar to the .atp file written by CodeXL's OpenCL profiler. When profiling
a ROCm application, the .atp file will have four main sections, the first three
of which are similar to the sections written by the OpenCL profiler. The CodeXL
documentation contains detailed information on the contents of the .atp file
for an OpenCL application.

Here is a brief description of the .atp file sections for the ROCm profiler:
* The first section is a file header containing information about the application profiled, the system the profile was gathered on, and some of   the options used to gather the profile.
* The second section is the API trace, showing a per-thread list of all HSA APIs called by the application. For each API, the parameters passed and the values returned by the API are shown.
* The third section is the host-side timestamps section, showing a per-thread list of all HSA APIs called by the application. For each API, a start time and end time is shown. For each hsa_amd_memory_async_copy call, two additional data transfer timestamps are collected.
* The fourth section is new for HSA. It is the device-side kernel timestamp section, showing a list of each kernel dispatched by the application. For each kernel, the following information is shown:
 * the name of the kernel symbol associated with the kernel dispatched (or "<UnknownKernelName>" if the profiler could not extract the symbol name.)
 * the kernel pointer value of the kernel dispatched.
 * the start timestamp of the kernel, indicating when the kernel started executing on the device. The timestamps represent nanoseconds.
 * the end timestamp of the kernel, indicating when the kernel finished executing on the device. The timestamps represent nanoseconds.
 * the duration of the kernel (in nanoseconds).
 * the agent handle of the agent the kernel was dispatched to.
 * If collecting an AQL packet trace, the following additional information is also shown:
  * the packet type
  * the packet index
  * the AQL packet structure

Most other CodeXLGpuProfiler switches will also work with the `--hsatrace` or `--hsaaqlpackettrace` switches:

* You can control the name and location of the .atp file using CodeXLGpuProfiler's `--outputfile` switch.
* You can generate a subset of the Trace Summary pages using CodeXLGpuProfiler's `--tracesummary` and `--atpfile` switch. For instance: `./CodeXLGpuProfiler --tracesummary --atpfile myapp.atp` will generate an API summary, a kernel summary and a top-ten kernel dispatch list from the data contained in the myapp.atp file. It will also produce a "Best Practices" summary file by applying a rules-based analysis of the .atp file. Currently two rules are supported:
 * An error will be reported if any HSA API returns an error code
 * A resource leak will be reported for mismatched create/destroy calls (i.e. if the application calls hsa_queue_create without a corresponding hsa_queue_destroy call)
 * Similarly, you can generate a summary while collecting a trace using the following command line: `./CodeXLGpuProfiler --hsatrace --tracesummary ApptoProfile`
* The following switches should also work:
 * `--envvar`
 * `--envvarfile`
 * `--fullenv`
 * `--sessionname`
 * `--workingdirectory`
 * `--interval`
 * `--maxapicalls`
 * `--apifilterfile`
 * `--sym`
* The following switches are not applicable to the ROCm Profiler:
 * `--nocollapse`
 * `--ret`

You can get more information on the switches mentioned above by reading the CodeXL User Guide or by executing:

  `./CodeXLGpuProfiler --hsatrace --help`
  
A version of the CXLActivityLogger instrumentation library which is supported
by the ROCm profiler is also included in this distribution. Using this API, you
can annotate your code and have the annotations appear in the timeline UI in
CodeXL. The latest version of the CXLActivityLogger library also contains new
APIs for controlling the collection of profiling data from within a profiled
application. Documentation on the CXLActivityLogger API can be found in the
distribution.

You can load an HSA .atp file into CodeXL 2.2 in order to see the application
timeline. See information below on where to download CodeXL from.

## Collecting GPU Performance Counters

To collect GPU performance counters:

   `./CodeXLGpuProfiler --hsapmc AppToProfile`

Executing CodeXLGpuProfiler with the `--hsapmc` switch will launch the specified
"AppToProfile" and allow the profiler to collect GPU Performance Counter
data for each kernel dispatched by the application. The output file will
contain the following information for each kernel dispatched:
* the name of the kernel symbol associated with the kernel dispatched (or "<UnknownKernelName>" if the profiler could not extract the symbol name.)
* The following set of dispatch parameters and compiler stats for each kernel:
 * the thread id of the host thread that dispatched the kernel
 * the dimensions of the grid (in work-items)
 * the dimensions of the work-group (in work-items)
 * the size of the shared memory used by the work-group (the work-group group segment size)
 * the number of vector GPRs used by the kernel
 * the number of scalar GPRs used by the kernel
* The values of any performance counters collected for the kernel
  

Most other CodeXLGpuProfiler switches will also work with the `--hsapmc` switch:

* You can control the name and location of the .csv file using CodeXLGpuProfiler's `--outputfile` switch.
* You can specify the performance counters to collect using the `--counterlist` switch.
* A kernel occupancy file can be generated while collecting performance counters by specifying the `--occupancy` switch.
* The following switches should also work:
 * `--envvar`
 * `--envvarfile`
 * `--fullenv`
 * `--sessionname`
 * `--workingdirectory`
 * `--interval`
 * `--maxkernels`
* The following switches are either not implemented yet or are not applicable to the ROCm Profiler:
 * `--kerneloutput`
 * `--kernellistfile`
 * `--singlepass`
 * `--nogputime`

The ROCm Profiler can only support collecting a set of performance counters that
can be collected in a single pass. It does not support kernel replay like the
OpenCL profiler. Thus, if you do not specify a list of performance counters to
collect, or if the set you specify can not be collected in a single pass, the
profiler will not collect all of the counters. It will enable as many counters
as it can fit into a single pass. To collect a full set of performance counters
for an application, you will need to profile the application multiple times and
specify a different set of counters for each profile run. Because of this
restriction, the `--singlepass` and `--nogputime` switches are always enabled
when profiling a ROCm application.

## System Setup 

This assumes you are starting from a system where you can run ROCm applications
outside of the profiler. The information here provides only additional steps
you need to perform to be able to profile ROCm applications. Please refer to
https://radeonopencompute.github.io for runtime and driver installation information.

In order to profile, you will need to make sure that the ROCR runtime
libraries (libhsa-runtime64.so, libhsa-runtime-ext64.so,
libhsa-runtime-tools64.so, and libhsakmt.so) can be found and loaded by the
application you are profiling. After a default installation, these libraries
are typically located in /opt/rocm/hsa/lib.

The ROCm Profiler is installed by default when following the installation
instructions available at https://radeonopencompute.github.io/install.html.
After successful installation, the profiler can be found in /opt/rocm/profiler.
There is also a link created at /opt/rocm/bin/rocm-profiler which can be used
to execute the profiler.

## Sample Usage

You can profile the vector_copy sample included in the ROCm release (typically in /opt/rocm/hsa/sample)
using the following steps:
 * Build the vector_copy sample using make
 * Verify that the sample executable runs
 * Execute `./CodeXLGpuProfiler --hsatrace vector_copy`
 * Execute `./CodeXLGpuProfiler --hsapmc vector_copy`

## Using this build with CodeXL2.2

This build is compatible with CodeXL 2.2, which can be downloaded from the
following location:

    https://github.com/GPUOpen-Tools/CodeXL/releases/tag/v2.2

The binaries included in this repository are newer than the binaries included in
CodeXL 2.2. In order to use this version of the profiler from within CodeXL, simply
copy the files in the bin directory into the main CodeXL directory (to replace the files
in CodeXL with the same-named files from here).

In order to see data transfer timing info on the timeline in CodeXL, a version newer than
2.2 is required. For now, you will need to clone and build CodeXL using the master branch
in order to see data transfers on the timeline.

## Building the ROCm Profiler

The source code of the ROCm Profiler is part of the [CodeXL repository](https://github.com/GPUOpen-Tools/CodeXL). See the [BUILD.md](https://github.com/GPUOpen-Tools/CodeXL/blob/master/BUILD.md) for instructions on building CodeXL. To build just the ROCm Profiler, execute the [backend_build.sh script](https://github.com/GPUOpen-Tools/CodeXL/blob/master/CodeXL/Components/GpuProfiling/Build/backend_build.sh) with the following command line:
 * ./backend_build.sh skip-oclprofiler skip-32bitbuild

By default the build will look for the HSA header files under /opt/rocm/hsa. To override this location, add the "hsadir" parameter:
 * ./backend_build.sh skip-oclprofiler skip-32bitbuild hsadir &lt;location&gt; 

## Related links
[ROCm Profiler blog post](http://gpuopen.com/getting-up-to-speed-with-the-codexl-gpu-profiler-and-radeon-open-compute/)

## Known Issues
* API Trace and Perf Counter data may be truncated or missing if the application being profiled does not call hsa_shut_down
* Kernel occupancy information will only be written to disk if the application being profiled calls hsa_shut_down
* When collecting a trace for an application that performs memory transfers using hsa_amd_memory_async_copy, if the application asks for the data transfer timestamps directly, it will not get correct timestamps. The profiler will show the correct timestamps, however.
* When collecting an aql packet trace, if the application asks for the kernel dispatch timestamps directly, it will not get correct timestamps. The profiler will show the correct timestamps, however.
* When the rocm-profiler package (.deb or .rpm) is installed along with rocm, it may not be able to generate the default single-pass counter files. If you do not see counter files in /opt/rocm/profiler/counterfiles, you can generate them manually with this command: "sudo /opt/rocm/profiler/bin/CodeXLGpuProfiler --list --outputfile /opt/rocm/profiler/counterfiles/counters --maxpassperfile 1"
