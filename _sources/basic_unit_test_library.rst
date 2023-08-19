Basic Unit Test Library
=======================

While unit tests provide greater assurance for developed software, setting up existing C unit test libraries can be tiresome. These notes will show how to create a basic unit test library using some simple regex and function pointers. 

Creating the Unit Test Source
-----------------------------

The first step is to create a C source file containing the unit tests. This should import the header for the code you want to test. Each test function should start with the word "test", return 0 on success, and return -1 on failure. The code below shows an example for testing a function which always returns zero. This will then need to be compiled into an object file.

.. code-block:: c

    #include <stdlib.h>
    #include <stdint.h>
    #include <stdio.h>

    #include "uut.h"

    int test_1(){
        int output = return_zero();
        if(output == 0){
            return 0;
        } else {
            return -1;
        } 
    }

Creating the Unit Test add_executable
-------------------------------------

There is a good chance you will have a lot of unit test functions - this bash script will create a C source file which calls them all in sequence using an array of function pointers.

.. code-block:: bash

    #!/bin/bash

    # Assume $1 is a .o object file
    # Assume $2 is the include file

    output=$(nm -g $1| grep "test_")

    SAVEIFS=$IFS
    IFS=$'\n'
    output=($output)
    IFS=$SAVEIFS

    cat > test.c <<EOF
    #include <stdlib.h>
    #include <stdio.h>
    #include <stdint.h>

    #include "$2"
    EOF

    for i in "${output[@]}"
    do
        len=`expr length "$i"`
        echo "int ${i:19:$len-19}();" >> test.c
    done

    cat >> test.c <<EOF

    int main(){
        int (*tests[])() = {
    EOF

    for i in "${output[@]}"
    do
        len=`expr length "$i"`
        echo "      &${i:19:$len-19}," >> test.c
    done

    cat >> test.c <<EOF
        };
        for(int i = 0; i < 62; i++){
                int result = (*tests[i])();
                if(result == 0){
                    printf("PASS\n");
                } else {
                    printf("FAIL\n");
                }
            }

        return 0;
    }
    EOF

Writing a Makefile
------------------

All these steps can be combined with the help of a Makefile, such as the one below.

.. code-block::

    uut.o:
        $(CC) -Wall -c uut.c -o uut.o

    unit_tests.o:
        $(CC) -Wall -c unit_tests.c -o unit_tests.o

    test:
        bash create_tests.sh unit_tests.o uut.h
        $(CC) -Wall -c test.c -o test.o
        $(CC) -Wall test.o uut.o unit_tests.o -o test
        ./test
