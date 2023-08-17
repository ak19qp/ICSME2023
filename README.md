# PASD: A Performance Analysis Approach Through the Statistical Debugging of Kernel Events

Dynamic performance analysis is vital for optimizing systems and finding performance bottlenecks. Traditional debugging struggles to locate issues in complex software due to hidden performance problems during execution. This artifact uses PASD (Performance Analysis through Statistical Debugging), which analyzes kernel-level trace events without altering application code. It identifies and reports key functions causing performance problems, maintaining normal software operations, and avoiding added complexity. PASD offers deep insights into kernel-level behavior, helping developers pinpoint bottlenecks, enhance efficiency, and improve software quality, user satisfaction, and system stability.

## This repository contains the artifact needed for:

### Data Collection:
- Record kernel events and their relevant call stacks.
- Extract functions from the call stacks.
- Extract relevant data to calculate the wait time of those kernel events and thereby the wait time of the functions themselves.

### Performance Debugging:
- Define threshold to differentiate success and fail runs.
- Find fail, fail observed, success and success ovserved for each unique functions.
- Perform Enhanced Statistical Debugging (ESD) on the functions' data.
- Rank the suspicious/problematic application functions responsible for performance issues based on the ESD data.

## Requirements

### Minimum System Requirements:

Operating System: Ubuntu 18+\
RAM: 8GB\
Storage: 50GB\
Processor: Intel i3 2.5 GHz

### Software Requirements:

Perf (Linux Perf)\
Firefox built with perf options enabled\
Python3\
Pip for Python package installations

## Environment Setup

### Data Collection
For data collection, it must be ensured that `Python3`, `Pip` and `Perf` is installed in the machine. Below are the instructions to set them up for Ubuntu:

1. Python3:
   - First verify if you already have it installed (and the version) by opening a terminal and entering the following command:
     
     `python3 --version`\
     If you get an error message then proceed to the installation method below. Else you are all set with Python3.
     
   - To install Python3, type in the following command in terminal:
     
     `sudo apt update`
     
     `sudo apt install python3`
     
    - Once Python3 is installed, proceed to install pip if it is already not present. You can verify this by typing in `pip` in the terminal.
2. Pip:
   - Open terminal and enter this command:
     
     `sudo apt-get install curl python3 python3-pip`
     
   - Verify installation by typing in `pip` in the terminal.

3. Perf:
   - Ensure if you already have Perf installed by typing in `perf` in the terminal. If you end up with an error message then proceed to the next step.
   - To install Perf, enter the following command in the terminal:
     
     `sudo apt-get install linux-tools-common`
     
   - To verify the installation, type in `perf` again in the terminal.

### Performance Debugging
For performance debugging, the application that needs to be analyzed has to be setup with debugger info/symbols enabled (which can be done if you have access to its binaries), or the application compilation method must support perf options. Without either of them performance debugging becomes difficult as we will not have access to the names of the functions from the call stack data, but rather the hex address values of those functions in the memory at the time of data collection.

1. Setup the application accordingly so that call stack data could be translated to the function names.
2. Download the python script `perf_perser_and_esd.py` from this repository into a folder (put it in the same folder where you would want to store your perf record data).

Some resources to translate addresses to function names: [addr2line](https://manpages.ubuntu.com/manpages/focal/en/man1/alpha-linux-gnu-addr2line.1.html), [nm](https://www.ibm.com/docs/en/zos/2.5.0?topic=scd-nm-display-symbol-table-object-library-executable-files).


## PASD Process

### Data Collection
- Our script currently support system calls, however, it could be easily extended to support various other kernel events. Therefore, for data collection we will show an example to collect system calls and their relevant call stack data with perf below.

- Open a terminal in the directory where you have downloaded the `perf_perser_and_esd.py` python script. We need to find the PID of the application that we want to collect data for which can be done multiple different ways. Here is a simple way to do so:
  
  `pstree -p | grep "[Application/process name goes here]" | head -1`

- Then copy the PID of the parent PID and lets move on to start Perf:
  
  `sudo perf record -g -e 'syscalls:sys_*' -p [PID goes here]`\
  This ensures that you are not recording system-wide data, but only the relevant application data. If you need to collect system-wide data (which I doubt you would), then remote the `-p` argument from the command.

- Once sufficient tasks has been performed and enough data has been collected, then stop the recording by pressing `Ctrl`+`C`. Make sure it gracefully exits so that perf could write all the data and not miss out on writing some events. So, ensure that you do not keep on pressing `Ctrl`+`C` more than once.

- When you have the perf.data file ready in the directory, then execute the following command in the terminal:
  
  `sudo perf script > [give it a file name]`\
  This will convert the perf data into human readable call stack data. For referencing to the next sections, we will name this file as `pcsdata` representing perf call stack data.


### Enhanced Statistical Debugging (ESD)
- For performing ESD, we need the `pcsdata` file, a threshold method (1. Mean+Stdv, 2. Fixed) and the `perf_perser_and_esd.py` python script from this repository.

- For Mean+Stdv threshold method `(threshold_type = 1)`, execute the following command in the terminal:

  `python3 perf_perser_and_esd.py [enter output file name] 1`
  Here the threshold that defines success and fail runs will be decided based on the individual function's mean+stdv of their overall wait time in the sample data.

- For a Fixed threshold method `(threshold_type = 2)`, execute the following command in the terminal:

  `python3 perf_perser_and_esd.py [enter output file name] 2 [enter threshold here in milliseconds]`
  As an example, if 10 was selected as the threshold, then whenever a function experienced a wait time of 10 milliseconds or higher in a system call, those runs would be considered as a fail run and vice versa.
  
- For referencing to the next section, we will be using `esddata` as the output file name.

### Analysis
Once we have the `esddata` file, which will be in a `comma-separated values (CSV)` format, this file could be opened in MS Excel or LibreOffice Calc to visualize and perform sorting. Once it is loaded in a spread sheet viewing tool, you can sort the output based on `Increase` (in descending order) to find the ranked list of prospective, suspicious and potentially problematic functions that need to be monitored for performance issues.


## Use Cases

We have provided three of the use case's data mentioned in the paper in this repository. In order to test out the EDS script `(perf_perser_and_esd.py)` the raw perf data along with each of the use case's instructions have been provided in detail in their individual folder's README file.

[Use Case 1](https://github.com/ak19qp/ICSME2023/tree/main/Use%20Cases/Use%20Case%201)
[Use Case 2](https://github.com/ak19qp/ICSME2023/tree/main/Use%20Cases/Use%20Case%202)
[Use Case 3](https://github.com/ak19qp/ICSME2023/tree/main/Use%20Cases/Use%20Case%203)

## Authors:
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
