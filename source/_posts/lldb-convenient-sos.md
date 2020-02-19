---
title: Easy way to configure SOS in LLDB
date: 2020-02-18T20:48:13+02:00
categories:
  - Debugging
  - Post-mortem
tags: 
 - Debugging 
 - C#
 - LLDB
 - Linux
top_img: lldb.png
cover: /2020/02/18/lldb-convenient-sos/commandline.svg
---
I don't have much experience in using [LLDB](https://lldb.llvm.org/) to debug .Net Core, so when I stumbled upon this little gem while looking at [dotnet/diagnostics GitHub repository](https://github.com/dotnet/diagnostics/tree/master/src/Tools), I was instantly curious to try it: up to this point, in order to load the [SOS plugin](https://docs.microsoft.com/en-us/dotnet/framework/tools/sos-dll-sos-debugging-extension), I had to manually look up its folder, then specify it inside LLDB (via the ``load plugin`` command).
This tool, however, should configure LLDB to load SOS plugin automatically!
How? First install the tool via .Net Core CLI:
```bash
dotnet tool install -g dotnet-sos
```
Then use the tool to configure SOS:
```bash
dotnet sos install
```

After running the second command, you'd see something like this:
```output
Installing SOS to /home/michael/.dotnet/sos from /home/michael/.dotnet/tools/.store/dotnet-sos/3.1.57502/dotnet-sos/3.1.57502/tools/netcoreapp2.1/any/linux-x64
Creating installation directory...
Copying files...
Creating new /home/michael/.lldbinit file - LLDB will load SOS automatically at startup
SOS install succeeded
```
That's it. And it is awesome that Microsoft goes this far to create better developer experience, I think.
