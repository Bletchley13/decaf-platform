# Startup #
> This part is a quick start guide for setting up and running DECAF, the binary analysis platform based on [QEMU](http://wiki.qemu.org/Main_Page). It assumes that you have some familiarity with Linux. The instructions are based on the release of DECAF shown in the [svn](http://code.google.com/p/decaf-platform/source/checkout) or the [downloads page](http://code.google.com/p/decaf-platform/downloads/detail?name=DECAF.tar.gz#makechanges), running on a Ubuntu 12.04 distribution of Linux. We intermix instructions with explanations about utilities to give an overview of how things work.

## 1. Compile ##
  1. DECAF is based on QEMU. It's useful to have a vanilla QEMU for testing and image development. Also, you need to install some stuff to compile qemu/decaf.
```
  sudo apt-get install qemu
  sudo apt-get build-dep qemu
```
  1. Before compile, you need to install the following libraries.
```
  # For the BFD library:
  sudo apt-get install binutils-dev
  # For the boost library:
  sudo apt-get install libboost-all-dev
```
  1. Configure&Make
> > DECAF has three basic settings-TCG tainting, VMI, TCG IR logging. You can enable/disable at the configuration step. By default, VMI is enabled and TCG tainting and TCG IR logging is disabled.
```
  #go to the root directory of DECAF's source folder.
  ./configure
  make
```
> > Enable TCG tainting
```
  #go to the root directory of DECAF's source folder.
  ./configure --enable-tcg-taint
  make
```
> > Enable TCG IR logging
```
  #go to the root directory of DECAF's source folder
  ./configure --enable-tcg-ir-log
  make
```
> > Disable VMI.
> > VMI is enabled by default. If you disable it, DECAF cannot retrieve os-level semantics although you have guest driver installed because we have removed the    support for guest driver(TEMU does VMI this way).
```
  #go to the root directory of DECAF's source folder.
  ./configure --disable-vmi
  make
```
## 2. Create a new VM ##

> While QEMU itself is compatible with almost any guest OS that runs on x86 hardware, DECAF requires more knowledge about the OS to bridge the semantic gap and provide information about OS abstractions like processes. DECAF currently supports Windows XP, Windows 7 and Linux, in order to use our tool, user has to have a working qemu image of the operating system. As how to make a qemu image,please see [QCOW2 image Format](http://people.gnome.org/~markmc/qcow-image-format.html).

> It's very slow to install an image using QEMU. You can try VirtualBox to create a QCOW format virtual disk and install OS image. QEMU/DECAF can directly load that QCOW image without any modifications.


  1. If you have a vmdk image(used by vmware),you can convert it to qcow2 format by:
```
  qemu-img convert win2kpro.vmdk -O qcow win2kpro.img
```
  1. if you have a vdi image(used by virtualBox),yo can convert it to qcow2 format by:
```
  VBoxManage clonehd --format RAW img.vdi img.raw
  qemu-img convert -f raw ubuntu.img -O qcow2 ubuntu.qcow
```
  1. With the new VMI support, guest driver is no longer needed by DECAF.Image created from vmware/virtualbox may not work for the new VMI. SO it's better to create an image from install file or have vmware/virtualbox drivers uninstalled before converted to qemu image. If the VMI doesn't work, please see [discussion](https://groups.google.com/forum/#!topic/decaf-platform-discuss/SK6-HdMf6Dg).

## 3. VMI configuration for Linux ##
> The configuration is for Linux only. There is no extra configuration for Windows.
  1. Compile and insert kernel module
> > Copy procinfo.c and Makefile under `[decaf_path]/shared/kernelinfo/procinfo_generic/` into some directory in the guest OS.
```
  # in guest OS
  cd [path of procinfo.c and Makefile]
  make
  sudo insmod ./procinfo.ko
  # It's OK if you see "Operation not permitted" since this module only print some message.
  dmesg
```
  1. Update procinfo.ini
> > You will see something like this.
```
  [20451.579763] strName = 3.2.0-37-generic
  [20451.579764] init_task_addr  = 3246178336
  [20451.579765] init_task_size  = 3236
  ...
  [20451.579787] ti_task         = 0
```
> > Remove the leading time and brackets. Copy the rest content into `[decaf_path]/shared/kernelinfo/procinfo_generic/procinfo.ini`. Add a new section for it. Increment info.total at the beginning of procinfo.ini.<br>
<blockquote>e.g.<br>
<pre><code>  [info]<br>
  total = 7<br>
  ...<br>
  [7]<br>
  strName = 3.2.0-37-generic<br>
  init_task_addr  = 3246178336<br>
  init_task_size  = 3236<br>
  ...<br>
  ti_task         = 0<br>
<br>
  ;DON'T FORGET TO UPDATE info.total<br>
</code></pre>
</blockquote><ol><li>Possible errors<br>
<ul><li>UTS_RELEASE is not defined or utsrelease.h cannot be found. Define UTS_RELEASE as a unique string by yourself.<br>
</li></ul></li><li>Configuration for shared library (optional)<br>
<blockquote>To hook library functions, you need to configure the offsets. For each guest OS, there is a library configuration file for it in <code>[decaf_path]/shared/kernelinfo/procinfo_generic/lib_conf/]</code>. The file name of the configuration file is <code>[strName].ini</code>. In a library configuration file, there is a section for each shared library file. In each section, there is a decaf_conf_libpath field for the shared library file name. decaf_conf_libpath is the file name, don't include any path. <br> <br>
<b>decaf_conf_libpath = libc-2.13.so</b> <br>
<del>decaf_conf_libpath = /usr/lib/libc-2.13.so</del> <br> <br>
For each function, there is also a filed. The field name is the name of the function. The field value is the offset. The field name and value can be obtainted using the following command in guest OS. You may need to adjust the command according to your environment.<br>
<pre><code>  objdump -T [path_of_shared_library_in_guest_OS] | awk '/\.text/ &amp;&amp; $6 !~ /\(.*\)/ {printf("%-30s= %d\n",$7,"0x"$1)}'<br>
</code></pre></blockquote></li></ol>

<h2>4. Startup the Virtual Machine</h2>
<blockquote>Now you have get everything ready, the next step is to start the emulator and run the virtual machine.<br>
</blockquote><ol><li>Go to DECAF/trunk/i386-­softmmu<br>
</li><li>Run the following command<br>
<pre><code>  ./qemu-­system­-i386 -­monitor stdio ­-m 512 ­-netdev user,id=mynet ­-device rtl8139,netdev=mynet “YOUR_IMAGE”<br>
</code></pre>
</li><li>If you want to use snapshot function, you can use ­snapshot option</li></ol>

<h2>5. compile and load plugins</h2>
<blockquote>DECAF provides many interfaces to trace internal events of the guest operating system. You can write your analysis plugins using these interfaces. To learn how to write plugins, <a href='http://code.google.com/p/decaf-platform/downloads/detail?name=decaf_plugins.tar.gz#makechanges'>plugin samples</a> is the best place to start with. Download <a href='http://code.google.com/p/decaf-platform/downloads/detail?name=decaf_plugins.tar.gz#makechanges'>plugin samples</a>. Take callbacktests plugin as an example.<br>
</blockquote><ol><li>compile plugins<br>
<pre><code>  cd ./callbacktests<br>
  #set decaf location<br>
  ./configure --decaf-path=root directory of decaf<br>
  make<br>
</code></pre>
</li><li>load plugins<br>
<pre><code>  #start virtual machine, change directory to (root directory of decaf)/i386-softmmu/<br>
   ./qemu-­system­-i386 -­monitor stdio -­m 512 ­-netdev user,id=mynet -­device rtl8139,netdev=mynet “YOUR_IMAGE”<br>
  #check available cmds<br>
  help<br>
  #load plugins<br>
  load_plugin XXX/callbacktests/callbacktests.so<br>
</code></pre>
</li><li>trace program<br>
<pre><code>  #trace a specific program<br>
  do_callbacktests calc.exe<br>
  #now you can start calc.exe in the guest operating system to see the results. <br>
</code></pre></li></ol>

<h2>6. Troubleshooting</h2>
<blockquote>This section describes some problems users have experienced when using DECAF, along with the most common causes of these problems.If you have any other questions,please post them on <a href='http://groups.google.com/group/decaf-platform-discuss'>Forum</a></blockquote>

<ol><li><b>After start Decaf, the terminal says " vnc server running on 127.0.0.1 7 5900" and there is no running guest os shown up.</b>
<ul><li>This is because SDL library is not properly installed. You just need to reinstall SDL.<br>
</li></ul></li><li><b>When compile plugin, it can not find lcrypto.</b>
<ul><li>you need to make a link to lcrypto.xxx.so to lcrypto.so. If you can not find lcrypto.xxx.so library,just install ssh.</li></ul></li></ol>






