Coding & Debugging RP2040 using Eclipse
=======================================

For the following tutorial to work, ensure that Eclipse is installed for Embedded C/C++ systems.

Creating the Project Structure
------------------------------

1. Create a new directory for the project to live in, with a ``build``` and ``src`` subdirectory.
2. In the project's root directory, create a file called ``CMakeLists.txt`` with the following contents:

.. code-block::

    cmake_minimum_required(VERSION 3.12)

    # Change your executable name to something creative!
    set(NAME pico_asm) # <-- Name your project/executable here!

    include(pico_sdk_import.cmake)

    project(pico_asm
            LANGUAGES C CXX ASM
            DESCRIPTION "Basic RP2040 Assembly Application")
    set(CMAKE_C_STANDARD 11)
    set(CMAKE_CXX_STANDARD 17)

    # Initialize the SDK
    pico_sdk_init()

    # Add your source files
    add_subdirectory("src/")

3. In the project's root directory, create a file called ``pico_sdk_import.cmake`` with the following contents:

.. code-block::
    
    # This is a copy of <PICO_SDK_PATH>/external/pico_sdk_import.cmake

    # This can be dropped into an external project to help locate this SDK
    # It should be include()ed prior to project()

    if (DEFINED ENV{PICO_SDK_PATH} AND (NOT PICO_SDK_PATH))
        set(PICO_SDK_PATH $ENV{PICO_SDK_PATH})
        message("Using PICO_SDK_PATH from environment ('${PICO_SDK_PATH}')")
    endif ()

    if (DEFINED ENV{PICO_SDK_FETCH_FROM_GIT} AND (NOT PICO_SDK_FETCH_FROM_GIT))
        set(PICO_SDK_FETCH_FROM_GIT $ENV{PICO_SDK_FETCH_FROM_GIT})
        message("Using PICO_SDK_FETCH_FROM_GIT from environment ('${PICO_SDK_FETCH_FROM_GIT}')")
    endif ()

    if (DEFINED ENV{PICO_SDK_FETCH_FROM_GIT_PATH} AND (NOT PICO_SDK_FETCH_FROM_GIT_PATH))
        set(PICO_SDK_FETCH_FROM_GIT_PATH $ENV{PICO_SDK_FETCH_FROM_GIT_PATH})
        message("Using PICO_SDK_FETCH_FROM_GIT_PATH from environment ('${PICO_SDK_FETCH_FROM_GIT_PATH}')")
    endif ()

    set(PICO_SDK_PATH "${PICO_SDK_PATH}" CACHE PATH "Path to the PICO SDK")
    set(PICO_SDK_FETCH_FROM_GIT "${PICO_SDK_FETCH_FROM_GIT}" CACHE BOOL "Set to ON to fetch copy of PICO SDK from git if not otherwise locatable")
    set(PICO_SDK_FETCH_FROM_GIT_PATH "${PICO_SDK_FETCH_FROM_GIT_PATH}" CACHE FILEPATH "location to download SDK")

    if (NOT PICO_SDK_PATH)
        if (PICO_SDK_FETCH_FROM_GIT)
            include(FetchContent)
            set(FETCHCONTENT_BASE_DIR_SAVE ${FETCHCONTENT_BASE_DIR})
            if (PICO_SDK_FETCH_FROM_GIT_PATH)
                get_filename_component(FETCHCONTENT_BASE_DIR "${PICO_SDK_FETCH_FROM_GIT_PATH}" REALPATH BASE_DIR "${CMAKE_SOURCE_DIR}")
            endif ()
            FetchContent_Declare(
                    pico_sdk
                    GIT_REPOSITORY https://github.com/raspberrypi/pico-sdk
                    GIT_TAG master
            )
            if (NOT pico_sdk)
                message("Downloading PICO SDK")
                FetchContent_Populate(pico_sdk)
                set(PICO_SDK_PATH ${pico_sdk_SOURCE_DIR})
            endif ()
            set(FETCHCONTENT_BASE_DIR ${FETCHCONTENT_BASE_DIR_SAVE})
        else ()
            message(FATAL_ERROR
                    "PICO SDK location was not specified. Please set PICO_SDK_PATH or set PICO_SDK_FETCH_FROM_GIT to on to fetch from git."
                    )
        endif ()
    endif ()

    get_filename_component(PICO_SDK_PATH "${PICO_SDK_PATH}" REALPATH BASE_DIR "${CMAKE_BINARY_DIR}")
    if (NOT EXISTS ${PICO_SDK_PATH})
        message(FATAL_ERROR "Directory '${PICO_SDK_PATH}' not found")
    endif ()

    set(PICO_SDK_INIT_CMAKE_FILE ${PICO_SDK_PATH}/pico_sdk_init.cmake)
    if (NOT EXISTS ${PICO_SDK_INIT_CMAKE_FILE})
        message(FATAL_ERROR "Directory '${PICO_SDK_PATH}' does not appear to contain the PICO SDK")
    endif ()

    set(PICO_SDK_PATH ${PICO_SDK_PATH} CACHE PATH "Path to the PICO SDK" FORCE)

    include(${PICO_SDK_INIT_CMAKE_FILE})

4. In the ``src`` directory, create a file called ``CMakeLists.txt`` with the following contents:

.. code-block::

    cmake_minimum_required(VERSION 3.14)

    # Include app source code file(s)
    add_executable(${NAME}
        ${CMAKE_SOURCE_DIR}/src/main.S
            ${CMAKE_SOURCE_DIR}/src/sdk_inlines.c
    )

    # Link to built libraries
    target_link_libraries(${NAME} LINK_PUBLIC
                        pico_stdlib
                        hardware_gpio
    )

    # Enable/disable STDIO via USB and UART
    pico_enable_stdio_usb(${NAME} 1)
    pico_enable_stdio_uart(${NAME} 1)

    # Enable extra build products
    pico_add_extra_outputs(${NAME})

5. Populate the source files in the ``src`` directory, altering the associated file names in the ``CMakeLists.txt`` file.

6. Change into the ``build`` directory and run the command ``cmake -G"Eclipse CDT4 - Unix Makefiles" -DCMAKE_BUILD_TYPE=Debug .``.

Configuring the Project in Eclipse
----------------------------------

1. In Eclipse, right click on the Project Explorer section and select New -> Project. This will open the new project wizard. 

.. image:: images/EclipseProjectWizard.png
    :width: 400
    :alt: Eclipse project wizard

2. In the wizard, select the "Makefile Project with Existing Code" option under "C/C++" before clicking "Next".

3. In the next window, enter the path to your project in "Existing Code Location" and leave everything else as default.

.. image:: images/EclipseImportCode.png
    :width: 400
    :alt: Eclipse Import Existing Code window

4. You will now have a new project in the "Project Explorer" panel. To configure builds correctly, right click on your project and select properties. 

.. image:: images/EclipseProjectSettings.png
    :width: 400
    :alt: Eclipse project properties window

5. Under the "C/C++" build section, append ``/build/`` to the "Build directory" path.

Eclipse can now build your RP2040 project correctly.

Debugging RP2040 in Eclipse
---------------------------

By default Eclipse will try to debug programs using GDB on the host system. This will clearly not work for the RP2040, so some configuration changes are required. Before following these instructions, ensure that the openocd plugin for Eclipse is installed. This is installed by default on modern embedded C/C++ versions of Eclipse.

1. Right click on your project and select "Debug As" followed by "Debug Configurations".
   
.. image:: images/DebugConfigurations.png
    :width: 400
    :alt: Eclipse debug properties window

2. Select the "GDB OpenOCD Debugging" section.
3. In the "main" tab, enter the path to the .ELF file you want to debug.
4. In the "Debugger" tab, change the openocd executable path to the one installed in your PicoSDR (in these examples the pico SDK is installed in ``/opt/Raspberry/pico``)
5. Add the following to the OpenOCD options section: ``-f /opt/Raspberry/pico/openocd/tcl/interface/cmsis-dap.cfg -f /opt/Raspberry/pico/openocd/tcl/target/rp2040.cfg -s /opt/Raspberry/pico/openocd/tcl/ -c "adapter speed 5000"``. Make sure to amend these paths to point to the equivalent directories on your system.
6. Change the GDB executable name to ``/bin/gdb-multiarch``

.. image:: images/DebuggerTab.png
    :width: 400
    :alt: Eclipse debugger tab

7. In the "SVD Path" tab, change the file path to ``/opt/Raspberry/pico/pico-sdk/src/rp2040/hardware_regs/rp2040.svd``, making sure to amend this to your pico-sdk path.

.. image:: images/SVDTab.png
    :width: 400
    :alt: Eclipse SVD path tab