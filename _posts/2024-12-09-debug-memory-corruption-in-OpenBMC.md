This note explains few methods to debug memory corruptions in the OpenBMC application. We will be using mboxd application as an example.

1. Using GDB command
OpenBMC generate core dump file on memory faults like segmentation fault. Find the fault location using gdb. There are two ways to attach gdb.
- Use method from Andrew's blog to install gdb and source on the openbmc system and then debug on system itself. [Core debug on openbmc system](https://amboar.github.io/notes/2022/01/13/openbmc-development-workflow.html)
- Copy BMC dump file out of openbmc system. Extract the core file and use [OpenBMC Build Scripts](https://github.com/ibm-openbmc/openbmc-build-scripts)
