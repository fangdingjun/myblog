以前调试Linux驱动的时候，知道在kernel里使用dump_stack可以打印出内核栈信息，可以很方便查看函数的调用关系。 现在我才知道glibc也支持获取栈信息，并且有相关的函数：


    #include <execinfo.h>
    int backtrace(void **buffer, int size);
    char **backtrace_symbols(void *const *buffer, int size);
    void backtrace_symbols_fd(void *const *buffer, int size, int fd);


以下是Man里的示例代码

    #include <execinfo.h>
    #include <stdio.h>
    #include <stdlib.h>
    #include <unistd.h>

    void
    myfunc3 (void)
    {
      int j, nptrs;
      #define SIZE 100
      void *buffer[100];
      char **strings;

      nptrs = backtrace (buffer, SIZE);
      printf ("backtrace() returned %d addressesn", nptrs);

      /* The call backtrace_symbols_fd(buffer, nptrs, STDOUT_FILENO)
         would produce similar output to the following: */

      strings = backtrace_symbols (buffer, nptrs);
      if (strings == NULL)
        {
          perror ("backtrace_symbols");
           exit (EXIT_FAILURE);
        }

      for (j = 0; j < nptrs; j++)
        printf ("%sn", strings[j]);

      free (strings);
    }

    static void  			/* "static" means don't export the symbol... */
    myfunc2 (void)
    {
       myfunc3 ();
    }

    void
    myfunc (int ncalls)
    {
      if (ncalls > 1)
        myfunc (ncalls - 1);
      else
        myfunc2 ();
    }

    int
    main (int argc, char *argv[])
    {
      if (argc != 2)
        {
          fprintf (stderr, "%s num-callsn", argv[0]);
          exit (EXIT_FAILURE);
        }

      myfunc (atoi (argv[1]));
      exit (EXIT_SUCCESS);
    }


编译时要加 -rdynamic 参数，否则只有函数地址，没有函数名
 

    $ gcc -rdynamic prog.c -o prog
    $ ./prog 3
    backtrace() returned 8 addresses
    ./prog(myfunc3+0x5c) [0x80487f0]
    ./prog [0x8048871]
    ./prog(myfunc+0x21) [0x8048894]
    ./prog(myfunc+0x1a) [0x804888d]
    ./prog(myfunc+0x1a) [0x804888d]
    ./prog(main+0x65) [0x80488fb]
    /lib/libc.so.6(__libc_start_main+0xdc) [0xb7e38f9c]
    ./prog [0x8048711]


