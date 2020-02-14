---
title: Nice little gotcha when loading SOS after launching .Net executable
date: 2020-02-14T19:55:15+02:00
categories:
  - Debugging
tags:
 - Debugging
 - WinDBG
 - .Net
 - Gotcha
author: Michael Yarichuk
top_img: /2020/02/12/setting-up-windbg/WinDbg.png
cover: /2020/02/14/windbg-launch-net-executable/gotcha.svg
---
When you launch .Net executable with WinDBG, in order to "catch" something nasty like ``AccessViolationExcetion``, the application will stop after loading with a "debugger break", and you will see something like this:

```dbgcommand
(6b8.37dc): Break instruction exception - code 80000003 (first chance)
ntdll!LdrpDoDebuggerBreak+0x30:
00007ffc`391d121c cc              int     3
```

At this point, if you try to load SOS, you will see an error.
```dbgcommand
0:000> .loadby sos coreclr
Unable to find module 'coreclr'
```
This happens because CLR is not loaded yet; the following command will make sure that WinDBG will stop as soon as .Net is loaded.
```dbgcommand
sxe ld clrjit
```
So, the whole procedure will look like this:
```dbgcommand
0:000> .loadby sos coreclr
Unable to find module 'coreclr'
CommandLine: 
Cannot execute '', Win32 error 0n87
    "The parameter is incorrect."
0:000> sxe ld clrjit
0:000> g
ModLoad: 00007ffc`372d0000 00007ffc`372fe000   C:\WINDOWS\System32\IMM32.DLL
ModLoad: 00007ffc`214c0000 00007ffc`21554000   C:\Program Files\dotnet\host\fxr\3.1.1\hostfxr.dll
ModLoad: 00007ffc`21420000 00007ffc`214b2000   C:\Program Files\dotnet\shared\Microsoft.NETCore.App\3.1.1\hostpolicy.dll
ModLoad: 00007ffb`d8060000 00007ffb`d85cb000   C:\Program Files\dotnet\shared\Microsoft.NETCore.App\3.1.1\coreclr.dll
ModLoad: 00007ffc`376f0000 00007ffc`37846000   C:\WINDOWS\System32\ole32.dll
ModLoad: 00007ffc`37860000 00007ffc`37924000   C:\WINDOWS\System32\OLEAUT32.dll
ModLoad: 00007ffc`36e30000 00007ffc`36e56000   C:\WINDOWS\System32\bcrypt.dll
(6b8.37dc): Unknown exception - code 04242420 (first chance)
ModLoad: 00007ffb`b5800000 00007ffb`b611d000   C:\Program Files\dotnet\shared\Microsoft.NETCore.App\3.1.1\System.Private.CoreLib.dll
ModLoad: 00007ffc`212d0000 00007ffc`21412000   C:\Program Files\dotnet\shared\Microsoft.NETCore.App\3.1.1\clrjit.dll
ntdll!NtMapViewOfSection+0x14:
00007ffc`3919c5c4 c3              ret
```

At this point, running ``.loadby sos coreclr`` will work, because the CLR is already loaded at this point.
> When running ``.loadby`` commands, the absense of output is an indicator of success. Only if the commands fails, you will see an error.
