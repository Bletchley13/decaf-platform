## When will we get x86 support ##

DroidScope has three layers of abstraction, NativeAPI, LinuxAPI and DalvikAPI. The difference between ARM and X86 is mainly present in the NativeAPI.

The NativeAPI consists of the memory read and write functions as well as low level callbacks - e.g. block begin, instruction begin and etc.

For the most part, the memory read and write functions are implemented in DECAF\_main.c and rely on QEMU's built in (target independent functions) which means they are target independent. The only target dependent memory function is memory\_read\_with\_pgd since x86 uses CR3 and ARM uses C2Base0 and C2Base1 (there are 2 PGDs which are selected and used based on masks).

To support the two architectures there are two different implementations in target-i386/DECAF\_main\_x86.c and target-arm/DECAF\_main\_arm.c.

Then there are the callbacks for the different events. Those are all implemented by inserting calls in specific locations within the translator TARGET/translate.c. Currently in the alpha only target-arm/translate.c has been patched (you can take a look at the changes and where the changes are by searching for the "START DECAF ADDITIONS " strings.) In essence to port the same event callback code to x86 requires making similar changes to target-i386/translate.c

I should point out that DECAF parent (the main project) already have these changes made - plus more events. So it is up to you whether you want to make the changes now or wait until we upload DECAF alpha.

Ohh, I should also mention that there are similar target-specific requirements for the DalvikAPI. For example getDalvikGLUE returns register [r6](https://code.google.com/p/decaf-platform/source/detail?r=6) in ARM and looks up the pointer on the stack. The target specific implementations are in DS\_main\_arm.c and etc. I haven't fully ported my old x86 code over yet so that needs work too.

## When will DS support Jelly Bean? How can I get Jelly Bean support? ##

DroidScope relies on the C structure definitions in vm/oo/array.h, object.h and etc. to interpret the Java Objects. (By the way this is why we require a 32-bit host. Those structures contains pointers and compiling on a 64-bit host would mean that the structure layouts are no longer valid and thus we can no longer do Java object introspection correctly) So getting Jelly Bean support would involve integrating the same files from Jelly Bean into DroidScope and then updating the code to reflect those changes (perhaps you don't need to update the code because the code relies on the structure definitions ??? I don't know the answer to this question). If you want to tackle that problem then go into the files in the dalvik directory and take a look at my comments. I put a "list of changes" in the front of each file to help you do this.

The other "potential" problem is in changes to the MTERP interpretter and JIT. I don't think this should be a major problem though. I would also like to point out that since newer versions of Gingerbread uses the odex file (while the older version uses classes.dex) the latest alpha release of DroidScope should be okay as well.

## I used a different kernel and ps, pt, pm doesn't work anymore ##

DroidScope relies on offsets for the linux introspection. You can look at the comments within read\_linux.c and kernelinfo.conf (inside the objs directory) for more info. The basic idea is that there are certain fields within the kernel data structures task\_struct, vm\_area\_struct, and etc. that we need. To obtain the offset of these fields, you will take a look at the structure definition note the relative offsets of the different fields (#ifdefs make it difficult but its not a big problem), find a field that you can get the offset for statically and then start to populate the configuration file with the different offsets.

For example, in 2.6.29 the task\_struct has the following fields

struct list\_head tasks;
struct mm\_struct **mm,**active\_mm;
//other stuff
pid\_t pid;
pid\_t tgid;


Basically the idea is that if you have the offset of either tasks, mm, pid or tgid, you can figure out what the offsets of the other fields are. And it just happens that there is an exported function "get\_task\_mm" which given a task\_struct returns the mm field. So you can find the offset by looking at the kernel binary that you are trying to use.

I would like to point out that DECAF parent has an automatic way of doing this as well.

## When I compile QEMU I get the followin error ##

I followed the instructions on the wiki page except when compiling Qemu.
> When doing 'make', it showed
> > ld: cannot find -lgthread-2.0

> The command that leads to this error is
> > ./../../prebuilt/linux-x86/toolchain/i686-linux-glibc2.7-4.4.3/bin/i686-linux-gcc -pthread -lgthread-2.0 -lrt -lglib-2.0    -rdynamic  -o objs/emulator   objs/intermediates/emulator/android/main-emulator.o objs/libs/emulator-common.a -pthread -lgthread-2.0 -lrt -lglib-2.0    -rdynamic -m32 -lm -lutil -lrt -lpthread

This problem is likely caused by the "prebuilt" bug. Basically, I copied the QEMU directory (i.e. external/qemu) elsewhere to build the DS and when I did that, the "prebuilt" check in android-configure.sh would just fail nicely and as a result, the system's gcc environment will be used instead.

If the directory was not moved out then, what you will find is that the prebuilt compiler is being used - which causes problems. The telltale sign that the wrong compiler is being used is the prebuilt path as shown above:

./../../prebuilt/linux-x86/toolchain/........

To fix the problem, you should go into the android-configure.sh script and disable the check for prebuilt components (i.e. comment the lines out) or use the --no-prebuilts configuration option.