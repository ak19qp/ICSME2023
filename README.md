# PASD: A Performance Analysis Approach Through the Statistical Debugging of Kernel Events

Mohammed Adib Khan\
Department of Computer Science\
Brock University\
St. Catharines, Canada\
ak19qp@brocku.ca


Morteza Noferesti\
Department of Computer Science\
Brock University\
St. Catharines, Canada\
mnoferesti@brocku.ca


Naser Ezzati-Jivan\
Department of Computer Science\
Brock University\
St. Catharines, Canada\
nezzati@brocku.ca

## Abstract:
Dynamic performance analysis is vital for optimizing systems and finding performance bottlenecks. Traditional debugging struggles to locate issues in complex software due to hidden performance problems during execution. This artifact uses PASD (Performance Analysis through Statistical Debugging), which analyzes kernel-level trace events without altering application code. It identifies key functions causing performance problems, maintaining normal software operations, and avoiding added complexity. PASD offers deep insights into kernel-level behavior, helping developers pinpoint bottlenecks, enhance efficiency, and improve software quality, user satisfaction, and system stability.

## This repository contains the artifact needed to:

### Data Collection:
- Record kernel events and their relevant call stacks.
- Extract functions from the call stacks.
- Extract relevant data to calculate the wait time of those kernel events and thereby the wait time of the functions themselves.

### Performance Debugging:
- Define threshold to differentiate success and fail runs.
- Find fail, fail observed, success and success ovserved for each unique functions.
- Perform Enhanced Statistical Debugging (ESD) on the functions' data.
- Rank the suspicious/problematic application functions responsible for performance issues based on the ESD data.


## Minimum System Requirements:

Operating System: Ubuntu 18+\
RAM: 8GB\
Storage: 50GB\
Processor: Intel i3 2.5 GHz

## Setup

1. Perf:
- Open terminal and enter this command:\
  `sudo apt install linux-tools-common`
- Verify installation by typing in "perf" in the terminal.

2. Firefox:
- Install Python with the following command in the terminal:\
  `sudo apt-get install curl python3 python3-pip`
  
- Install Mercurial with the following command in the terminal:\
  `python3 -m pip install --user mercurial`
  
- Test Mercurial with this command:\
  `hg version`
- Bootstrap a copy of the Firefox source code with these commands in terminal (make sure to cd to the location where you want to download first):\
  `curl https://hg.mozilla.org/mozilla-central/raw-file/default/python/mozboot/bin/bootstrap.py -O`\
  `python3 bootstrap.py`
  
- Build Firefox:
  First modify the mozconfig file in the downloaded source folder by adding `ac_add_options --enable-perf` in the file. If the `mozconfig` file is not present then create an empty file named "mozconfig" and put in the command there.\
  Now execute these commands inside the root directory of the source folder:\
  `hg up -C central`\
  `./mach build`

## Official Firefox build guides:

- [Linux Build](https://firefox-source-docs.mozilla.org/setup/linux_build.html)
- [JIT Profiling](https://firefox-source-docs.mozilla.org/performance/jit_profiling_with_perf.html)
- [Configuring Build Options](https://firefox-source-docs.mozilla.org/setup/configuring_build_options.html)

## Data collection, generation and artifact execution:

1. To run the built Firefox, cd into the root directory of the source folder and use these commands in the terminal:\
  `export IONPERF=ir`\
  `./mach run`

2. Find the PID of Firefox. Since Firefox has a lot of child processes, it is easier to find its parent process' PID using the following command:\
   `pstree -p | grep "firefox" | head -1`\
   Make sure that you are not copying some other firefox's PID such that you already use Firefox in your PC and it is already running at the same time as the built Firefox.

4. Download the script perf_perser_and_esd.py from this repository and save it in a folder. Then open a terminal in that folder directory and run the following commands:\
  `sudo perf record -g -e 'syscalls:sys_*' -p (PID of firefox)`

5. Run some firefox tasks or you may try to replicate some firefox bugs from bugzilla if time permits (however, that will require you to rebuilt the correct Firefox version (mentioned in the bugzilla report) again with the perf option enable).

6. Once the task has been done, stop the Perf record by pressing `Ctrl+C`. Wait for it to gracefully exit, don't press the keys more than once.
   
7. Important note only for the artifact evaluation: Remember to NOT run Perf for too long as it will generate lots of data which will take a long time to be processed by the script in this repo. For the purpose of review, we recommend running Perf for no more than 5 seconds. You must also ensure that firefox pid is correctly hooked with the command mentioned in step 4, otherwise it will record huge systemwide data.

9. Now run the following commands in the terminal to convert the perf data into readable text:\
   `sudo perf script > test.txt`
- Don't change the output file name. If change is needed then you must change this name in the codes of the python script from this repo as well.

8. Open the downloaded script `perf_perser_and_esd.py` from this repo in a text editor. Set variable `threshold_type` in line 10 to `1` to use mean + standard deviation as the threshold that will be used to determine a fail run. Any wait or execution time below this threshold will be considered a success run. You may set `threshold_type` to `2` and then set the value in milliseconds for the variable `threshold_value_for_type_2` to use a hardcoded threshold.

9. Finally, run the python script using the command in the terminal:\
  `python3 perf_perser_and_esd.py`
- This script will calculate execution time / wait time of system calls, the functions which made the system calls by analyzing the call stacks, and finally perform enhanced statistical debugging and generate a csv output. You can sort the output based on `Increase` using MS Excel or LibreOffice Calc to find the ranked list of prospective and suspicious and potentially problematic functions that need to be monitored for performance issues.

## Notes
- If the addresses converted to names show no function names, then it is highly likely that the `ac_add_options --enable-perf` option for mozconfig was not correctly setup during the firefox build.
- Address to name translation might take a long time if perf data was too large. We recommend not running `perf record` for more than 5 seconds for evaluation and review purposes of this artifact.
- Average time to complete ESD as well as address to name translation for roughly 1GB perf script output file is around 1hr. So, it must be kept under 200 MB (by reducing perf record time) for faster review.
