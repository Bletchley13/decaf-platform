DroidScope (DS) is a dynamic analysis platform based off of the Android Emulator.

Please see the original paper for more information on the concept.

https://www.usenix.org/conference/usenixsecurity12/droidscope-seamlessly-reconstructing-os-and-dalvik-semantic-views


## DECAF DroidScope ##

Since the original paper, we have been porting DS to run on top of the DECAF binary analysis platform the immediate advantages are:

# Seamless ARM and x86 Native API support
# Dynamic loading of plugins
# More refined NativeAPI - more callbacks, and some bug fixes
# Better Virtual Machine Introspection support

NOTE: DECAF is based off a newer version of QEMU, however the core changes to translate.c, monitor.c and etc. are the same.

## DroidScope Getting Started ##
We provide **[a Virtual Box image](http://sycurelab.ecs.syr.edu/image/DroidScope.tar.gz)** which has DroidScope installed. You can use that directly for convenience. If you wanna compile it, please see the following instructions.

The current version of DS is built for Android Gingerbread. Since we mainly deal with the 32-bit ARM architecture and we include some files from the Android source code as part of DS we require that the host machine be a 32-bit machine. DECAF does not have this limitation.

Thus far, DroidScope has been tested on Linux Mint 14 Nadia, but it should also work for Ubuntu 12.10 since Nadia is based off of it.

### Compiling Android ###

#### Jelly Bean 4.3 ####

From a fresh install (I used Xubuntu 12.04.3 AMD64 in this case)

Please follow the directions on the android open source project for package requirements. Here are the commands I ran (they should be the same or at least very similar).

1. Get the necessary packages and set up the magical link for libGL

sudo apt-get install git gnupg flex bison gperf zip curl libc6-dev libncurses5-dev:i386 x11proto-core-dev   libx11-dev:i386 libreadline6-dev:i386 libgl1-mesa-glx:i386   libgl1-mesa-dev g++-multilib mingw32 tofrodos   python-markdown libxml2-utils xsltproc zlib1g-dev:i386 libsdl-dev

sudo ln -s /usr/lib/i386-linux-gnu/mesa/libGL.so.1 /usr/lib/i386-linux-gnu/libGL.so

2. Download the Java SDK from Oracle (Remember to get Java 6 as prescribed by AOSP). You can install it into the proper directories - I just unzipped the tarball and then changed my PATH and CLASSPATH to the appropriate directories.

PATH=/home/lok/Downloads/jdk1.6.0\_45/bin:$PATH #MAKE sure you DON'T have a / after bin, that causes lots of headaches
CLASSPATH=~/Downloads/jdk1.6.0\_45/jre/lib/:~/Downloads/jdk1.6.0\_45/lib/ #not sure if I really needed to do this, but it worked with it so it can't hurt right?

3. Download the Android Source. In my case I downloaded the latest and greatest (at the time)

repo init -u https://android.googlesource.com/platform/manifest -b android-4.3\_r2.2
repo sync

4. Build Android

source build/env-setup.h
lunch full-eng #that is the one I tested with
make -j4 #build android

5. Test Android

emulator #yes.. emulator will run the just built avd

#### Gingerbread ####
One of the biggest difficulties in compiling Android in newer distributions is GCC 4.7. The following are some directions for building the Android source on Nadia

From a fresh install

First – Install the necessary packages:

sudo apt-get install git-core gnupg flex bison gperf dpkg-dev gcc-multilib build-essential   zip curl libc6-dev libncurses5-dev:i386 x11proto-core-dev   libx11-dev:i386 libreadline6-dev:i386 libgl1-mesa-glx:i386   libgl1-mesa-dev g++ g++-multilib mingw32 openjdk-6-jdk tofrodos   python-markdown libxml2-utils xsltproc zlib1g-dev zlib1g-dev:i386 libgl-dev

Second – Grab the source (http://source.android.com/source/downloading.html)

Third – Download and apply the patch (https://haven.ecs.syr.edu/~lok/AndroidGingerbread_GCC47.diff)

Fourth – Either do an update on QEMU to get to version 12 (http://source.android.com/source/known-issues.html) or download and apply the patch (https://haven.ecs.syr.edu/~lok/AndroidGingerbread_QEMU_12.diff)

Fifth – Build the source according to the directions (http://source.android.com/source/building.html)

### Compiling DroidScope ###

DroidScope is integrated DIRECTLY into the android emulator source in "external/qemu" and we build DroidScope in the qemu directory directly. In order to build qemu as a standalone we must first install some more packages.

sudo apt-get install libglib2.0-dev libasound2-dev libpulse-dev libesd0-dev binutils-dev binutils-multiarch

Once that is done, then download the patch from the downloads section and patch the source.

Once patched run the configuration script (e.g. ./android-configure) and then run make to build it.

#### Android 4.3.2.2 ####

There are more changes to the qemu emulator in Android 4.3.2.2

First. All of the changes are made to the copy of the source in external/qemu. The main reason is because android-configure.sh is written to base some include directories using relative directories and prebuilts which is nasty. I will deal with them later. So, right now we will just keep the qemu source where it is.

Second. We must built DroidScope with the --try-64 option. The main reason is because DECAF (DS) requires libglib2.0-dev but unfortunately, libglib2.0-dev is a 64-bit package (we need a 64 bit host machine for AOSP) and if we tried to build a 32-bit executable (i.e., without the --try-64 option which results in the -m32 flag) we will get an error like

/usr/bin/ld: cannot find -lgthread-2.0
/usr/bin/ld: cannot find -lglib-2.0


Then, trying to install libglib2.0-dev:i386 gives me a very serious warning. So I chose not to install it. Online forums (http://ubuntuforums.org/showthread.php?t=1998047) seem to say that you can install it, but I am too afraid of breaking the system. The list of proposed changes to the system is simply too long.

### Running DroidScope ###

To run DroidScope simply go into the "objs" directory and run the emulator. The command that I have used is:

./emulator -sysdir ~/scratch/android-gingerbread/out/target/product/generic/ -kernel ~/scratch/android-gingerbread/prebuilt/android-arm/kernel/kernel-qemu-armv7 -qemu -monitor stdio

Make sure you "touch ~/scratch/android-gingerbread/out/target/product/generic/userdata-qemu.img" to create that file, otherwise you will get an error.

Running the command will bring up the qemu command prompt. Typing "help" will reveal the commands with the DECAF commands on the bottom. The commands that you should see are "ps", "pt" and "pm".

"ps" will show the shadow process list. You should be able to run that directly.

There is an additional command called "get\_symbol\_address" which can be used to determine the guest virtual address of a particular symbol.

For example, "get\_symbol\_address 62 "/lib/libc.so" "printf"" should return the address of printf.

If you run the command you will likely receive 0xffffffff as the address. This is because the symbol database has not been populated yet.

To do this you need to run the "./setupdumps.sh" script. The script does a number of things

# Download the libraries from the device into the "dumps" directory
# Download the apps from the virtual device into the "dumps" directory
# disassemble all of those files
# parse them to obtain the symbols

Run the script and follow the messages to fix any errors.

Since the symbols are loaded when the libraries (within the shadow module list) are first discovered, you will need to kill the emulator and restart it in order to see "get\_symbol\_address" work.

### Plugins ###

While plugins used to be built into the emulator they are now built as a separate dynamic library.

The plugins are located in the "DECAF\_plugin" directory. Currently only the DalvikInstructionTracer is functional.

To build the DalvikInstructionTracer, you will need to first configure the plugin. Make sure to specify --target=android (e.g. ./configure --target=android --decaf-path=~/scratch/qemu)

Once configured, it can be built using make.

To load the plugin you will need to go back to the qemu prompt and use the "load\_plugin" command. Note that tab-completion works so it might be a good idea to use it.

For example, we can load the plugin using: "load\_plugin ../DECAF\_plugins/DalvikInstructionTracer/dalvikinstructiontracer.so"

It should return a message stating that it was loaded successfully.

Once loaded another try at the "help" command will reveal additional commands, in particular the "trace\_by\_pid" command.

You can try this new command (which is provided by the plugin) by first starting "calculator" in the virtual machine and then running "ps" at the qemu prompt to find out the pid and then execute the command: "trace\_by\_pid 345" where 345 is the PID of calculator2.

If successful you should see a couple of 0's. Go back to the virtual device and interact with the calculator to see a bunch of log entries.