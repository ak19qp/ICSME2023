## Installation Instructions

1. Perf:
- Open terminal and enter this command:\
  sudo apt install linux-tools-common
- Verify installation by typing in "perf" in the terminal.

2. Firefox:
- Install Python with the following command in the terminal:\
  sudo apt-get install curl python3 python3-pip
  
- Install Mercurial with the following command in the terminal:\
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
