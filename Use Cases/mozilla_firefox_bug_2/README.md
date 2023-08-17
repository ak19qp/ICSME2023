# Use Case: Firefox Tripadvisor.ca CPU Exhaustion Bug

In this use case, we analyze a performance bug reported in [Bugzilla](https://bugzilla.mozilla.org/show\_bug.cgi?id=1565019) in 2019, which pertains to an extraordinary elevation in CPU usage while loading the website [tripadvisor.ca](https://tripadvisor.ca) in the Firefox browser. The report indicates that the main thread monopolizes CPU resources for a protracted duration. In our experimental environment, designed to replicate this bug, we experienced system-wide crashes on multiple occasions, attributable to resource exhaustion.

## TL;DR Note:

If you just want to evaluate our `perf_perser_and_esd.py` script without going through the process of building Firefox and recording the perf data, then please download the data folder from this use case folder which already contains the perf data for this use case and run the following command in the terminal from within the data folder directory:

`cat case_study_2_* > case_study_2.tar.gz`

`tar -xvf case_study_2.tar.gz`

Once the file has been unzipped the execute the following command from the `perf_perser_and_esd.py` file's directory:

### For Mean+Stdev as threshold:
`python3 perf_perser_and_esd.py case_study_2.txt esddata.csv 1`

### For fixed threshold:
`python3 perf_perser_and_esd.py case_study_2.txt esddata.csv 2 [enter threshold in milliseconds]`


## Case Study Setup & Data collection

In an approach similar to the previous Firefox [use case](https://github.com/ak19qp/ICSME2023/tree/main/Use%20Cases/mozilla_firefox_bug_1), the experimental setup entailed compiling Firefox from the source with the 'Perf' tag enabled. 'Perf' was employed to capture call stack data associated with system call events hooked to the Firefox PID. The website tripadvisor.ca was loaded recurrently to accumulate sufficient call stack data.

The call stack data collected in this study were analyzed to identify unique function names and their associated wait times for different system calls. The task of the experiment was carried out multiple times over 110 seconds. A total amount of 1.23 GB of data containing 398297 call stacks was collected, sufficient to deliver meaningful insights into the wait time of unique functions during system calls.


## Analysis & Discussion

Implementing statistical debugging on the collated data for individual functions, and then ranking them by the 'Increase' value followed by a cut-off of 15% of the list, reveals three significant function calls across multiple addresses in the list. This implies their execution by distinct threads. These functions are `gethostbyaddr_r@@GLIBC_2.2.5`, `pthread_cond_signal@@GLIBC_2.3.2`, and `getifaddrs_internal`.

The function `gethostbyaddr_r` is entrusted with reverse Domain Name System (DNS) resolution and is a thread-safe variant of the gethostbyaddr function. The function `pthread_cond_signal`, an integral part of the POSIX threads (pthreads) library, provides a parallel programming interface grounded in threads for Unix-like operating systems. This function is employed for thread synchronization, facilitating threads to await the fulfillment of a specific condition. Lastly, the `getifaddrs_internal` function retrieves data regarding the network interfaces available on the system, such as their IP addresses, network masks, and interface names. It allocates memory for a linked list of struct ifaddrs structures, each encapsulating information about an interface.

When the functions are sorted based on the highest standard deviation and mean, `std::panicking::try` and `std::panic::catch_unwind` appear prominently on the list. Along with these, multiple function calls from the class `mozilla::net::nsSocketTransportService`, responsible for network layer communications, are also found towards the top of the list. This analysis provides insights into the functions potentially contributing to the observed performance issues.


![Table: Results](https://github.com/ak19qp/ICSME2023/blob/main/Use%20Cases/mozilla_firefox_bug_2/cs2_table.PNG)


Upon meticulous examination of our results, it becomes apparent that the performance issue is intrinsically tied to network layer communication. The website tripadvisor.ca operates with a substantial assortment of interconnected websites and URLs, which it references to populate its content.

Table II highlights the culprit functions from the sorted ranked list of candidate functions. Our analysis, derived from statistical debugging, indicates that the parent thread commissions numerous child threads to retrieve various data from different URLs during the loading of tripadvisor.ca. Given that `pthread_cond_signal` exhibits an extended system call wait time concurrent with the network-related functions, it can be inferred that a failure occurs at the code level. This inference is supported by the activation of `std::panicking::try` and `std::panic::catch_unwind`.

We postulate that, amidst this failure, the catch block perpetually generates new threads to reattempt the same task. It is during this period that the CPU usage escalates dramatically. This relentless cycle of creating and terminating threads to retry the same task persists for an extended duration causing performance bottleneck and CPU resource exhaustion.

Our analysis gains further validation from the bug report, which attributes the occurrence of this issue to a dependency on off-main-thread (OMT) networking. The report further indicates an extensive array of processes transpiring on the main thread, with a majority triggered by initiating network requests. This correlation corroborates the findings from our method, bolstering our analysis that network-related functions play a significant role in this performance issue.
