---
layout: post
title:  "Exploring Linux IO with cat"
---

I have been reading an interesting series of [blog posts about io\_uring](https://unixism.net/2020/04/io-uring-by-example-part-1-introduction/) and was inspired to investigate Linux syscall performance myself.

In the spirit of the blog posts that inspired me, let's explore `cat`.
GNU Coreutils `cat` seems like a pretty simple utility.
It concatenates its inputs and writes them to stdout.

Let's see how fast the coreutils implementation runs:

```
[admin@uqbar hello-syscalls]$ time cat /dev/zero | head -c 1G > /dev/null

real    0m0.349s
user    0m0.079s
sys     0m0.457s
[admin@uqbar hello-syscalls]$ time cat /dev/zero | head -c 10G > /dev/null

real    0m3.327s
user    0m0.796s
sys     0m4.598s
```

The results seem pretty linear, which is reasonable.
These are, after all, bytes on a hard drive which all need to be moved.
They can be chunked into huge batches, but at some point you're going to be running in a loop over the chunks.

Let's see if we can write a similar program.
We won't worry about reading from stdin, concatenating multiple files, reading from special files, or anything but the most basic functionality.
A CS 101 implementation might look like:

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char **argv) {
    FILE *f;
    int c;

    if (argc != 2) {
        fprintf(stderr, "usage: %s [FILE]\n", argv[0]);
        return 1;
    }

    f = fopen(argv[1], "r");
    if (!f) {
        perror("fopen");
        return 1;
    }

    while ((c = fgetc(f)) != EOF) {
        fputc(c, stdout);
    }

    return 0;
}
```

If we compile and test this, we can see the results are something like:

```
[admin@uqbar hello-syscalls]$ time ./cat /dev/zero | head -c 1G > /dev/null

real    0m3.872s
user    0m3.648s
sys     0m0.800s
[admin@uqbar hello-syscalls]$ time ./cat /dev/zero | head -c 10G > /dev/null

real    0m38.682s
user    0m36.495s
sys     0m8.003s
```

Okay, this is definitely a lot slower than coreutils.
Let's break down what `time` is telling us.
We're still scaling linearly.
The user time is over 50x higher than that of coreutils, so we're definitely doing more work in userspace.
The sys time is a bit under double coreutils, so while we're doing some more syscalls, it's not our biggest problem here.

### Stop copying!

Since fputc and fgetc are buffered functions, not every call results in a `read` or `write` syscall.
However, that means there's a userspace buffer associated with each file, and our program is actually copying gigabytes of characters from the input buffer, to the local variable `c`, to the output buffer.
That's a lot of copying between userspace buffers!
Let's patch this to `read` straight into a buffer, and then `write` that buffer to stdout.

A pretty straightforward patch might look like this:

```
[admin@uqbar hello-syscalls]$ git diff
diff --git a/cat.c b/cat.c
index e43d051..b837364 100644
--- a/cat.c
+++ b/cat.c
@@ -1,23 +1,42 @@
 #include <stdio.h>
 #include <stdlib.h>
 
