# Lab1: Xv6 and Unix utilities

:penguin: **ALL ASSIGNMENTS HAVE PASSED THE TESTS** :white_check_mark:

- [x] [sleep(easy)](#1-sleep-easy)
- [x] [pingpong(easy)](#2-pingpong-easy)
- [x] [primes (moderate)/(hard)](#3-primes-moderatehard)
- [x] [find (moderate)](#4-find-moderate)
- [x] [xargs (moderate)](#5-xargs-moderate)

This lab will familiarize you with xv6 and its system calls.

## [1. sleep (easy)](#lab1-xv6-and-unix-utilities)

Read the recommended files following the guide and you will benefits from them.
Don't forget to find `UPROGS` and add your program at the bottom in `Makefile`.

### _user/sleep.c_

```c
#include "kernel/types.h"
#include "user/user.h"

void
main(int argc, char *argv[]) 
{
    int interval;

    if (argc < 2) {
        fprintf(2, "An argument needs to determine the sleep time.\n");
        exit(1);
    }

    /* argv[0] is the program name */
    interval = atoi((const char *)argv[1]);
    sleep(interval);

    exit(0);
}
```

## [2. pingpong (easy)](#lab1-xv6-and-unix-utilities)

_If no data is available, a `read` on a pipe will wait for either data to be written or for all file descriptors referring to the `write` end to be closed._ [page16, book-risc-rev3.pdf](book-riscv-rev3.pdf).

### _user/pingpong.c_

```c
#include "kernel/types.h"
#include "user/user.h"

void
main(void)
{
    int p[2]; // pipe
    int pid;

    // Create a pipe, put read/write file descriptors in p[0] and p[1].
    // This operation will create pipe for parent and child separately,
    // however, with shared buffer.
    pipe(p);
    if (fork() == 0) {
        char buf[2];
        if (read(p[0], buf, 1) != 1) {
            fprintf(2, "Cannot read one byte from parent.\n");
            exit(1);
        }
         // prevent child read blocking
        close(p[0]);
        if (write(p[1], (char *)"a", 1) != 1) {
            fprintf(2, "Cannot write one byte to parent.\n");
            exit(1);
        }
        // close child write to prevent the parent read blocking
        close(p[1]);
        pid = getpid();
        fprintf(1, "%d: received ping\n", pid);
        exit(0);
    } // child
    else {
        if (write(p[1], (char *)"b", 1) != 1) {
            fprintf(2, "Cannot write one byte to child.\n");
            exit(1);
        }
        close(p[1]);
        wait(0); // wait child process exit
        char buf[2];
        if (read(p[0], buf, 1) != 1) {
            fprintf(2, "Cannot read one byte from child.\n");
            exit(1);
        }
        close(p[0]);
        pid = getpid();
        fprintf(1, "%d: received pong\n", pid);
    } // parent

    exit(0);
}
```

## [3. primes (moderate)/(hard)](#lab1-xv6-and-unix-utilities)

`read` and `write` syscall needs to coordinate with each other. I partially compile the code with `write` syscall only, leading to -1 returned code from `write`. I stuck in this error almost 1 hour!!!

The process gets data from its left neighbors, which means we need to write the processed data to child `read` pipe so that the child process can deal with them separately.

### _user/primes.c_

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

#define DEBUG (1)

void
newproc(int p[2]) 
{
    int prime, n, ret;

    close(p[1]); // prevent read blocking when there's no other data
    ret = read(p[0], (int *)&prime, 4);
    /* The write pipe has been closed, the end of a data file had been reached*/
    if (ret == 0) {
        close(p[0]);
        exit(0);
    }
    if (ret != 4) {
        fprintf(2, "pid %d: read prime number error\n", getpid());
        exit(1);
    }
    fprintf(1, "prime %d\n", prime);

    /* Create new pipe for next data */
    int pnext[2];
    pipe(pnext);
    /* Create new process for next stage */
    if (fork() == 0) {
        newproc(pnext);
    }
    else {
        close(pnext[0]); // parent needs only write for new pipe
        while (1) {
            ret = read(p[0], (int *)&n, 4);
            if (ret == 0) {
                break;
            }
            if (ret != 4) {
                fprintf(2, "pid %d: read error\n", getpid());
                exit(1);
            }
            if (n % prime) {
                write(pnext[1], (int *)&n, 4);
            }
        }
        /* cleanup */
        close(p[0]);
        close(pnext[1]);
        wait(0);
    }
    exit(0);
}


int
main(void) 
{
    int i, p[2];

    pipe(p);

    if (fork() != 0) {
        close(p[0]); // write only
        /* first process needs to scan all numbers */
        for (i = 2; i <= 35; i++) {
            if (write(p[1], (int *)&i, 4) != 4) {
                fprintf(2, "pid %d: write error\n", getpid());
                exit(1);
            }
        }
        close(p[1]);
        wait(0);
        exit(0);
    }
    else {
        newproc(p);
    }

    return 0;
}
```

## [4. find (moderate)](#lab1-xv6-and-unix-utilities)

Copy the main part of `ls` function in `user/ls.c`. If you fully understand the logic of `ls` function, it is easy for you to solve this problem.

- In `kernel/fs.h`, the annotation explains that the directory is a file containing a sequence of  `struct dirent`. Therefore, we need to check the _dirent_'s name to guarantee the no recursion into "." and "..".
- Function `fstat()` has store the file details in the `struct stat st` variable. Function `stat()` acquires details from the specific filename and store the information in `st`. Pointer `p` stores the address of `buf` end, so `buf` in `stat(buf, &st)` consist of the path prefix and the next searched item name.
  - If the item is `T_DIR`, we recurse into this directory.
  - If the item is `T_FILE`, we compare its name with target file name.

### _user/find.c_

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"


void
find (char *path, char *fname)
{
    int fd;
    char buf[512], *p;
    struct dirent de;
    struct stat st;

    if((fd = open(path, 0)) < 0) {
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    /* show the file related info obtained from 'fd'(file descriptor) */
    if(fstat(fd, &st) < 0) {
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    switch (st.type) {
        case T_DEVICE:
        case T_FILE:
            fprintf(2, "First argument needs to be an directory.\n");
            break;

        case T_DIR:
            if (strlen(path) + 1 + DIRSIZ + 1 > sizeof(buf))
            {
                printf("ls: path too long\n");
                exit(1);
            }
            strcpy(buf, path);
            p = buf + strlen(buf);
            *p++ = '/';
            /* Find a useful target file */
            while (read(fd, &de, sizeof(de)) == sizeof(de)) {
                if (de.inum == 0 || strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
                    continue;
                memmove(p, de.name, DIRSIZ);
                p[DIRSIZ] = 0;
                /* stat will show the details of the file structure */
                if (stat(buf, &st) < 0) {
                    printf("ls: cannot stat %s\n", buf);
                    continue;
                }
                if (st.type == T_DIR) {
                    find(buf, fname);
                }
                else if (st.type == T_FILE) {
                    if (strcmp(fname, de.name) == 0) {
                        fprintf(1, "%s\n", buf);
                    }
                }   
            }
            break;
    }
    close(fd);
}


int
main(int argc, char *argv[])
{
    if(argc < 2) {
        fprintf(2, "please give the program a"
                    "correct file or directory name.\n");
        exit(1);
    }

    find (argv[1], argv[2]);
    exit(0);

    return 0;
}
```

## [5. xargs (moderate)](#lab1-xv6-and-unix-utilities)

_When `exec` succeeds, it does not return to the calling program. `exec` takes two arguments: the name of the file containing the executable and an array of string arguments._

At **chapter1, page 12**, the author gives an example of `exec` usage. Therefore, `xargs` string can be obtained from `argv[1]`, identical with the user program name and executable command. Therefore, according to the usage introduction, the second argument array in `exec` needs to contains `argv[1]`.

I originally have a wishful thinking that I need to combine the pipe input all together. Actually, the `xargs` handles pipe input read from each line with its own command arguments.

### _user/xargs.c_

```c
#include "kernel/types.h"
#include "kernel/param.h"
#include "user/user.h"

int
readline(char *args[MAXARG], int offset)
{
    char buf[512];
    int idx, retval, spilt_idx;

    idx = 0;
    while (1) {
        retval = read(0, buf + idx, 1);
        if (retval == 0) {
            break;
        }
        if (retval != 1) {
            fprintf(2, "read error\n");
            exit(1);
        }

        if (idx >= 511) {
            fprintf(2, "Argument too long\n");
            exit(1);
        }

        if (*(buf + idx) == '\n') {
            break;
        }
        idx++;
    }
    buf[idx] = 0;
    if (idx == 0) {
        return 0;
    }

    spilt_idx = 0;
    /* spilt one line arguments */
    while (spilt_idx < idx) {
        args[offset++] = buf + spilt_idx;
        while (buf[spilt_idx] != ' ' && spilt_idx < idx) {
            spilt_idx++;
        }
        while (buf[spilt_idx] == ' ' && spilt_idx < idx) {
            buf[spilt_idx++] = 0;
        }
    }
    return offset;
}


int
main (int argc, char *argv[])
{
    int i;
    char *cmd;
    char *args[MAXARG];

    if (argc < 2) {
        fprintf(2, "xargs needs at least one command, optional arguments\n");
        fprintf(2, "Usage: some_commands | xargs command (arg ...)\n");
        exit(1);
    }

    /* Obtain xargs command first */
    cmd = malloc(strlen(argv[1] + 1)); // char array ends up with '\0'
    strcpy(cmd, argv[1]);

    /* Obtain xargs arguments */
    for (i = 1; i < argc; i++) {
        args[i - 1] = malloc(strlen(argv[i]) + 1);
        strcpy(args[i - 1], argv[i]);
    }

    /* Obtain input from the pipe, attach them behind the args */
    while (readline(args, argc - 1 /* move forward */)) {
        if (fork() == 0) {
            exec(cmd, args);
            fprintf(2, "exec error\n");
            exit(1);
        }
        wait(0);
    }
    exit(0);

    return 0;
}
```
