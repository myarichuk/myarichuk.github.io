---
title: Setting up WinDBG for analyzing memory dumps
date: 2020-02-13T01:25:23+02:00
categories:
  - Debugging
  - Post-mortem
tags:
 - Debugging
 - WinDBG
 - .Net
author: Michael Yarichuk
top_img: WinDbg.png
cover: /2020/02/12/setting-up-windbg/WinDbg_256.png
---
When I needed to investigate a memory dump for a first time, I stared at WinDBG window, not knowing how to begin. My google-fu yielded mixed results - I had to sift through lots of information, sometimes incorrect, sometimes outdated, only after some experimentation, I was able to actually understand what was going on.
Though, in a hindsight, WinDBG is much less complex than it seemed in the first place. 

## A word (or three) on WinDBG
WinDBG wikipedia article states that *WinDbg is a multipurpose debugger for the Microsoft Windows computer operating system, distributed by Microsoft*
That is nice, but what does it *mean*?  
  
Essentially, WinDBG provides a GUI and a CLI for a debugging engine (defined in DbgEng.dll) that comes as part of [Debugging Tools for Windows](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools), an engine that can debug both user-mode and kernel-mode code. As far as I know, WinDBG is also used to develop Windows itself by Microsoft devs, it is a low-level debugger without all the bells and whistles of Visual Studio, but it is very very powerful.
> Why CLI? Because most of the interaction with the "debugee" will be done using text commands typed in WinDBG command propmpt. 

By the way, there are more debuggers that share the same engine - [CDB](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugging-using-cdb-and-ntsd) for user-mode debugging and [KDB](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugging-using-kd-and-ntkd) for kernel-mode debugging. The only different between CDB, KDB and WinDBG is that CDB/KDB are console only and can debug only user-mode and kernel-mode respectively, while WinDBG has UI and can be used to debug both modes.
> In case all of this new to you, kernel-mode debugging means that you debug operating systems and drivers, and user-mode debugging means you debug regular programs that run within OS

## First Steps
Since we are talking about analyzing a memory dump, the first step would be to actually get a memory dump, which is essentially a snapshot of all that the process contains, things like thread information, allocated memory etc. This can be done via multiple tools, but personally, I usually use [Sysinternals Process Explorer](https://docs.microsoft.com/en-us/sysinternals/downloads/process-explorer)
Simply select the process you'd like to take a memory dump of, right click and... that's it.
![Process Explorer](process-explorer.png)
> Mini-dumps are very small and contain *only* information about threads and full dumps contain pretty much everything, their size is much larger, of course.


Now that we have a dump, we need to open it. WinDBG is a part of [Debugging Tools for Windows](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools) as well, but I would recommend using [WinDBG Preview version](https://www.microsoft.com/en-us/p/windbg-preview/9pgjgd53tn86?activetab=pivot:overviewtab) - which has the same features as "original" WinDBG, is quite stable and has much better performance (it utilizes multiple CPU cores, unlike the "original").

## The dump is opened, now what?
First, we need to properly setup PDB file sources. The idea is to provide a local cache for symbol files and supply symbol server addresses for both Microsoft system libraries and other symbol servers.
> If you are not sure *what* PDB files are, take a look at the article "[PDB Files: What Every Developer Must Know](https://www.wintellect.com/pdb-files-what-every-developer-must-know)". 

In order to configure PDB sources, you need to specify them in a symbol server search path format. This can be done either in WinDBG GUI or by setting `` _NT_SYMBOL_PATH`` environment variable.
For example, on my dev machine, it looks like this: ``cache*e:\Symbols;srv*e:\Symbols*https://msdl.microsoft.com/download/symbols``.

The format is ``cache*[local cache folder 1]*[local cache folder 2];srv*[local cache folder]*[symbol server path 1]*[symbol server path 2]* ... ``

What if symbols are missing or there is an issue? In this case, use ``!sym noisy on`` command to see what symbols are missing and where WinDBG tries to look for them - after this command, each operation that requires symbols would print information on where they were found.
Notice how after executing a ``!threads`` command, WinDBG outputs where did if looks for .Net Core symbols.
```dbgcommand
0:000> !sym noisy on
noisy mode - symbol prompts on
0:000> !threads
SYMSRV:  BYINDEX: 0x2
         e:\symbols
         coreclr.pdb
         38DDCD021DAD44F5831B9DB6C0C69E9B1
SYMSRV:  PATH: e:\symbols\coreclr.pdb\38DDCD021DAD44F5831B9DB6C0C69E9B1\coreclr.pdb
SYMSRV:  RESULT: 0x00000000
DBGHELP: coreclr - private symbols & lines 
        e:\symbols\coreclr.pdb\38DDCD021DAD44F5831B9DB6C0C69E9B1\coreclr.pdb
ThreadCount:      3
UnstartedThread:  0
BackgroundThread: 3
PendingThread:    0
DeadThread:       0
Hosted Runtime:   no
                                                                                                        Lock  
 DBG   ID OSID ThreadOBJ           State GC Mode     GC Alloc Context                  Domain           Count Apt Exception
   0    1 3b24 000002321A5A6580    20220 Preemptive  000002321C1E1680:000002321C1E1FD0 000002321a59b9c0 0     Ukn 
   4    2 1be0 000002321A6576F0    2b220 Preemptive  0000000000000000:0000000000000000 000002321a59b9c0 0     MTA (Finalizer) 
   6    3 40e8 000002321A65D840  102a220 Preemptive  0000000000000000:0000000000000000 000002321a59b9c0 0     MTA (Threadpool Worker) 
```

## Dump is open and symbols are configured. Now what?
Thats it. Now, you start debugging! 
Note that WinDBG is highly extensible, most of its commands are provided by extensions. By default, ``ext.dll`` extension gets loaded automatically, but if you have a memory dump of a .Net process, then you will need to use SOS extension (Son of Strike).
> commands provided by extensions always have the ``!`` prefix

## SOS extension
This extensions provides information about .Net internals; it is installed with .NET Framework.

So, how to load this extension? Simply, run the following command if you want to analyze memory dump taken from full .Net Framework process
```dbgcommand
.loadby sos clr
```

And if you took the memory dump from .Net Core process, the command would be slightly different/
```dbgcommand
.loadby sos coreclr
```

However, there is a very annoying gotcha: in order to properly analyze a .Net memory dump, loaded ``sos.dll`` must match the exact version of .Net the process ran under (because ``sos.dll`` is built as part of the framework)

In case of version mismatch, you will see an error like this:
```dbgcommand
0:037> !clrstack
The version of SOS does not match the version of CLR you are debugging.  Please load the matching version of SOS for the version of CLR you are debugging.
CLR Version: 4.0.30319.1
SOS Version: 4.0.30319.235
```

If you have this problem, you can either fetch the correct ``sos.dll`` from the machine the dump was taken on or use an undocumented behavior described in an awesome blog post: [Automatically Load the Right SOS for the Minidump](https://www.wintellect.com/automatically-load-the-right-sos-for-the-minidump/)

## Conclusion
This is the kind of post I wish I had when I started using WinDBG. Hopefully, you will find it useful.
