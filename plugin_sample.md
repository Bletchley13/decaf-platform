To demonstrate how DECAF enables and solves various binary analysis problems, we showcase three plugins.By hooking the entries and exits of APIs specified in a configuration file, API Tracer is able to trace the API invocations of a specified process and the pro- cesses spawned from it. Keylogger Detector keeps track of tainted keystrokes propagating in the OS kernel and across user-level pro- cesses to detect keyloggers. Instruction Tracer logs instructions executing within a specified context (such as a user-level process, or a kernel module). These plugins are mostly platform agnostic.

## API tracer ##
API tracer is a simple plugin that given a configuration file containing a list of APIs and their arguments, monitors the execution of a program and prints the execution of such APIs to a trace file.

To use apitracer:
  1. compile DECAF with vmi enabled.
```
./configure --enable-vmi
make

```
  1. compile the apitracer plugin.
```
 # cd to the plugin's source folder
./configure --decaf-path=root directory of DECAF
make

```
  1. use DECAF to start guest OS and load apitracer plugin.
```

 # start virtual machine, change directory to (root directory of decaf)/i386-softmmu/
 ./qemu-­system­-i386 -­monitor stdio -­m 512 ­-netdev user,id=mynet -­device rtl8139,netdev=mynet “YOUR_IMAGE”
 # load plugins
load_plugin path/to/apitracer/plugin/apitracer.so

```
  1. Set the program to trace.
```
 # A sample config file is included in the apitracer folder.
trace_by_name <program_name> <out_trace_file_path> <in_config_file_path>

```
  1. Inside the VM, start the program to be traced.
  1. To stop tracing,
```
trace_stop

```

## Keylogger Detector ##

Leveraging the VMI, tainting and event-driven programing features of DECAF, Keylogger Detector is capable of identifying keyloggers and analyzing their stealthy behaviors.

By sending tainted keystrokes into the guest system and observing if any untrusted code modules access the tainted data, we can detect keylogging behavior.The sample plugin can introduce tainted keystrokes into the guest system and identify which modules read the tainted keystroke by registering to receive DECAF_READ\_TAINTMEM\_CB and DECAF\_KEYSTROKE\_CB callback events. To capture the detailed stealthy behaviors, Key- logger Detector implements a shadow call stack by registering for the DECAF\_BLOCK\_END callback. Whenever the callback is triggered, we check the current instruction. If it is a call instruction, we retrieve the function information using VMI and push the current program counter onto the shadow call stack. If it is a ret instruction and pairs with the entry on the top of the shadow call stack, we pop it from the stack. When the DECAF\_READ\_TAINTMEM\_CB callback is invoked, we retrieve information about which process, module, and function read the tainted keystroke data from the function call stack._

To use keylogger detector
  1. compile DECAF with tcg tainting and vmi enabled.
```
./configure --enable-tcg-taint --enable-vmi
make

```
  1. compile keylogger detector plugin.
```
 # cd to the plugin's source folder
./configure --decaf-path=root directory of DECAF
make
```
  1. use DECAF to start guest OS and load keylogger detector plugin.
```

 # start virtual machine, change directory to (root directory of decaf)/i386-softmmu/
 ./qemu-­system­-i386 -­monitor stdio -­m 512 ­-netdev user,id=mynet -­device rtl8139,netdev=mynet “YOUR_IMAGE”
 # load plugins
load_plugin path/to/keylogger/plugin/keylogger.so

```
  1. enable keylogger detector. you can use "help" command to check the command supported by DECAF and keylogger detector.
```
   enable_keylogger_check LOCATION_OF_LOG_FILE
```
  1. turn on pointers tainting. Because guest os needs to translate scan code to ASCII code for every keystroke. This translation is table lookup operation. So we need to turn on pointers tainting. To avoid overtainting problem, we just turn on pointers read tainting.
```
enable_tainting
taint_pointers on off
```
  1. start your suspicious program and introduce a tainted keystroke into notepad.exe
```
taint_sendkey c
```
  1. when you see ‘c’ in notepad.exe,you can disable taintmodule check.
```
disable_keylogger_check
```
  1. now check the log to see if keystroke is fetched by your suspicious program. If yes, it's a keylogger malware.

## Instruction Tracer ##