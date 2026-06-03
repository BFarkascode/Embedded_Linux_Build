# Embedded_Linux_Build
A walkthrough on how to have our own embedded Linux using Buildroot and Yocto.

## General description
Something I have often asked myself was: why? Why bother with embedded Linux or any complex operating system as an embedded system engineer? Isn’t slapping an OS on a solution just simply there to turn a low level problem into a high level one…i.e., to allow software wizards to use their python skills instead of using C?

Well, sort of…and definitely there are pros and cons for jumping onto the Linux train. And yes-yes, a skilled embedded system engineer should be able to choose his infrastructure well, using the right tool for the right job.

### Embedded Linux
Anyway, what is even embedded Linux?

Linux is an advanced open-source operating system that could go on-par with wide spread computer-based operating systems, such as Windows or Mac OS when using the appropriate tools…and that’s where the key capability of Linux lays: we don’t have to run with a package that is given to us and instead, we can tailor it to whatever needs we have. No bloated drivers, no bloated user interface if we don’t want it why still having the same kind of organisational and infrastructure-management capabilities from a terminal that we have associated with an operating system. Of course, we can have all those goodies too if we wish by running some of the distributions such as Ubuntu or Debian. Linux also allows highly portable code, meaning that what is written to one device will be applicable to another one running Linux. It removes the complexity of understanding the hardware when writing an application…that is, as mentioned above, turning any problem into a software problem. It allows us to implement of a lot of code and drivers in our system without us bothering ourselves with it.
Anyway, in some way, a minimalistic embedded Linux is just a Zephyr RTOS with a terminal and file-based data management added to it. The comparison doesn’t just stay there since Zephyr uses Linux kernel for the devtree and the hardware config (making me think that the entire point of Zephyr was to strip even the terminal of embedded Linux).

But then what can embedded Linux do to us that Zephyr can’t? Well, that is a valid question and considering the general turn by the industry towards Zephyr, we might have the answer: from a system point of view, not much, unless we want to run complex code. In other words, we need to use embedded Linux – and, by proxy, python or java – because most advanced code libraries demand it, but, in reality, for simple tasks – and for real real-time applications – microcontrollers are the way to go - with or without a scheduler. There is a reason after all, why most modern devboards have a microcontrollers (for RTOS) and a microprocessor (for Linux) on them…

In short, we need embedded Linux since nobody bothered to port the complex python libraries to Zephyr yet.

### MPU vs MCU
 Okay, so now we made sense of why we need embedded Linux…but we have an MCU for RTOS and an MPU for Linux?
 
First and foremost, an MPU is ”assumed” to be only a computing unit while an MCU is the entire package. This generally makes an MPU faster while sporting roughly the same package size. On the flip-side, an MPU would not be able to do much without external memory and peripherals attached to it.

The distinction between MCUs and MPUs has become a lot murkier lately with certain MCUs capable to clock up to a few hundred MHz and MPUs coming with certain peripherals already added to their form. Some of the chunkier ST MCU devboards such as the STM32F746-Discovery is shown to be able to run embedded Linux after all, so it isn’t a tall order to assume that MCUs and MPUs will become one and the same in the near future. In general, we could still say that MPUs are better for multitasking. It is also possible to run an MPU without Linux, such as has been demonstrated by Zephyr so it isn’t just that MCUs go more advanced, MPUs can potentially go simpler, if needed.

Anyway, Linux generally demands a lot of resources/overhead – especially memory – that would generally not be possible with an MCU. Most embedded Linux projects have been built with the “legacy” MCU-MPU distinction though, so if we are to run embedded Linux on a hardware, we ought to pick up an MPU board like the STM32MP157 or the BeagleBone Black (or the most known one, the Raspberry Pi). These devboards come with an MPU on them, multiple external peripherals to communicate with USB or ethernet, plus a sizeable RAM to provide the resources necessary to run the OS. As ROM/system hard drive, they both use an SDcard.

