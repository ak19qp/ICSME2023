# Use Case: "ls" Slow Performance in Large Directories

In this case study, we investigated a performance bug that arises from the usage of the 'ls' command in Linux, which is a component of the GNU Core Utilities package. The 'ls' command is a commonly used command-line utility in Unix-like operating systems. Its purpose is to display a list of files and directories within a specified directory or the current working directory. When executed without any arguments, 'ls' will present the contents of the current directory, providing a concise listing that includes the names of files and directories.

Reports on various online discussion platforms: [serverfault](https://serverfault.com/questions/316951/why-might-ls-color-always-be-slow-for-a-small-directory), [superuser](https://superuser.com/questions/1345268/ls-command-very-slow) as well as RedHat BugZilla reports: [1](https://bugzilla.redhat.com/show_bug.cgi?id=1290036), [2](https://bugzilla.redhat.com/show_bug.cgi?id=467508), have indicated that 'ls' exhibits significant slowness when dealing with directories containing a large number of files and folders. Furthermore, it is also slow when handling a large number of top-level entries. This issue specifically arises when color coding is enabled for the 'ls' command.

## TL;DR Note:
If you just want to evaluate our `perf_perser_and_esd.py` script without going through the process of building ls and recording the perf data, then please download the data folder from this use case folder and run the following command in the terminal from within the data folder directory:

`tar -xvf case_study_3.tar.gz`

Once the file has been unzipped the execute the following command from the `perf_perser_and_esd.py` file's directory:

### For Mean+Stdev as threshold:
`python3 perf_perser_and_esd.py case_study_3.txt esddata.csv 1`

### For fixed threshold:
`python3 perf_perser_and_esd.py case_study_3.txt esddata.csv 2 [enter threshold in milliseconds]`

## Case Study Setup & Data Collection

For this use case, we began by installing the libc6-dbg package using the command 'sudo apt-get install libc6-dbg'. This ensured that the libc compiler had debugger information available for the compiled program. We also installed the build-essential package in the same manner to facilitate the building process. Subsequently, we downloaded the source code of the GNU Core Utilities package and proceeded to extract and compile it with debugger information enabled.

Once the necessary preparations were complete, we utilized a bash script to generate a substantial number of random files and folders, along with numerous top-level directories. We employed the 'Perf' tool to execute the 'ls' command and record call stacks whenever system calls were made by 'ls'. The command used for this purpose was `perf record -a -g -e 'syscalls:sys\_*' /[coreutils directory]/src/ls --color=always /[directory to inquire]`. Additionally, we also repeated the call stack recording process with Perf while disabling the color parameter of 'ls' and collect 276,412 call stack data. To obtain a comprehensive dataset for statistical analysis, we executed 'ls' through Perf multiple times under various loads and stress conditions.

## Analysis & Discussion

In order to analyze we first have to convert the perf data into readable data using the command:

`perf script > case_study_3.txt`

Next we executed the script using the command:

`python3 perf_perser_and_esd.py case_study_3.txt esddata.csv 1`

The `esddata.csv` file is then opened with MS Excel and the unique functions were then sorted based on their 'Increase' value and ranked accordingly.

![Table: Results](https://github.com/ak19qp/ICSME2023/blob/main/Use%20Cases/ls_bug/cs3_table.PNG)

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
Once we have the `esddata.csv` file, which will be in a `comma-separated values (CSV)` format, this file could be opened in MS Excel or LibreOffice Calc to visualize and perform sorting. Once it is loaded in a spread sheet viewing tool, you can sort (in descending order) the output based on `Increase` to find the ranked list of prospective, suspicious and potentially problematic functions that need to be monitored for performance issues. This `esddata` file could also be used with other scripts for performing your own analysis.

## Notes
- If the addresses converted to names show no function names, then it is highly likely that the `ac_add_options --enable-perf` option for mozconfig was not correctly setup during the firefox build.
- Address to name translation might take a long time if perf data was too large. We recommend not running `perf record` for more than 5 seconds for evaluation and review purposes of this artifact.
- Average time to complete ESD as well as address to name translation for roughly 1GB perf script output file is around 1hr. So, it must be kept under 200 MB (by reducing perf record time) for faster review.
