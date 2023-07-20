Emulating the RP2040 with Ghidra
================================

Ghidra (an open-source SRE framework written by the NSA) provides a feature-full emulator which works on all the architectures Ghidra supports. These notes will show how to emulate a basic RP2040 ELF.

Prerequisites
-------------

In order to follow these notes, both the latest version of Ghidra and the RP2040 SDK must be installed. Resources on how to do this can be found here:

* `Install Ghidra <https://ghidra-sre.org/InstallationGuide.html>`_
* `Install RP2040 SDK <https://datasheets.raspberrypi.com/pico/getting-started-with-pico.pdf>`_

Building Hello World
--------------------

The first binary I attempted to emulate was a basic Hello World script consisting of the following main function:

.. code-block:: C

    #include <stdio.h>
    #include "pico/stdlib.h"

    int main() {
        stdio_init_all();
        for(int i = 0; i < 10; i++) {
            printf("Hello, world!\n");
            sleep_ms(1000);
        }
    }

I configures the CMakeLists.txt file to output only on the USB interface using the following:

.. code-block:: cmake

    cmake_minimum_required(VERSION 3.13)

    include(pico_sdk_import.cmake)

    project(hello_serial C CXX ASM)
    set(CMAKE_C_STANDARD 11)
    set(CMAKE_CXX_STANDARD 17)

    pico_sdk_init()

    if (TARGET tinyusb_device)
        add_executable(pico_hello
                hello_serial.c
                )

        # pull in common dependencies
        target_link_libraries(pico_hello pico_stdlib)

        # enable usb output, disable uart output
        pico_enable_stdio_usb(pico_hello 1)
        pico_enable_stdio_uart(pico_hello 0)

        # create map/bin/hex/uf2 file etc.
        pico_add_extra_outputs(pico_hello)

    elseif(PICO_ON_DEVICE)
        message(WARNING "not building hello_usb because TinyUSB submodule is not initialized in the SDK")
    endif()

Emulating Main (Attempt 1)
--------------------------

Once the code had been compiled, I started a new Ghidra project and imported the ELF file. I then ran Ghidra's automated analysis tools and had a quick look through the de-compiled code. 

Initially, I attempted to emulate main by running the following Ghidra script:

.. code-block:: python

    from ghidra.app.emulator import EmulatorHelper
    from ghidra.program.model.symbol import SymbolUtilities


    def getAddress(offset):
        return currentProgram.getAddressFactory().getDefaultAddressSpace().getAddress(offset)

    def getSymbolAddress(symbolName):
        symbol = SymbolUtilities.getLabelOrFunctionSymbol(currentProgram, symbolName, None)
        if (symbol != None):
            return symbol.getAddress()
        else:
            raise("Failed to locate label: {}".format(symbolName))

    def getProgramRegisterList(currentProgram):
        pc = currentProgram.getProgramContext()
        return pc.registers

    def main():
        CONTROLLED_RETURN_OFFSET = 0

        mainFunctionEntry = getSymbolAddress("main")
        emuHelper = EmulatorHelper(currentProgram)
        controlledReturnAddr = getAddress(CONTROLLED_RETURN_OFFSET)

        # Tell emulator to run the main() function
        mainFunctionEntryLong = int("0x{}".format(mainFunctionEntry), 16)
        emuHelper.writeRegister(emuHelper.getPCRegister(), mainFunctionEntryLong)
                    
        print("Emulation starting at 0x{}".format(mainFunctionEntry))
        while monitor.isCancelled() is False:
            
            executionAddress = emuHelper.getExecutionAddress()  
            if (executionAddress == controlledReturnAddr):
                print("Emulation complete.")
                return

            print("Address: 0x{} ({})".format(executionAddress, getInstructionAt(executionAddress)))

            # single step emulation
            success = emuHelper.step(monitor)
            if (success == False):
                lastError = emuHelper.getLastError()
                printerr("Emulation Error: '{}'".format(lastError))
                return

        emuHelper.dispose()

    main()