Lastly, we will be able to communicate with them through the serial interface.

Lastly-lastly, we will have to set the boot selection for the boards if we want to use the SDcard  as the boot source. For the MP157, those will be the switches on the back of the board, for the Beagle, it will be the Boot button on the front (must be pushed when applying power to the board, resetting will not activate it). Mind, the Beagle comes with a pre-installed Linux Debian distro stored in local ROM that will be used to run the board should we not set the boot selection up properly.

Below we will be building an OS for both boards, but not for the Pi.

 ## Git it dun’
Congrats, so you have made the decision to run embedded Linux. What are you to do next?

Well, the same way as you would handle a Raspberry Pi, you could get a pre-set SDcard from your devboard provider that already has the OS set up for you (all the peripherals, clocking, user interface, memory manager, file system and so on).

Alternatively, you could build your own OS and select yourself, what you need. You do need to have a system running Linux already.

IMPORTANT!!!

I do recommend getting an older version of whatever Linux distro is the most up to date though since there seems to have a rather annoying delay between the newest distro’s launch and the packages catching up with the changes. This means that even if officially things should work, some packages may not play nice with each other, thumbling you into a whole cascade of error messages. For example, I am using Ubuntu 22.04 since that is the last version that is proven to work with all the packages. Later Ubuntu versions come built-in with python 3.12 or later for instance which are not compatible with some of the code we will be using for our build process due to depreciation. It is not obvious to roll back to python 3.10 though.

Lastly, before we take a deep-dive, whenever I am writing in FULL CAPITAL something in a line of code, that section has to be adjusted to whatever element our code is addressing, be that the machine we are building for or a peripheral we want to interrogate.

Lastly-lastly, we will have to remove the “tmp” folder (“rm -rf /tmp”) in our build to properly remove a build and start from scratch. This is recommended when transitioning from one board to the other since this folder may take up to 30 GB of storage.

### Buildroot
Buildroot (or vendor tools) will allow someone to build their own OS with minimal headache. These tools are specifically tailored to put together the OS for us (or, more precisely, the image file we have to put on the SDcard) and give a wide berth of options to choose from. We can select the kernel, we can add packages, drives, patches and so on. It is probably the fastest and easiest way to get our own official distro without doing any of the legwork.

Buildroot itself is compatible with a lot of devboards (including the two we have as test subjects) and if we aren’t too keen to go deep into what is going on with the Linux build, it would do a perfectly acceptable job.
There is one pickle though: running buildroot takes hours since it rebuilds the image from scratch every time. This is obviously not an option if we are developing something for our custom Linux build and would want to update it regularly doing testing and debugging.

No…to make that feasible even remotely, we will need to do the build…manually. We are here for the pain though, so…yeah!

### Yocto Project
Yocto is an umbrella project holding together a lot of different elements (such as the work done by OpenEmbedded or “oe”) to implement a custom embedded Linux build. It is an official Linux project with ongoing support (with some delay on updating).

Now that we have that clear, let’s touch upon a few recurring elements:
- Reference distribution: Poky is a reference distribution in a sense that it provides us a baseline recipe and metadata to manipulate for our build. It includes all the drivers, code, and tools that we will need. The “poky.conf” file will show, which Linux distribution we can use for the build with the immediate branch (for the “dunfell” branch, it is Ubuntu 22.04 ). Mind, we don’t need to use poky, we can instead use another reference, such as “kas”.
- Bitbake: Bitbake is the tool that will read in all the data and construct the image. It is coming with poky.
- Layers and metadata and recipes: The build system calls folders “layers” and files “metadata” which “recipes” being special configuration metadata about how the build should proceed. Most layers are given the name “meta-”. 
- BSP layer: we will need to have a specific layer for our board (board support package or BSP) included in the build to set the hardware up properly, similarly to how ST provides BSP files to run the sensors on its devboards. For the mp157, this will be the “meta-st-stm32mp” layer, for the Beagle, it will be “meta-beagleboard”. We will have to manually include these in our builds. For a custom board, we will need to have a custom BSP file (or patch an existing one, as we will see later). One should be very much aware of the dependencies of these layers, otherwise the build will not go through (they should be written in the layer’s README file often with links for the download as well included in the description). The layers which are known to work can be found here: OpenEmbedded Layer Index - layers
- Machine: we will also have to define the “machine” for the build to look for these BSP files.
- Image: an “image” will be the output of the build, what will have to be put on the SDcard.

