This note goes over few methods to debug ***memory corruption*** in OpenBMC application. The ***mboxd*** application is used as an example.

1. First method is to use GDB command to find fault location by inspecting stack traces. The OpenBMC generates core file on memory faults like segmentation fault. There are two ways to attach gdb to the core file.
	- You can debug on openBMC machine by installing gdb and debug source. [Andrew's blog](https://amboar.github.io/notes/2022/01/13/openbmc-development-workflow.html) explains this method in detail.
	- You can also copy BMC dump file out of OpenBMC machine. Extract the core file from dump and use [OpenBMC Build Scripts tools](https://github.com/ibm-openbmc/openbmc-build-scripts) to attach the GDB.

2.  Second method is to use memcheck tool in [Valgrind](https://valgrind.org/docs/manual/mc-manual.html) which detects memory errors in the application. It requires running the application under valgrind tool.
	- Update [Yocto recipe](https://github.com/openbmc/openbmc/blob/master/meta-ibm/recipes-phosphor/mboxd/mboxd_%25.bbappend) to add debugging information. Then compile and update OpenBMC machine.
		```
		+FULL_OPTIMIZATION:append=" -Og "
		+TARGET_CPPFLAGS:append=" -O0 "
		```
	- Edit service file for mboxd to append valgrind command as shown below.  
		- On openBMC, run command `systemctl edit --full mboxd.service` to edit the file. Then change ExecStart line as show below.
		```
		+ExecStart=/usr/bin/valgrind --tool=memcheck --leak-check=full --keep-debuginfo=yes --xtree-memory=full --xtree-memory-file=/var/log/mboxd-xtree.log --errors-for-leak-kinds=definite,possible --undef-value-errors=yes --keep-stacktraces=alloc-and-free --show-mismatched-frees=yes --log-file=/var/log/mboxd.log /usr/bin/env mboxd --flash {FLASH_SIZE} --window-size 1M --window-num {WINDOW_NUM} $MBOXD_ARGS
		```
       - Restart the mboxd service using command: ***systemctl restart mboxd.service***
	- Run the test to reproduce memory issue. Check log file for traces and errors.
	- Once testing is done undo all service file updates by removing file  ***/etc/systemd/system/mboxd.service*** created by edit.
    - ***Note:*** You can also update [Yocto mboxd recipe file](https://github.com/openbmc/openbmc/blob/master/meta-phosphor/recipes-phosphor/mboxd/mboxd/mboxd.service) instead of updating service file on the OpenBMC machine. This requires you to compile the BMC image and update the system.

3. Third method is to use [GCC address sanitizer tool](https://github.com/google/sanitizers/wiki/addresssanitizer) developed by google to detect memory access errors such as use-after-free, memory leaks. Compile OpenBMC code with following recipe changes.
	- There is already a recipe in Yocto for gcc address sanitizer. So update [Yocto mboxd recipe](https://github.com/openbmc/openbmc/blob/master/meta-phosphor/recipes-phosphor/mboxd/mboxd_git.bb) to pull [gcc-sanitizer](https://github.com/openbmc/openbmc/blob/master/poky/meta/recipes-devtools/gcc/gcc-sanitizers_14.1.bb) by adding the dependency.
		```
		DEPENDS += "gcc-sanitizers"
		```
	- Update [Yocto phosphor images recipe](https://github.com/openbmc/openbmc/blob/master/meta-ibm/recipes-phosphor/images/obmc-phosphor-image.bbappend) to add extra packages like mboxd source/debug, libasan runtime library for Address Sanitizer and libubsan library for Undefined Behavior Sanitizer
		```
		+OBMC_IMAGE_EXTRA_INSTALL:append:p10bmc = " mboxd-src mboxd-dbg libasan libubsan"
		```
	- Update [Yocto phosphor images recipe](https://github.com/openbmc/openbmc/blob/master/meta-ibm/recipes-phosphor/images/obmc-phosphor-image.bbappend) to add compiler debug and address sanitizer flags to turn on memory check. Refer [GCC documentation](https://gcc.gnu.org/onlinedocs/gcc-6.4.0/gcc/Link-Options.html) for more options.
		```
		+FULL_OPTIMIZATION:append=" -Og -fsanitize=address,undefined "
		+TARGET_CPPFLAGS:append=" -O0 "
		```
	- ***Note***: You can replicate this mboxd example for any application in Yocto.
