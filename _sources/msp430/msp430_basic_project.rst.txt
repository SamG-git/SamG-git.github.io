Creating a Basic Project for MSP430
===================================

Building a C Project
--------------------

This project uses the MSP430FR5969 Launchpad as a reference platform - some aspects will be different if another platform is used.

1. Create a new directory for the project.
2. Inside this directory, create a file called blink.c with the following contents:

.. code-block:: C 
    
    #include <msp430.h>				

    int main(void) {
        WDTCTL = WDTPW | WDTHOLD;		// Stop watchdog timer
        P1DIR |= 0x01;					// Set P1.0 to output direction

        for(;;) {
            volatile unsigned int i;	// volatile to prevent optimization

            P1OUT ^= 0x01;				// Toggle P1.0 using exclusive-OR

            i = 10000;					// SW Delay
            do i--;
            while(i != 0);
        }
        
        return 0;
    }

3. Create a Makefile with the following contents:

.. code-block:: make

    OBJECTS=blink.o
    GCC_DIR = /opt/TI/msp430-gcc/bin
    SUPPORT_FILE_DIRECTORY = /opt/TI/msp430-gcc/include

    DEVICE = MSP430FR5969
    CC = $(GCC_DIR)/msp430-elf-gcc
    GDB = $(GCC_DIR)/msp430-elf-gdb
    CFLAGS = -I $(SUPPORT_FILE_DIRECTORY) -mmcu=$(DEVICE) -mlarge -mcode-region=either -mdata-region=lower -Og -Wall -gLFLAGS = -L $(SUPPORT_FILE_DIRECTORY) -T $(DEVICE).ld
    LFLAGS = -L $(SUPPORT_FILE_DIRECTORY) -Wl,-Map,$(MAP),--gc-sections

    all: ${OBJECTS}
        $(CC) $(CFLAGS) $(LFLAGS) $? -o $(DEVICE).out

    debug: all
        $(GDB) $(DEVICE).out

1. Run the `make` command to build the binary.

Loading the Binary
------------------

1. Open a terminal window and run ``LD_LIBRARY_PATH=/opt/TI/msp430_toolchain/lib /opt/TI/msp430_toolchain/bin/mspdebug tilib``
2. 2. In a seperate terminal window, run ``make debug``
3. In the gdb menu, run ``target remote :2000``
4. Run ``load`` to load the binary onto the MSP430
5. Run ``continue`` to start the app