Lastly, the list of libraires we need to tun Yocto is this:

sudo apt install -y bc build-essential chrpath cpio diffstat gawk git texinfo wget gdisk pyhton3 python3-pip bash-completion libssl-dev zstd liblz4-tool

IMPORTANT!!!

1)	Yocto takes a MASSIVE chunk of storage on your computer. Despite what the manuals and everything says, for me, it was not enough to have only 50 GB of free space to run Yocto. As a matter of fact, with Ubuntu 22, even 100 GB was not enough…
2)	The “Scarthgap” branch while build properly has resulted in a boot file system that would not mount properly on my devices. More precisely, the bootfs partition was shown to be empty. This turned out to be a build error since, when switching to “dunfell”, the problem solved itself.
3)	BB configuration files are NOT C syntax! For instance, use the appropriate commenting when adding text to them (“#” instead of “//”).

### First build
Let’s just discuss a few notes:

1)	All bits and pieces that we are using must be compatible with each other, that is, we must use the same branch for all of them. The general yocto release are listed here: Releases - Yocto Project. Use “git checkout BRANCHNAME” to set the branch.
2)	We need to source the poky environment to make bitbake work and to add everything we need to the path. This sourcing must be done every time, though only the first time we need to give the command the name of the build folder (if we want a custom build folder, that is). Use the line “source poky/oe-init-build-env BUILDFOLDERNAME”. This must be done each time we are working in a separate build folder, otherwise bitbake will not know, which place to look for (and will just build a standard “poky” distro in a standard “build” folder) or create a phantom folder somewhere with a build we didn’t actually want to have.
3)	Bitbake may be blocked from execution by the apparmor in newer ubuntu versions. If we choose to remove this block temporarily, it will have to be redone every time we run bitbake, otherwise we will have an error message. Use "echo 0 | sudo tee /proc/sys/kernel/apparmor_restrict_unprivileged_userns" to temporarily remove the apparmor block.
4)	A menu system very much reminiscent to buildroot can be pulled up by calling bitbake with “menuconfig”.
5)	The first bitbake execution may take a few hours, Later ones will be significantly faster, depending on how much in the build construction has been modified.

Regarding steps to take to run yocto:

1)	Get all the layers and their dependencies. Set them all up to be on the same branch.
2)	Source bitbake (and remove any of the blocks on it)
3)	In the generated build folder, check the layers that are currently included in the build recipe (“bitbake-layers show-layers”).
4)	Modify the “bblayers.conf” file in the build’s “conf” folder to have the other layers you need for your build to succeed. Recheck the layers as seen above once done.
5)	Double-check the machine name in the bsp layer’s “MACHINENAME.conf” file (should be in “conf/machine”) and the dependencies (should be the same as the README above)
6)	Modify “local.conf” file in the build’s “conf” folder to target to the right machine
7)	Execute bitbake with the selected recipe (see here for examples: 12 Images — The Yocto Project ® 5.3-tip documentation). Core-image-minimal is the simplest yocto recipe (yocot Linux distribution).
8)	To start over completely, remove the “tmp” folder in the build directory. To simply clean the existing build, call bitabke with “cleanall SELECTEDRECIPE”

Of note, we can check the manifest file – “NAMEOFTHERECIPE-MACHINENAME.manifest” – to understand, what packages have been included in our Linux build. The core minimal should have only about 24 packages.

