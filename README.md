# ROCm Profiler April 2016 Release

## Overview
This release of the ROCm Profiler is compatible with the ROCm 1.0
release that appears in the following repositories:

https://github.com/RadeonOpenCompute/ROCm

This build of the profiler is supported on Carrizo, and Fiji (Boltzmann).  Kaveri
support has been deprecated, and while the profiler should still work, there may be some
issues when using it on Kaveri.

The ROCm Profiler is integrated into the CodeXL GPU Profiler Backend (aka
"CodeXLGpuProfiler").  CodeXL is the official delivery vehicle for the ROCm
Profiler, but it is being made available here as well for those who want to
use the profiler independently of CodeXL.

Information contained here is specific to CodeXLGpuProfiler version 4.0.12417. This build
is functionally equivalent to the version of the profiler included in [CodeXL 2.0](https://github.com/GPUOpen-Tools/CodeXL)

##Table of Contents
* [What's New](#WhatsNew)
* [Collecting an Application Trace](#ApplicationTrace)
* [Collecting GPU Performance Counters](#PerfCounters)
* [System Setup](#SystemSetup)
* [Sample Usage](#SampleUsage)
* [Using with CodeXL 2.0](#CodeXL2.0)
* [Related Links](#RelatedLinks)
* [Known Issues](#KnownIssues)
* [License](Legal/CodeXLEndUserLicenseAgreement-Linux.htm)

<ANAME="WhatsNew">
## What's New

* Profiler executable has been renamed from "sprofile" to "CodeXLGpuProfiler"
* Support for ROCm 1.0
* Support for CodeXL 2.0
* CodeXL is now open-sourced.  As the ROCm Profiler is a component of CodeXL, the source code of the profiler is part of the [CodeXL repository](https://github.com/GPUOpen-Tools/CodeXL)
* Many bug fixes to improve stability and functionality

<A NAME="ApplicationTrace">
## Collecting an Application Trace

To collect an application trace with kernel timestamps:

   `./CodeXLGpuProfiler --hsatrace AppToProfile`

Executing CodeXLGpuProfiler with the `--hsatrace` switch will launch the specified
"AppToProfile" and allow the profiler to trace all HSA APIs called by the
application. In addition, the profiler will also gather 2 sets of timestamp
information:
* Host-side CPU timestamps for all HSA APIs called by the application
* Device-side GPU timestamps for all HSA kernels dispatched by the application

The results from the profiler will be written to an .atp file that is very
similar to the .atp file written by CodeXL's OpenCL profiler.  When profiling
a ROCm application, the .atp file will have four main sections, the first three
of which are similar to the sections written by the OpenCL profiler. The CodeXL
documentation contains detailed information on the contents of the .atp file
for an OpenCL application.

Here is a brief description of the .atp file sections for the ROCm profiler:
* The first section is a file header containing information about the application profiled, the system the profile was gathered on, and some of   the options used to gather the profile.
* The second section is the API trace, showing a per-thread list of all HSA APIs called by the application. For each API, the parameters passed and the values returned by the the API are shown.
* The third section is the host-side timestamps section, showing a per-thread list of all HSA APIs called by the application.  For each API, a start time and end time is shown.
* The fourth section is new for HSA.  It is the device-side kernel timestamp section, showing a list of each kernel dispatched by the application.  For each kernel, the following information is shown:
 * the name of the kernel symbol associated with the kernel dispatched (or "<UnknownKernelName>" if the profiler could not extract the symbol name.)
 * the kernel pointer value of the kernel dispatched.
 * the start timestamp of the kernel, indicating when the kernel started executing on the device. The timestamps represent nanoseconds.
 * the end timestamp of the kernel, indicating when the kernel finished executing on the device.  The timestamps represent nanoseconds.
 * the duration of the kernel (in nanoseconds).
 * the agent handle of the agent the kernel was dispatched to.

Most other CodeXLGpuProfiler switches will also work with the `--hsatrace` switch:

* You can control the name and location of the .atp file using CodeXLGpuProfiler's `--outputfile` switch.
* You can generate a subset of the Trace Summary pages using CodeXLGpuProfiler's `--tracesummary` and `--atpfile` switch.  For instance: `./CodeXLGpuProfiler --tracesummary --atpfile myapp.atp` will generate an API summary, a kernel summary and a top-ten kernel dispatch list from the data contained in the myapp.atp file. It will also produce a "Best Practices" summary file by applying a rules-based analysis of the .atp file.  Currently two rules are supported:
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
  
A version of the AMDTActivityLogger instrumentation library which is supported
by the ROCm profiler is also included in this distribution. Using this API, you
can annotate your code and have the annotations appear in the timeline UI in
CodeXL. The latest version of the AMDTActivityLogger library also contains new
APIs for controlling the collection of profiling data from within a profiled
application. Documentation on the AMDTActivityLogger API can be found in the
distribution.

You can load an HSA .atp file into CodeXL 2.0 in order to see the application
timeline.  See information below on where to download CodeXL from.

<A NAME="PerfCounters">
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
OpenCL profiler.  Thus, if you do not specify a list of performance counters to
collect, or if the set you specify can not be collected in a single pass, the
profiler will not collect all of the counters.  It will enable as many counters
as it can fit into a single pass. To collect a full set of performance counters
for an application, you will need to profile the application multiple times and
specify a different set of counters for each profile run.  Because of this
restriction, the `--singlepass` and `--nogputime` switches are always enabled
when profiling a ROCm application.

<A NAME="SystemSetup">
## System Setup 

This assumes you are starting from a system where you can run ROCm applications
outside of the profiler. The information here provides only additional steps
you need to perform to be able to profile ROCm applications.  Please refer to
https://github.com/RadeonOpenCompute/ROCm for runtime and driver
installation information.

In order to profile, you will need to make sure that the ROCR runtime
libraries (libhsa-runtime64.so, libhsa-runtime-ext64.so,
libhsa-runtime-tools64.so, and libhsakmt.so) can be found and loaded by the
application you are profiling. After a default installation, these libraries
are typically located in /opt/rocm/hsa/lib.

<A NAME="SampleUsage">
## Sample Usage

You can profile the vector_copy sample included in the ROCm release (typically in /opt/rocm/hsa/sample)
using the following steps:
 * Build the vector_copy sample using make
 * Verify that the sample executable runs
 * Execute `./CodeXLGpuProfiler --hsatrace vector_copy`
 * Execute `./CodeXLGpuProfiler --hsapmc vector_copy`

<A NAME="CodeXL2.0">
## Using this build with CodeXL2.0

This build is compatible with CodeXL 2.0, which can be downloaded from the
following location:

    https://github.com/GPUOpen-Tools/CodeXL/releases/tag/v2.0

At this time, the binaries included in this repository are functionally equivalent
to the binaries included in CodeXL 2.0.  Should this repository be updated with a
new build, this section will be updated to provide instructions on how to use the
new build from within CodeXL.

<A NAME="RelatedLinks">
## Related links
[ROCm Profiler blog post](http://gpuopen.com/getting-up-to-speed-with-the-codexl-gpu-profiler-and-radeon-open-compute/)

<A NAME="KnownIssues">
## Known Issues
* API Trace and Perf Counter data may be truncated if the application being profiled does not call hsa_shut_down
* Kernel occupancy information will only be written to disk if the application being profiled calls hsa_shut_down
* API Trace data does not include any information about AMD-specific extension functions called by the application