This gave the following output:

.. code-block::

    Emulation starting at 0x1000035c
    Address: 0x1000035c (push {r4,lr})
    Address: 0x1000035e (bl 0x100045ec)
    Address: 0x100045ec (push {r4,lr})
    Address: 0x100045ee (bl 0x10004898)
    Address: 0x10004898 (push {r4,r5,r6,lr})
    Address: 0x1000489a (sub sp,#0x10)
    Address: 0x1000489c (movs r3,#0xd0)
    Address: 0x1000489e (lsls r3,r3,#0x18)
    Address: 0x100048a0 (ldrb r2,[r3,#0x0])
    Address: 0x100048a2 (ldr r3,[0x10004958])
    Address: 0x100048a4 (strb r2,[r3,#0x0])
    Address: 0x100048a6 (ldr r3,[0x1000495c])
    Address: 0x100048a8 (ldrb r3,[r3,#0x0])
    Address: 0x100048aa (bl 0x100069cc)
    Address: 0x100069cc (push {r4,lr})
    Address: 0x100069ce (movs r0,#0x0)
    Address: 0x100069d0 (bl 0x100054c8)
    Address: 0x100054c8 (push {r4,r5,r6,lr})
    Address: 0x100054ca (movs r5,r0)
    Address: 0x100054cc (ldr r3,[0x1000556c])
    Address: 0x100054ce (ldrb r0,[r3,#0x0])
    Address: 0x100054d0 (cmp r0,#0x0)
    Address: 0x100054d2 (beq 0x100054d6)
    Address: 0x100054d6 (movs r2,#0x53)
    Address: 0x100054d8 (movs r1,#0x0)
    Address: 0x100054da (ldr r0,[0x10005570])
    Address: 0x100054dc (bl 0x10004340)
    Address: 0x10004340 (ldr r3,[0x10004348])
    Address: 0x10004342 (ldr r3,[r3,#0x0])
    Address: 0x10004344 (bx r3)
    Address: 0x0000534c (None)
    emulate_rp2040.py> Emulation Error: 'Instruction decode failed (Bytes do not form a legal instruction.), PC=0000534c'

By analysing the decompiled binary I could see that the following function calls were made before the error happened: main() > stdio_init_all() > stdio_usb_init() > tusb_init() > tud_init() > __wrap_memset(void)

The __wrap_memset(void) function decompiles to this:

.. code-block:: C

    _Bool tud_init(uint8_t rhport)

    {
        ...
        __wrap_memset(&_usbd_dev,0,0x53);
        ...
    }

    void __wrap_memset(void)

    {
                        /* WARNING: Treating indirect jump as call */
        (*(code *)0x534d)();
        return;
    }

The emulator is unable to find valid instructions at address 0x534d and therefore throws an error. It looks like the __wrap_memset() function should be setting 83 bytes of memory to 0 at the address of _usb_dev, however this does not appear to be what the __wrap_memset is actually doing. Looking at the Ghidra decompilation, we can see that _usbd_dev has an address -f 0x2000097C.

Creating A Memory Map 
---------------------

One way to improve the emulation accuracy is to give the emulator a representative map of the memory as present on the device. To do this, follow the steps in the link below to get a GDB link running using a pico debug probe:

* `Pico Debug Probe <https://www.raspberrypi.com/documentation/microcontrollers/debug-probe.html>`_

Next, run the following commands to dump the contents of the relevant sections of memory to binary files.

.. code-block::

    (gdb) dump binary memory pico_rom.bin 0x00000000 0x00004000
    (gdb) dump binary memory pico_xip.bin 0x10000000 0x11000000
    (gdb) dump binary memory pico_sram_0_5.bin 0x20000000 0x20042000
    (gdb) dump binary memory pico_peripherals.bin 0x40000000 0x40080000
    (gdb) dump binary memory pico_sram_alias.bin 0x21000000 0x21040000
    (gdb) dump binary memory pico_dma.bin 0x50000000 0x50100000
    (gdb) dump binary memory pico_usb.bin 0x50100000 0x50200000
    (gdb) dump binary memory pico_ahb_lite.bin 0x50200000 0x50500000
    (gdb) dump binary memory pico_sio.bin 0xd0000000 0xd000017c
    (gdb) dump binary memory pico_ppb.bin 0xe0000000 0xe0100000

These can then be loaded into memory before the Ghidra script starts emulation. Add the following before the for loop in the main function of the emulation script:

.. code-block:: python
    
    dir = str(askDirectory("Select Memory Directory", "OK"))+"/"
    
    # Map memory
    ROM  = read_memory(dir+"pico_rom.bin")
    XIP  = read_memory(dir+"pico_xip.bin")
    SRAM = read_memory(dir+"pico_sram_0_5.bin")
    SRAM_AL = read_memory(dir+"pico_sram_alias.bin")
    AHB_LITE = read_memory(dir+"pico_ahb_lite.bin")
    DMA = read_memory(dir+"pico_dma.bin")
    PERIPH = read_memory(dir+"pico_peripherals.bin")
    PPB = read_memory(dir+"pico_ppb.bin")
    SIO = read_memory(dir+"pico_sio.bin")
    USB = read_memory(dir+"pico_usb.bin")

    currentProgramMemory = emuHelper.readMemory(getAddress(0x10000000), 0x01000000)

    
    # Write Memory
    emuHelper.writeMemory(getAddress(0x00000000), ROM)
    emuHelper.writeMemory(getAddress(0x12000000), currentProgramMemory)
    emuHelper.writeMemory(getAddress(0x13000000), currentProgramMemory)
    emuHelper.writeMemory(getAddress(0x14000000), currentProgramMemory)
    emuHelper.writeMemory(getAddress(0x18000000), currentProgramMemory)
    emuHelper.writeMemory(getAddress(0x20000000), SRAM)
    emuHelper.writeMemory(getAddress(0x21000000), SRAM_AL)
    emuHelper.writeMemory(getAddress(0x40000000), PERIPH)
    emuHelper.writeMemory(getAddress(0x50000000), DMA)
    emuHelper.writeMemory(getAddress(0x50100000), USB)
    emuHelper.writeMemory(getAddress(0x50200000), AHB_LITE)

This will allow the emulation to get much futher, however the error "Unimplemented CALLOTHER pcodeop (DataMemoryBarrier), PC=10001d40" will be encountered once the emulator tries to execute the dmb (DataMemoryBarrier) instruction. 

Skipping Sections of Code 
-------------------------

To get around the error we can tell Ghidra to skip certain parts of the executable. Append the following before the for loop in the emulation script:

.. code-block:: python

    # Set up list of known-error addresses
    err_addr  = [0x10001d40, 0x10000384, 0x100003d4]
    skip_addr = [0x10001d44, 0x100003c8, 0x100003d8]

And alter the loop to look like this:

.. code-block:: python

    print("Emulation starting at 0x{}".format(mainFunctionEntry))
    while monitor.isCancelled() is False:
        
        executionAddress = emuHelper.getExecutionAddress()  
        if (executionAddress == controlledReturnAddr):
            print("Emulation complete.")
            return

        print("Address: 0x{} ({})".format(executionAddress, getInstructionAt(executionAddress)))

        # single step emulation
        success = emuHelper.step(monitor)
        if (success == False):
            lastError = emuHelper.getLastError()
            if (executionAddress.offset in err_addr):
                emuHelper.writeRegister(emuHelper.getPCRegister(), skip_addr[err_addr.index(executionAddress.offset)])
                print("Warning: '{}'".format(lastError))
            else:
                printerr("Emulation Error: '{}'".format(lastError))
                return

    emuHelper.dispose()

Whenever the emulator tries to execute an instruction storred in err_addr it will print a warning and skip to the instruction pointed to by skip_addr. 