#### Boot sequence
Let’s do a little sidestep first and explore, how Linux boots.

As the device powers up, we will have a ROM section that will check for some basic boot options and look for memory storage. Practically speaking, this is the phase where the device checks the state of the boot pins on the device (just to remind, these are the two switches on the back of the MP157 and the boot button on the Beagle). Of note, many devices come with local storage that can already run some form of basic Linux as a back-up.

The following step is the initiation of the first stage bootloader (fsbl). Of note, this part is still very hardware specific. What they are and how many of them there are could also depend on the bootchain we are using. Anyway, the fsbl does basic initiation of clocks, memory and storage of the MPU. It also calls the next phase.

The next is the second stage bootloader (ssbl) which will usually be U-boot or Grub, loaded in by the fsbl from fip (firmware image package). This stage initialises the board itself and looks for/loads into RAM the Linux kernel. Mind, the second stage bootloader can usually be interacted with directly through a serial port. It can also have FTP network access with a PC host to allow easier updating. It also allows setting up a kernel on the host and then make u-boot boot the target machine from this using the FTP.

Then comes the Linux kernel itself, which will be the initiation of Linux with its drivers and libraries. The devtree is here. The kernel will be able to change the configurations that were set by the two previous stages. The kernel is cross-compiled by the toolchain on the host device to our specific machine.

Lastly, we will mount the rootfs which will be the Linux root partition.

Mind, for security reasons, it is VERY much recommended to define a vendorfs partition for third party code and userfs sandbox partition. Users will only have access to their own sandbox when setting up the boot, which will protect all sensitive code from accidental corruption. Userfs and vendorfs are not necessary though to set up a bare-bones Linux.

IMPORTANT!!!!
From ssbl and above, we will have a serial output from the MPU with which we can interact or read to look for bugs. This is particularly useful when attempting to debug the boot process and find out, why and where it has collapsed.

For some more info, check the STM32MP1 Platform boot (mind, this is just the MP157 boot, other machines may have a different sequence), the file is called "STM32MP1-Software-Platform_boot_BOOT.pdf".

#### Setting up the SDcard
Normally, we have a one-and-done image file that we can just copy on the SDcard and everything will be set for us. Yocto on the other hand, does not give us one image file but multiple ones.

Coming back to the results we have acquired by running bitbake, as we can notice when digging into the “tmp” folder, we will have a whole bunch of images generated by the build (exact path may vary, just look for the “SELECTEDRECIPE-MACHINENAME.ext4” file or files with the name “bootfs” or “rootfs”).

Now, to make sense of these, we will have to find the SDcard layout document. This should be a “tsv” document generated by the build that will specify for us precisely, what we need to do with these files. More precisely, it will tell us, how we need to partition our SDcard and what we need to put on each partition (see the column “Binary”). It also will tell, what name we need to give the partitions. This file must be followed religiously, or the boot will fail. Mind, the layout file may vary depending on the machine used or the branch of the build, so one should always find the one made for the particular build.

The trick here is multiple.

