Lazy Memory Management
======================

In C there is no garbage collector and no smart pointers, you have to do your own memory management. This can lead to mistakes, which is why wrapping ``malloc`` can sometimes be useful to ensure no memory leaks occur. The following is an example of hwo this can be done, however it not thoroughly tested and *should not be used on production code*.

Implementation
--------------

First, we create a file called ``lazy_memory.h`` which contains out ``malloc`` wrapper. This contains a structure, two global variables and three functions. The ``lmalloc()`` function is what we will use instead of ``malloc``. It calls the ``stdlib.h`` ``malloc`` function and stores a copy of the pointer in a linked list. ``lfree()`` will free a given pointer and delete it's reference in the linked list if a reference is found. When it comes for the program to end, we call ``free_all()`` which goes through the remaining linked list and de-allocates all the associated memory.

.. code-block:: c 

    #include <stdlib.h>

    typedef struct __memory__ {
        void * ptr;
        struct __memory__ * next;
    } __memory__;

    static __memory__ *end = NULL;
    static __memory__ *start = NULL;

    static void __safe_free(void* ptr){
        if(ptr != NULL) free(ptr);
    }

    void * lmalloc(size_t size){
        if(end == NULL){
            end = malloc(sizeof(__memory__));
            start = end;
        } else {
            end->next = malloc(sizeof(__memory__));
            end = end->next;
        }

        end->ptr = malloc(size);
        end->next = NULL;
        return end->ptr;
    }

    void lfree(void* ptr){
        if((long) ptr == (long) start->ptr){
            __memory__ *tmp_start = start->next;
            __safe_free(start);
            start = tmp_start;
            goto free_mem;
        }
        
        __memory__ * mem = start->next;
        __memory__ * prev = start;
        do {
            if((long) ptr == (long) mem->ptr){
                prev->next = mem->next;
                __safe_free(mem);
                break;
            }
            prev = mem;
            mem = mem->next;
        } while(mem != NULL);
                
    free_mem:
        __safe_free(ptr);
    }

    void free_all(){
        if(start == NULL) return;

        do {
            __memory__ * tmp = start->next;
            __safe_free(start->ptr);
            __safe_free(start);
            start = tmp;
        } while(start != NULL);

    }

To test this, lets create a simply ``main.c`` file with the following contents:

.. code-block:: c 

    #include <stdio.h>
    #include "lazy_memory.h"

    int main(){
        void * ptrs[10];
        for(int i = 0; i < 10; i++){
            ptrs[i] = lmalloc(i);
        }

        for(int i = 0; i < 10; i++){
            printf("%04lx ", (long) ptrs[i]);
        }

        printf("\n");

        lfree(ptrs[0]);
        lfree(ptrs[5]);
        lfree(ptrs[9]);

        printf("%04lx ", (long)ptrs[1]);
        printf("%04lx ", (long)ptrs[2]);
        printf("%04lx ", (long)ptrs[3]);
        printf("%04lx ", (long)ptrs[4]);
        printf("%04lx ", (long)ptrs[6]);
        printf("%04lx ", (long)ptrs[7]);
        printf("%04lx\n", (long)ptrs[8]);

        free_all();
        return 1;
    }

We can then compile this code and check for any leaks using ``valgrind``. In this case, no memory leaks are detected.

The following should be noted:

    - ``free_all()`` does not check for any memory assigned using the standard ``malloc``.
    - If any variable in the linked list is manually deallocated using ``free()`` then ``free_all()`` will fail. 
    - ``lmalloc()`` does not check for or handle ``malloc`` returning ``NULL``.

Performance Hit 
----------------

To test the performance of this method I wrote two basic C programs. The first, shown below, allocates 1000000 pointers of 2 bytes in size, frees some of them, and then manually frees the rest:

.. code-block:: c 

    #include <stdio.h>
    #include "lazy_memory.h"

    int main(){

        void * ptrs[1000000];
        for(int i = 0; i < 1000000; i++){
            ptrs[i] = malloc(2);
        }

        for(int i = 0; i < 1000000; i++){
            printf("%04lx ", (long) ptrs[i]);
        }

        printf("\n");

        free(ptrs[0]);
        free(ptrs[5]);
        free(ptrs[9]);

        for(int i = 0; i < 1000000; i++){
            if(i == 0 || i == 5 || i == 9) continue;
                printf("%04lx ", (long) ptrs[i]);
        }

        for(int i = 0; i < 1000000; i++){
            if(i == 0 || i == 5 || i == 9) continue;
            free(ptrs[i]);
        }
        return 1;
    }

The second program uses the lazy memory management versions of malloc and free and is shown below:

.. code-block:: c 

    #include <stdio.h>
    #include "lazy_memory.h"

    int main(){

        void * ptrs[1000000];
        for(int i = 0; i < 1000000; i++){
            ptrs[i] = lmalloc(2);
        }

        for(int i = 0; i < 1000000; i++){
            printf("%04lx ", (long) ptrs[i]);
        }

        printf("\n");

        lfree(ptrs[0]);
        lfree(ptrs[5]);
        lfree(ptrs[9]);

            for(int i = 0; i < 1000000; i++){
            if(i == 0 || i == 5 || i == 9) continue;
                    printf("%04lx ", (long) ptrs[i]);
            }

        free_all();
        return 1;
    }


Using the first method with manual memory management the program took 0.097 seconds to execute using the command ``time (./a.out > /dev/null)``. With lazy memory management, the program took 0.121 seconds to run using the same command. This indicates that using this lazy memory management method adds approximately 20% to required execution time for memory allocation and de-allocation. It should also be noted that, using ``lfree`` on more recently allocated memory will take longer than on older allocated memory. This is due to ``lfree`` having to search through more of the linked list to find the correct entry to remove.