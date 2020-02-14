---
title: Segmentation faults when using P/Invoke = pointer issues? Not necessarily
date: 2020-01-26T23:28:27.378Z
categories:
  - Debugging
  - Post-mortem
tags: 
 - Debugging 
 - C#
 - Gdb
 - Linux
author: Michael Yarichuk
top_img: https://upload.wikimedia.org/wikipedia/commons/e/e3/GDB-screenshot.gif
cover: https://upload.wikimedia.org/wikipedia/commons/c/cc/Gdb_archer_fish.svg
---
When debugging new [RavenDB's](https://ravendb.net/) 32-bit [pager](https://en.wikipedia.org/wiki/Paging) for Linux-based ARM environments, which has platform specific functionality implemented in C and P/Invoked from C# code, I ran into an issue: when starting, RavenDB was throwing a segmentation fault and crashing. Since the C# code didn't change much, my immediate suspect was some sort of pointer issue in C code, such as trying to dereference a *null* pointer. 

## GDB is awesome for handling segfaults
The [GNU Debugger](https://en.wikipedia.org/wiki/GNU_Debugger) or GDB is very good at tracing such issues. Let's see how we can find a segfault source in a small example.  
Consider the following code:
```cpp
void throw_segment_fault()
{
	int* x = 0;
	*x = 5;
}

int main()
{
	throw_segment_fault();
}
```

Now, let's compile this and use GDB to find where the segfault happens:
> note that ``a.out`` is an executable compiled from *segmentFaultThrower.c*
```bash
gcc segmentFaultThrower.c
gdb --args ./a.out
```

Running GDB with such parameters will yield the following output
```gdb
GNU gdb (Ubuntu 8.1-0ubuntu3.2) 8.1.0.20180409-git
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./a.out...(no debugging symbols found)...done.
(gdb)
```

We have now run our application with GDB attached and paused. Executing ``run`` command actually starts the program.
```gdb
(gdb) run
Starting program: /mnt/e/projects/SegmentFaultTests/a.out

Program received signal SIGSEGV, Segmentation fault.
0x000000000800060a in throw_segment_fault ()
(gdb)
```

Now running ``bt`` command (backtrack) will show the stack trace where the segfault has happened.
```gdb
(gdb) bt
#0  0x000000000800060a in throw_segment_fault ()
#1  0x0000000008000621 in main ()
(gdb)
```

This is nice, but GDB can do better! Compiling our application with ``-g`` flag, will include debug information into executable.  
So, if we re-compile with the flag, start GDB and issue ``run`` command, we will see the following
```gdb
(gdb) run
Starting program: /mnt/e/projects/SegmentFaultTests/a.out

Program received signal SIGSEGV, Segmentation fault.
0x000000000800060a in throw_segment_fault () at segmentFaultThrower.c:5
5               *x = 5;
```

Running ``bt`` will now print source code lines in the stack trace
```gdb
(gdb) bt
#0  0x000000000800060a in throw_segment_fault () at segmentFaultThrower.c:5
#1  0x0000000008000621 in main () at segmentFaultThrower.c:10
```

## If segfaults happen when P/Invoking a function, it is not necessarily because of null pointers! 
The new 32-bit pager that I mentioned at the beginning, was using P/Invokes to C code that was used to access operating-system APIs, such as memory-mapping related functions. 

I made sure to compile the C library with ``-g`` flag and ran RavenDB with GDB attached. I saw the following output (notice the output of ``bt`` command):
```gdb
Thread 1 "Raven.Server" received signal SIGSEGV, Segmentation fault.
rvn_mmap_file (sz=65536, flags=<optimized out>, handle=0x0, offset=9151260362719350484, addr=0xe6f2ebb, detailed_error_code=0x76af562c <vtable for InlinedCallFrame+8>) at src/posix/mapping.c:421
421         *addr = rvn_mmap(NULL, sz, PROT_READ | PROT_WRITE, mmap_flags, mfh->fd, offset);
(gdb) bt
#0  rvn_mmap_file (sz=65536, flags=<optimized out>, handle=0x0, offset=9151260362719350484, addr=0xe6f2ebb, detailed_error_code=0x76af562c <vtable for InlinedCallFrame+8>) at src/posix/mapping.c:421
#1  0x513da610 in ?? ()
Backtrace stopped: previous frame identical to this frame (corrupt stack?)
```
Such output looked weird to me, especially the *corrupt stack* part, so I looked at the relevant code.  
The P/Invoke call in C# part looked like this:
```cs
var rc = rvn_mmap_file(size, 
  _copyOnWriteMode ? MmapOptions.CopyOnWrite : MmapOptions.None, 
  _handle, offset, 
  out var startingBaseAddressPtr,
  out var errorCode);
```
In the C part, ``rvn_mmap_file()`` signature looks like this:
```cpp
EXPORT int32_t rvn_mmap_file(int64_t sz, int64_t flags, void *handle, int64_t offset, void **addr, int32_t *detailed_error_code)
```
>In this case, ``int64_t`` is a typedef for ``long long`` and ``int32_t`` is a typedef for ``int``.  
  
The first thing I noticed is that the ``handle`` parameter value is 0 (which means ``null`` pointer) and the ``offset`` parameter has unreasonably large value. 
In C# code, by the point the ``rvn_mmap_file()`` is invoked, ``_handle`` is guaranteed to have a value (otherwise the code would have failed earlier). Together with *corrupt stack* notification from GDB while executing the ``bt`` command, I suspected that some offsets are wrong, since the segfault happens when invoking ``rvn_mmap_file()`` itself. 

After looking some more at the code, I noticed that the ``flags`` parameter is ``int64_t`` and the definition of the corresponding flags enum in C# looks like this:
```cs
[Flags]
public enum MmapOptions
{
   None = 0,
   CopyOnWrite = (1 << 0),
   DeleteOnClose = (1 << 1),
}
```
  
Since in C# enums are of ``System.Int32`` type, this in fact was the issue. The fix was simply to change the ``flags`` type so the signature became:
```cpp
EXPORT int32_t rvn_mmap_file(int64_t sz, int32_t flags, void *handle, int64_t offset, void **addr, int32_t *detailed_error_code)
```
  
## The moral of the story
Usually, segmentation faults are associated with null point dereference or other types of pointer issue, but as we can see here, this doesn't have to be so.