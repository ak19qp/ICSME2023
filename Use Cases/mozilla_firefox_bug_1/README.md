# Use Case: Firefox CSS Animation Rendering Bug

In this use case, we will examine a performance bug reported for Firefox within Mozilla's [Bugzilla repository](https://bugzilla.mozilla.org/show_bug.cgi?id=1637586). According to the report, Firefox encounters severe performance degradation during the rendering of CSS animations; a task handled quite competently by its contemporaneous browser counterparts. Despite the initial reporting of this performance bug dating back several years, the issue persistently remains unresolved. The report identifies functions within the GFX Web Renderer class as harboring the malfeasant functions responsible for this issue. Our proposed methodology has successfully corroborated this assertion by detecting the functions from the same culpable class.

## TL;DR Note:
If you just want to evaluate our `perf_perser_and_esd.py` script without going through the process of building Firefox and recording the perf data, then please follow the TL;DR sections of [Use Case: "ls" Slow Performance in Large Directories](https://github.com/ak19qp/ICSME2023/tree/main/Use%20Cases/ls_bug) or [Use Case: Firefox Tripadvisor.ca CPU Exhaustion Bug](https://github.com/ak19qp/ICSME2023/tree/main/Use%20Cases/mozilla_firefox_bug_2)

## Use Case Setup

In the context of this use case, we have reproduced the bug reported in the [Bugzilla report](https://bugzilla.mozilla.org/show_bug.cgi?id=1637586). The [3D CSS animation](https://looping-squares.superhi.com/) was repeatedly loaded in Firefox to unveil all potential functions correlated with a system call for performing this task. The Firefox browser employed for this investigation was compiled from the source with the 'perf' flag enabled, thus enabling Just-In-Time (JIT) profiling following the Firefox documentation by Mozilla.

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

4. Replicate the bug mentioned in the [bugzilla repository](https://bugzilla.mozilla.org/show_bug.cgi?id=1637586).

5. Once the task has been done, stop Perf record by pressing `Ctrl+C`. Wait for it to gracefully exit, don't press the keys more than once.
   
6. Important note only for the artifact evaluation: Remember to NOT run Perf for too long as it will generate lots of data which will take a long time to be processed by the script. For the purpose of review, we recommend running Perf for no more than 5 seconds. You must also ensure that firefox pid is correctly hooked, otherwise it will record huge system-wide data.

7. Now run the following commands in the terminal to convert the perf data into readable text:\
   `sudo perf script > [give a name to the output file]`\
   Assuming the output file name is `pcsdata` to reference with the next sections.

9. For performing Enhanced Statistical Debugging (ESD), we need the `pcsdata` file, a threshold method (1. Mean+Stdv, 2. Fixed) and the `perf_perser_and_esd.py` python script from this repository.
   - For Mean+Stdv threshold method `(threshold_type = 1)`, execute the following command in the terminal:
     
     `python3 perf_perser_and_esd.py [pcsdata/input file name] [output file name] 1`\
     Here the threshold that defines success and fail runs will be decided based on the individual function's mean+stdv of their overall wait time in the sample data.
   - For a Fixed threshold method `(threshold_type = 2)`, execute the following command in the terminal:
     `python3 perf_perser_and_esd.py [pcsdata/input file name] [output file name] 2 [enter threshold here in milliseconds]`\
     As an example, if 10 was selected as the threshold, then whenever a function experienced a wait time of 10 milliseconds or higher in a system call, those runs would be considered as a fail run and vice versa.
   - For referencing to the next section, we will be using `esddata.csv` as the output file name.


### Analysis
Once we have the `esddata.csv` file, which will be in a `comma-separated values (CSV)` format, this file could be opened in MS Excel or LibreOffice Calc to visualize and perform sorting. Once it is loaded in a spread sheet viewing tool, you can sort (in descending order) the output based on `Increase` to find the ranked list of prospective, suspicious and potentially problematic functions that need to be monitored for performance issues. This `esddata.csv` file could also be used with other scripts for performing your own analysis.

## Discussion
After collecting the trace events, the performance metric is defined, which in this case is the system call duration, calculated from the timestamp difference between entry and exit of the respective system calls. Based on this metric and the call stack data provided by Perf, the failure or success label is assigned to the call stack. In this use case, 1316 unique functions are identified. For each function, statistical debugging metrics (Failure, Context, and Increase) are calculated. The Increase value of the functions are then used to sort and rank the functions. A cut-off value is then applied, disregarding anything below the top 15% of the list. This reveals the functions that are more directly implicated in failures, with the most significant ones appearing at the very top of the list. Table I highlights the culprit functions responsible for the performance bugs found in the ranked cut-off list.

![Table: Results](https://github.com/ak19qp/ICSME2023/blob/main/Use%20Cases/mozilla_firefox_bug_1/cs1_table.PNG)

Through our use case, we have recognized the substantial value of applying statistical debugging techniques, specifically using runtime data of kernel events such as system call waiting time, to uncover critical functions that impact the performance of CSS animation rendering in Firefox. Table I presents the results of the proposed PASD, which identifies three crucial Firefox functions in the `WebRenderCommandBuilder` class, matching the findings reported in [Bugzilla](https://bugzilla.mozilla.org/show\_bug.cgi?id=1637586). This external validation enhances the credibility and reliability of our findings, ensuring internal consistency.

It is important to note that in statistical debugging, 'Increase' values can occasionally be negative, as demonstrated in Table I.However, we do not exclude functions that exhibit a negative 'Increase' value, since this metric is only employed for the purpose of ranking potential functions, not for any other analytical process.

Given that our analysis is based solely on runtime function calls and does not have access to the code branches (as defined by original definitions of predicates), it is feasible to observe negative 'Increase' values, even for functions that are the root cause of issues. this arises due to two main reasons outlined below:
- Interactions with other predicates: The negative 'Increase' value of a specific predicate may be influenced by interactions with other predicates or conditions in the code.
- Hidden dependencies: Certain predicates may have hidden dependencies or indirect effects on failures. These dependencies may not be immediately apparent and can result in a negative 'Increase' value.

Therefore, irrespective of the 'Increase' value being positive, zero, or negative, we do not directly use it to discard candidate functions. Instead, we employ it to only sort and rank the list of candidate functions.


## Notes
- If the addresses converted to names show no function names, then it is highly likely that the `ac_add_options --enable-perf` option for mozconfig was not correctly setup during the firefox build.
- Address to name translation might take a long time if perf data was too large. We recommend not running `perf record` for more than 5 seconds for evaluation and review purposes of this artifact.
- Average time to complete ESD as well as address to name translation for roughly 1GB perf script output file is around 1hr. So, it must be kept under 200 MB (by reducing perf record time) for faster review.
