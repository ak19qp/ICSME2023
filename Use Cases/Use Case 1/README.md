# Case Study 1: Firefox CSS Animation Rendering Bug

In this use case, we will examine a performance bug reported for Firefox within Mozilla's [Bugzilla repository](https://bugzilla.mozilla.org/show_bug.cgi?id=1637586). According to the report, Firefox encounters severe performance degradation during the rendering of CSS animations; a task handled quite competently by its contemporaneous browser counterparts. Despite the initial reporting of this performance bug dating back several years, the issue persistently remains unresolved. The report identifies functions within the GFX Web Renderer class as harboring the malfeasant functions responsible for this issue. Our proposed methodology has successfully corroborated this assertion by detecting the functions from the same culpable class.


## Case Study Setup

In the context of this case study, we have reproduced the bug reported in the [Bugzilla report](https://bugzilla.mozilla.org/show_bug.cgi?id=1637586). The [3D CSS animation](https://looping-squares.superhi.com/) was repeatedly loaded in Firefox to unveil all potential functions correlated with a system call for performing this task. The Firefox browser employed for this investigation was compiled from the source with the 'perf' flag enabled, thus enabling Just-In-Time (JIT) profiling following the Firefox documentation by Mozilla.

### Setting-up Firefox:

- Install Mercurial with the following command in the terminal (must have pip installed):
  
  `python3 -m pip install --user mercurial`
  
- Test Mercurial with this command:
  
  `hg version`
  
- Bootstrap a copy of the Firefox source code with these commands in terminal (make sure to cd to the location where you want to download first):
  
  `curl https://hg.mozilla.org/mozilla-central/raw-file/default/python/mozboot/bin/bootstrap.py -O`\
  `python3 bootstrap.py`
  
- Build Firefox:

  First modify the mozconfig file in the downloaded source folder by adding `ac_add_options --enable-perf` in the file. If the `mozconfig` file is not present then create an empty file named "mozconfig" and put in the command there.\
  Now execute these commands inside the root directory of the source folder:\
  `hg up -C central`\
  `./mach build`

- To run this built Firefox execute the following commands in the terminal:

  `export IONPERF=ir`\
  `./mach run`

#### Official Firefox build guides:

- [Linux Build](https://firefox-source-docs.mozilla.org/setup/linux_build.html)
- [JIT Profiling](https://firefox-source-docs.mozilla.org/performance/jit_profiling_with_perf.html)
- [Configuring Build Options](https://firefox-source-docs.mozilla.org/setup/configuring_build_options.html)



### Data collection:

1. Find the PID of Firefox. Since Firefox has a lot of child processes, it is easier to find its parent process' PID using the following command:\
   `pstree -p | grep "firefox" | head -1`\
   Make sure that you are not copying some other firefox's PID such that you already use Firefox in your PC and it is already running at the same time as running the built Firefox.

2. Download the script `perf_perser_and_esd.py` from this repository and save it in a folder. Then open a terminal in that folder directory and execute the following command:
   
  `sudo perf record -g -e 'syscalls:sys_*' -p [PID of firefox]`

3. Replicate the bug mentioned in the [bugzilla repository](https://bugzilla.mozilla.org/show_bug.cgi?id=1637586).

4. Once the task has been done, stop Perf record by pressing `Ctrl+C`. Wait for it to gracefully exit, don't press the keys more than once.
   
5. Important note only for the artifact evaluation: Remember to NOT run Perf for too long as it will generate lots of data which will take a long time to be processed by the script. For the purpose of review, we recommend running Perf for no more than 5 seconds. You must also ensure that firefox pid is correctly hooked, otherwise it will record huge system-wide data.

6. Now run the following commands in the terminal to convert the perf data into readable text:\
   `sudo perf script > [give a name to the output file]`\
   Assuming the output file name is `pcsdata` to reference with the next sections.

7. For performing Enhanced Statistical Debugging (ESD), we need the `pcsdata` file, a threshold method (1. Mean+Stdv, 2. Fixed) and the `perf_perser_and_esd.py` python script from this repository.
   - For Mean+Stdv threshold method `(threshold_type = 1)`, execute the following command in the terminal:
     
     `python3 perf_perser_and_esd.py [pcsdata/input file name] [output file name] 1`\
     Here the threshold that defines success and fail runs will be decided based on the individual function's mean+stdv of their overall wait time in the sample data.
   - For a Fixed threshold method `(threshold_type = 2)`, execute the following command in the terminal:
     `python3 perf_perser_and_esd.py [pcsdata/input file name] [output file name] 2 [enter threshold here in milliseconds]`\
     As an example, if 10 was selected as the threshold, then whenever a function experienced a wait time of 10 milliseconds or higher in a system call, those runs would be considered as a fail run and vice versa.
   - For referencing to the next section, we will be using `esddata` as the output file name.


### Analysis
- This script will calculate execution time / wait time of system calls, the functions which made the system calls by analyzing the call stacks, and finally perform enhanced statistical debugging and generate a csv output. You can sort the output based on `Increase` using MS Excel or LibreOffice Calc to find the ranked list of prospective and suspicious and potentially problematic functions that need to be monitored for performance issues.

## Notes
- If the addresses converted to names show no function names, then it is highly likely that the `ac_add_options --enable-perf` option for mozconfig was not correctly setup during the firefox build.
- Address to name translation might take a long time if perf data was too large. We recommend not running `perf record` for more than 5 seconds for evaluation and review purposes of this artifact.
- Average time to complete ESD as well as address to name translation for roughly 1GB perf script output file is around 1hr. So, it must be kept under 200 MB (by reducing perf record time) for faster review.