+#include <fcntl.h>
+#include <unistd.h>
+
+#define BUF_SIZE 4096
+
 int main(int argc, char **argv) {
-    FILE *f;
-    int c;
+    int fd;
+    ssize_t nr;
+    char buf[BUF_SIZE];
 
     if (argc != 2) {
         fprintf(stderr, "usage: %s [FILE]\n", argv[0]);
         return 1;
     }
 
-    f = fopen(argv[1], "r");
-    if (!f) {
-        perror("fopen");
+    fd = open(argv[1], O_RDONLY);
+    if (fd < 0) {
+        perror("open");
         return 1;
     }
 
-    while ((c = fgetc(f)) != EOF) {
-        fputc(c, stdout);
+    while ((nr = read(fd, buf, BUF_SIZE)) > 0) {
+        ssize_t tw = 0;
+        while (tw < nr) {
+            ssize_t nw = write(STDOUT_FILENO, buf + tw, nr - tw);
+            if (nw < 0) {
+                perror("write");
+                return 1;
+            }
+            tw += nw;
+        }
+    }
+
+    if (nr < 0) {
+        perror("read");
+        return 1;
     }
 
        return 0;
[admin@uqbar hello-syscalls]$ time ./cat /dev/zero | head -c 1G > /dev/null

real    0m0.404s
user    0m0.165s
sys     0m0.643s
[admin@uqbar hello-syscalls]$ time ./cat /dev/zero | head -c 10G > /dev/null

real    0m3.952s
user    0m1.665s
sys     0m6.231s
```

This resulted in a 20x reduction in user time compared to fputc cat.
Our user time is now only 2x higher than that of coreutils.

Let's dig into what coreutils is doing.
The meat of it seems to be in the function `simple_cat` of src/cat.c, which calls `full_write` and `safe_read` from gnulib.
Those are pretty straightforward wrappers around `read` and `write` that handle errors and retries, so I won't dig into them here.

```c
static bool
simple_cat (char *buf, idx_t bufsize)
{
  /* Loop until the end of the file.  */

  while (true)
    {
      /* Read a block of input.  */

      ptrdiff_t n_read = safe_read (input_desc, buf, bufsize);
      if (n_read < 0)
        {
          error (0, errno, "%s", quotef (infile));
          return false;
        }

      /* End of this file?  */

      if (n_read == 0)
        return true;

      /* Write this block out.  */

      if (full_write (STDOUT_FILENO, buf, n_read) != n_read)
        write_error ();
    }
}
```

Ok.
This doesn't seem fundamentally different from our implementation.
So what can explain the performance difference?

### Call me

We already looked at the code and it's not clear what the difference is.
So let's poke around at runtime and see what syscalls are actually being made by either implementation.
Running programs under strace can affect their performance, but in this case I can't think of any reason it would affect the number of syscalls made.

```
[admin@uqbar hello-syscalls]$ strace -e write -c cat /dev/zero | head -c 1G > /dev/null
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.143143          17      8193         1 write
------ ----------- ----------- --------- --------- ----------------
100.00    0.143143          17      8193         1 total
[admin@uqbar hello-syscalls]$ strace -e write -c ./cat /dev/zero | head -c 1G > /dev/null
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.427177           1    262145         1 write
------ ----------- ----------- --------- --------- ----------------
100.00    0.427177           1    262145         1 total
```

There you have it. We can see that our implementation is making 32 times as many `write` syscalls as coreutils.
Maybe we can just crank our buffer size up by 32x?

```
[admin@uqbar hello-syscalls]$ git diff
diff --git a/cat.c b/cat.c
index b837364..d3a8e49 100644
--- a/cat.c
+++ b/cat.c
@@ -4,7 +4,7 @@
 #include <fcntl.h>
 #include <unistd.h>
 
-#define BUF_SIZE 4096
+#define BUF_SIZE 4096 * 32
 
 int main(int argc, char **argv) {
     int fd;
[admin@uqbar hello-syscalls]$ strace -e write -c ./cat /dev/zero | head -c 1G > /dev/null
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.146091          17      8193         1 write
------ ----------- ----------- --------- --------- ----------------
100.00    0.146091          17      8193         1 total
[admin@uqbar hello-syscalls]$ time ./cat /dev/zero | head -c 10G > /dev/null

real    0m3.303s
user    0m0.799s
sys     0m4.531s
```

Awesome!
It looks like we're getting performance pretty similar to coreutils.

### Adjusting buffers

Just for fun, let's look at the syscall counts for our fputc implementation:

```
[admin@uqbar hello-syscalls]$ strace -e write -c ./cat /dev/zero | head -c 1G > /dev/null
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.130992           0    262145         1 write
------ ----------- ----------- --------- --------- ----------------
100.00    0.130992           0    262145         1 total
```

The same number of syscalls as our `write` implementation!
Like I mentioned before, fputc and fgetc are buffered functions.
It looks like they're even buffering the same number of bytes as our other implementation was.
What would happen if we adjust the size of the buffers to match coreutils?

```
[admin@uqbar hello-syscalls]$ git diff                                            
diff --git a/cat.c b/cat.c
index d5ec9de..3b0794f 100644
--- a/cat.c
+++ b/cat.c
@@ -1,9 +1,13 @@
 #include <stdio.h>
 #include <stdlib.h>

+#define BUF_SIZE 4096 * 32
+
 int main(int argc, char **argv) {
     FILE *f;
     int c;
+    char buf_f[BUF_SIZE];
+    char buf_stdout[BUF_SIZE];

     if (argc != 2) {
         fprintf(stderr, "usage: %s [FILE]\n", argv[0]);                       
@@ -16,6 +20,16 @@ int main(int argc, char **argv) {                           
         return 1;
     }

+    if (setvbuf(f, buf_f, _IOFBF, BUF_SIZE) != 0) {                           
+        perror("setvbuf");
+        return 1;
+    }
+
+    if (setvbuf(stdout, buf_stdout, _IOFBF, BUF_SIZE) != 0) {                 
+        perror("setvbuf");
+        return 1;
+    }
+
     while ((c = fgetc(f)) != EOF) {
         fputc(c, stdout);
     }
[admin@uqbar hello-syscalls]$ strace -e write -c ./cat /dev/zero | head -c 1G > /dev/null
% time     seconds  usecs/call     calls    errors syscall                     
------ ----------- ----------- --------- --------- ----------------            
100.00    0.008961           1      8193         1 write                       
------ ----------- ----------- --------- --------- ----------------            
100.00    0.008961           1      8193         1 total                       
[admin@uqbar hello-syscalls]$ time ./cat /dev/zero | head -c 10G > /dev/null      

real    0m45.747s
user    0m41.174s
sys     0m4.491s
```

Cool!
Our sys time is now similar to that of our `write` implementation.
Our user time is still much higher, but that's because of the copying.

### Even less copying!

What if we could copy data from one file descriptor to another, without the need to copy it into userspace at all?

The good news is we can, with syscalls like `splice` and `sendfile`.

The bad news is that their portability is limited, and their capabilities seem to vary a bit depending on kernel version.
Neither can write to a file with `O_APPEND`.
`splice` requires that one of the file descriptors is a pipe, and is only available in Linux.

The following `splice` based implementation won't work unless stdout is piped, but we can still demonstrate the principle.

```c
#define _GNU_SOURCE

#include <stdio.h>
#include <stdlib.h>

#include <fcntl.h>
#include <unistd.h>

#define BUF_SIZE 4096 * 32

int main(int argc, char **argv) {
    int fd;
    ssize_t n;

    if (argc != 2) {
        fprintf(stderr, "usage: %s [FILE]\n", argv[0]);
        return 1;
    }

    fd = open(argv[1], O_RDONLY);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    do {
        n = splice(fd, NULL, STDOUT_FILENO, NULL, BUF_SIZE, 0);
    } while (n > 0);

    if (n < 0) {
        perror("splice");
        return 1;
    }

    return 0;
}
```
```
[admin@uqbar hello-syscalls]$ time cat /dev/zero | head -c 10G > /dev/null

real    0m3.449s
user    0m0.783s
sys     0m4.765s
[admin@uqbar hello-syscalls]$ time ./cat /dev/zero | head -c 10G > /dev/null

real    0m3.186s
user    0m1.015s
sys     0m4.529s
[admin@uqbar hello-syscalls]$ cat /dev/zero | dd bs=1M > /dev/null
^C24+103074 records in
24+103074 records out
13555466240 bytes (14 GB, 13 GiB) copied, 2.79653 s, 4.8 GB/s

[admin@uqbar hello-syscalls]$ ./cat /dev/zero | dd bs=1M > /dev/null
^C37+264414 records in
37+264414 records out
17368547328 bytes (17 GB, 16 GiB) copied, 2.88044 s, 6.0 GB/s
```
