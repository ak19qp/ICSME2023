# PASD: A Performance Analysis Approach Through the Statistical Debugging of Kernel Events

This repository contains the artifact needed to:
1. Generate Perf data related to kernel events (we will be using system calls as an example) made by applications (we will be using Firefox as an example).
2. Extract functions from callstack while those kernel events took place.
3. Extract relevant data to calculate the wait time length of those kernel events.
4. Perform Enhanced Statistical Debugging (ESD) on the extracted data.
5. Rank the suspicious/problematic application functions responsible for performance issues based on the ESD data.

Usually when a callstack data is read, it will show memory addresses of the functions. This is not useful for debugging in the sense that we do not know which exact function we are looking at. Therefore, these addresses needs to be translated into their function names. This could be done with debugger info enabled with the built and if the binaries are available for the application. We will be using Firefox for this example as they already provide a built-in support for Perf.

## Minimum System Requirements:

Operating System: Ubuntu 18+\
RAM: 8GB\
Storage: 50GB\
Processor: Intel i3 2.5 GHz (need enough to run Firefox smoothly)

## Setup

1. Perf:
- Open terminal and enter this command:\
  sudo apt install linux-tools-common
- Verify installation by typing in "perf" in the terminal.

2. Firefox:
- Install Python with the following command in terminal:\
  sudo apt-get install curl python3 python3-pip
  
- Install Mercurial with the following command in terminal:\
  python3 -m pip install --user mercurial
  
- Test Mercurial with this command:\
  hg version
- Bootstrap a copy of the Firefox source code with these commands in terminal (make sure to cd to the location where you want to download first):\
  curl https://hg.mozilla.org/mozilla-central/raw-file/default/python/mozboot/bin/bootstrap.py -O\
  python3 bootstrap.py
  
- Build Firefox:
  First modify the mozconfig file in the downloaded source folder by adding "ac_add_options --enable-perf" in the file. If the "mozconfig" file is not present then create an empty file named "mozconfig" and put in the command there.\
  Now execute these commands inside the root directory of the source folder:\
  hg up -C central\
  ./mach build

## Official Firefox build guides:

- [Linux Build](https://firefox-source-docs.mozilla.org/setup/linux_build.html)
- [JIT Profiling](https://firefox-source-docs.mozilla.org/performance/jit_profiling_with_perf.html)
- [Configuring Build Options](https://firefox-source-docs.mozilla.org/setup/configuring_build_options.html)

## Data collection, generation and artifact execution:

1. To run the built Firefox, cd into the root directory of the source folder and use these commands in the terminal:\
  export IONPERF=ir\
  ./mach run

2. Find the PID of Firefox. Since, Firefox has a lot of child processes, it easier to find it's parent process' PID using the following command:\
   pstree -p | grep "firefox" | head -1\
   Make sure that you are not copying some other firefox's PID such that you already use Firefox in your PC and it is already running at the same time as the built Firefox.

4. Download the script perf_perser_and_esd.py from this repository and save it in a folder. Then open a terminal in that folder directory and run the following commands:\
  sudo perf record -g -e 'syscalls:sys_*' -p (PID of firefox)

5. Run some firefox tasks or you may try to replicate some firefox bugs from bugzilla.

6. Once the task has been done, stop Perf record by pressing Ctrl+C. Wait for it to gracefully exit, don't press they keys more than once.
   
7. Important note only for the artifact evaluation: Remember to NOT run Perf for too long as it will generate lots of data which will take a long time to be processed by the script in this repo. For the purpose of review, we recommend running Perf for no more than 5 seconds. You must also ensure that firefox pid is correctly hooked with the command mentioned in step 4, otherwise it will record huge systemwide data.

9. Now run the following commands in the terminal to convert the perf data into readable text:\
   sudo perf script > test.txt
- Don't change the output file name test.txt. If change is needed then you must change this name in the codes of the python script from this repo as well.

8. Open the downloaded script "perf_perser_and_esd.py" from this repo and select a threshold type. Set variable "threshold_type" at the top to 1 to use mean + standard deviation as the threshold that will be used to determine a fail run. Or set it to 2 and then set the value in milliseconds for the variable threshold_value_for_type_2 to use a hardcoded threshold.

9. Finally run the python script using the command in the terminal:\
  python3 perf_perser_and_esd.py
- This script will calculate execution time / wait time of system calls, the functions which made the system calls by analyzing the call stacks, and finally perform enhanced statistical debugging and generate a csv output. You can sort the output based on 'Increase' using MS Excel or LibreOffice Calc to find the ranked list of prospective and suspicious and potentially problematic functions that needs to be monitored for performance issues.

## Notes
- If the addresses converted to names show no function names, then it is highly likely that the "ac_add_options --enable-perf" option for mozconfig was not correctly done during the firefox build.
- Address to name translation might take a long time if perf data was too large. We recommend not running perf record for more than 5 seconds for evaluation and review purposes of this artifact.
- Average time to complete ESD as well as address to name translation for roughly 1GB perf script output file is around 1hr. So, it must be kept under 200 MB (by reducing perf record time) for faster review.
