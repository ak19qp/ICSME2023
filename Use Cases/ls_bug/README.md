# Use Case: "ls" Slow Performance in Large Directories

In this use case, we investigated a performance bug that arises from the usage of the 'ls' command in Linux, which is a component of the GNU Core Utilities package. The 'ls' command is a commonly used command-line utility in Unix-like operating systems. Its purpose is to display a list of files and directories within a specified directory or the current working directory. When executed without any arguments, 'ls' will present the contents of the current directory, providing a concise listing that includes the names of files and directories.

Reports on various online discussion platforms: [serverfault](https://serverfault.com/questions/316951/why-might-ls-color-always-be-slow-for-a-small-directory), [superuser](https://superuser.com/questions/1345268/ls-command-very-slow) as well as RedHat BugZilla reports: [1](https://bugzilla.redhat.com/show_bug.cgi?id=1290036), [2](https://bugzilla.redhat.com/show_bug.cgi?id=467508), have indicated that 'ls' exhibits significant slowness when dealing with directories containing a large number of files and folders. Furthermore, it is also slow when handling a large number of top-level entries. This issue specifically arises when color coding is enabled for the 'ls' command.

## TL;DR Note:
If you just want to evaluate our `perf_perser_and_esd.py` script without going through the process of building ls and recording the perf data, then please download the data folder from this use case folder which already contains the perf data for this use case and run the following command in the terminal from within the data folder directory:

`tar -xvf case_study_3.tar.gz`

Once the file has been unzipped the execute the following command from the `perf_perser_and_esd.py` file's directory:

### For Mean+Stdev as threshold:
`python3 perf_perser_and_esd.py case_study_3.txt esddata.csv 1`

### For fixed threshold:
`python3 perf_perser_and_esd.py case_study_3.txt esddata.csv 2 [enter threshold in milliseconds]`

## Use Case Setup & Data Collection

For this use case, we began by installing the libc6-dbg package using the command 'sudo apt-get install libc6-dbg'. This ensured that the libc compiler had debugger information available for the compiled program. We also installed the build-essential package in the same manner to facilitate the building process. Subsequently, we downloaded the source code of the GNU Core Utilities package and proceeded to extract and compile it with debugger information enabled.

Once the necessary preparations were complete, we utilized a bash script to generate a substantial number of random files and folders, along with numerous top-level directories. We employed the 'Perf' tool to execute the 'ls' command and record call stacks whenever system calls were made by 'ls'. The command used for this purpose was `perf record -g -e 'syscalls:sys\_*' /[coreutils directory]/src/ls --color=always /[directory to inquire]`. Additionally, we also repeated the call stack recording process with Perf while disabling the color parameter of 'ls' and collect 276,412 call stack data. To obtain a comprehensive dataset for statistical analysis, we executed 'ls' through Perf multiple times under various loads and stress conditions.

## Analysis & Discussion

In order to analyze we first have to convert the perf data into readable data using the command:

`perf script > case_study_3.txt`

Next we executed the script using the command:

`python3 perf_perser_and_esd.py case_study_3.txt esddata.csv 1`

The `esddata.csv` file is then opened with MS Excel and the unique functions were then sorted based on their 'Increase' value and ranked accordingly. Table III highlights the culprit functions identified from the sorted ranked list. The results of this use case emphasize the significance of not disregarding functions with negative 'Increase' values. While `__GI___statfs` is not directly part of the ls class's code, it is the top-ranked function in the list. On the other hand, `do_lstat` is a direct function from the ls class and exhibits a negative 'Increase' value. It is worth noting that ls executes statfs through `do_lstat`, and the issue lies not in statfs itself, but rather in how `do_lstat` calls statfs.

![Table: Results](https://github.com/ak19qp/ICSME2023/blob/main/Use%20Cases/ls_bug/cs3_table.PNG)

Upon analyzing the results from the ranked list of functions, we can observe that `__GI___statfs` appears as the top candidate. This finding is consistent with the information provided in the answer from [superuser](https://superuser.com/questions/1345268/ls-command-very-slow), which states that individual statfs calls are made to gather information about the type of file, permissions, and file capabilities for the purpose of setting colors accordingly. This aligns with our observations from the ranked list.

Furthermore, upon conducting further analysis, we discovered two significant functions: `do_lstat` and `print_color_indicator`, as indicated in Table III. Upon reviewing the ls source code, we found that `do_lstat` is the function responsible for calling the `statfs` function and determining whether color codings are required, utilizing the `print_color_indicator` function. When color coding is enabled, `do_lstat` needs to retrieve additional metadata related to each files and folders, which ultimately contributes to the performance problem. This discovery identifies the actual cause of the performance issue.

The findings from [serverfault](https://serverfault.com/questions/316951/why-might-ls-color-always-be-slow-for-a-small-directory) and [bugzilla](https://bugzilla.redhat.com/show_bug.cgi?id=1290036) further validates our method, as they also mentioned that disabling color coding for 'ls' significantly improves its performance which aligns with our method's findings.


