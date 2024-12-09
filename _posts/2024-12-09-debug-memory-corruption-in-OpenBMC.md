This note explains few methods to debug memory corruptions in the OpenBMC application. We will be using mboxd application as an example.

## Find stack trace using GDB command
OpenBMC generate core dump file on memory faults like segmentation fault.
