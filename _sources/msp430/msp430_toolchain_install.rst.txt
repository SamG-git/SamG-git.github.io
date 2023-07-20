Installing an MSP430 Toolchain
==============================

GNU Compiler Collection & GDB
-----------------------------

To install GCC for the MSP 430 , go to the Downloads section of this page and grab either the gcc source and the header and support files or the precompiled binaries:

- `GCC for MSP430 <https://www.ti.com/tool/MSP430-GCC-OPENSOURCE#downloads>`_

If you've grabbed the precompiled binaries ruin the install script with sudo privileges. 

Otherwise create a folder for the source-code to reside (e.g. /opt/TI/) and extract the source. Once extracted, change to the source-full directory and run ``bash README-build.sh``. This will download all the required dependencies from the internet and compiling them, leaving the binaries in the ``install`` folder. Finally, make a folder in the /opt/TI/ directory called msp430_toolchain and copy the contents of the ``install/usr/local`` directory into it.

NOTE: Compiling GCC from source currently does not produce a working C compiler, however the assembler seems to work fine.

MSPDebug
--------

MSPDebug replaces msp430-gdbproxy used for creating GDB sessions with the MSP430. To install, clone the following GitHub repository to the /opt/TI directory created earlier:

- `MSPDebug <https://github.com/dlbeer/mspdebug>`_ 

Enter the new directory and run the ``make`` command. Then, copy the ``mspdebug`` binary to /opt/TI/msp430_toolchain/bin.

Building libmsp430.so 
---------------------

MSPDebug requires the libmsp430.so file to be built before it will run properly with Launchpad products. To build this:

1. Download the MSPdebug stack source code from here: `MSP Debug Stack <https://www.ti.com/tool/MSPDS>`_ 

2. Clone the hidapi source code from here: `HID API <https://github.com/libusb/hidapi>`_ 
3. Enter the hidapi directory and build using these commands:

.. code-block:: bash

    ./bootstrap
    ./configure CFLAGS='-g -O2 -fPIC'
    make

4. unzip the MSPDebugStack.... file using the command ``unzip <filename> -d MSPDebugStack``

5. Copy the hidapi.h file to MSPDebugStack/ThirdParty/include
6. Copy hid.o to MSPDebugStack/ThirdParty/lib64 
7. Edit the Makefile by changing ``HIDOBJ := $(LIBTHIRD)/hid-libusb.o`` to ``HIDOBJ := $(LIBTHIRD)/hid.o``
8. Build using ``make``
9. Copy the libmsp430.so file to /opt/TI/msp430_toolchain/lib/