-	First and foremost, just because one bitbake branch build asks for one partition setup, another one may ask for something different. To be more precise, the “dunfell” build that is done within the Digikey training (and the one we will do here) will be different compared to building in “scarthgap”: we will have additional partitions to set and the naming will be different as well. As such, one MUST always check the layout file for each exact build, otherwise the SDcard may not work.
-	The second trick is that the partition naming can be very strict. How the booting process is scripted (see above) is that each phase will search for a certain name or a certain UUID of a partition and, if it does not find it, it will fail. We will have to change the UUIDs and/or the names of the partitions in case the generated image is hard-coded to do as such. The usual names are fsbl1 and fsbl2 for the first stage, fip for the ssbl, bootfs for the boot and rootfs for the root. If needed, we will likely have to find an “extlinux.conf” file somewhere on bootfs and check what the rootfs UUID should be (if the UUID is not a match, the kernel will not find the root). The same might need to be done for the userfs and vendorfs partitions, but I have not tested those.
-	The third trick is that each partition should have a particular file system type and, potentially, additional activation flags. More precisely, the first stage bootloader partitions (both) and the fip partition should all be simple binary partitions, while the bootfs and rootfs ones must be Linux filesystems with bootfs also having the “bootable” flag added to it.
-	I am not sure about the exact sizing of the partitions if they must follow the offset demanded by the layout file (according to the ST documentation, no), nevertheless they must be big enough to hold the files we will transfer to them. The fsbl first stage bootloader partitions (naming is fsbl1 and fsbl2) must be at least 256kB, the ssbl second stage bootloader (naming is usually fip or fip-a) at least 2MB, the bootfs at least 64 MB and the rootfs the rest.
-	The card type must be set to “gpt”.
-	
We can then use “fdisk SDCARDDEV” to process the card. The SDCARDDEV can be extracted from “lsblk”. Mind, the Sdcard may have to be mounted/imported to Linux first.

For more on the SDcard populating, see here (but don’t forget to check the branch the proposed build applies to): https://wiki.st.com/stm32mpu/wiki/How_to_populate_the_SD_card_with_dd_command

## To read
I am mostly going along the Shawn Hymel training on embedded Linux, albeit he only does it for the MP157:

Introduction to Embedded Linux Part 1 - Buildroot | Digi-Key Electronics

He technically seems to follow this particular lab here, in case someone prefers a more text-based guide compared to a video:

Yocto Project and OpenEmbedded development training – Bootlin

I also suggest checking this video training too to get a feel for more complex actions and how Yocto works:

Intro to Yocto I wish I was given | Part 1: The Basics - YouTube

A very detailed embedded Linux build can be found below as well for multiple board types. Just a note, it is using existing data for the compilation process, not creating everything from scratch and pretty much does the exact same thing what we do using “bitbake”, just doing everything by hand. It is a worthwhile read since it explains the build process very well:

Embedded Linux training – Bootlin

Lastly, the reference manuals for the devices we intend to use.

## Particularities
I was intentionally using generalised speech above to avoid pointing to any specific board in the project. Nevertheless, the mp157 worked only using the “dunfell” branch as shown by the digikey training.

For the Beagle, the situation is a bit more complex since the last board update was on “danny” (last update is from 2014). Also, documentation has become rather lacking in the last few years, making it very difficult to figure out, what one should do to make a custom distro for the Beagle. One alternative is to just use the yocto example for the Beagle (select “beaglebone-yocto” as the machine in the “local.conf” file, dependencies are only on the already included yocto layers so no need to adjust the bblayers file) or make everything manually by finding the BSP is called “meta-beagle” or “meta-ti” (an official TI layer on github with multiple dependencies, check the readme files) and then build for the “beaglebone” machine. It thankfully will run on “dunfell” as well. Of note, while the build went through with this approach as well, there was no layout file generated, making it practically impossible for us to construct the SDcard. There is some outdated information on the official BeagleBone site on how to do it, though it didn’t quite match with what I have managed to generate using “dunfell”. Another issue comes with serial communication: while the beagle’s in-built Debian will communicate through the USB with the host, our own embedded Linux will have to be talked to using an external FTDI converter (a USB-serial bridge).

Anyway, due to lacking documentation and low activity on the topic in the forums plus the unnecessary complexity from reading the output from the Beagle, I have decided to carry on with only the MP157 instead. I am thus sharing only the two build configuration files for this machine.

Apart from these two config files for the build (which are shared), everything else is provided by Yocto.

## Conclusion
With any luck, the first build above will result in a terminal prompt running on the device with which we can interact via serial connection (I use TerraTerm on Windows). This will be a simple build, a “minimal” one as put by the poky reference distribution which will not have any drivers activated, no additional peripherals, only what is set as a baseline by the boot sequence.

We will change that in the next project.

