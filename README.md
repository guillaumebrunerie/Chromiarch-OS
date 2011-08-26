Chromiarch OS
=============

Description
-----------

Chromiarch OS is a way to have both Chrome OS and Arch Linux running simultaneously on
a Samsung Chromebook. Chromiarch OS consists mainly of this documentation together with
some useful scripts.
A video showing Chromiarch OS will be uploaded soon.

Drawbacks
---------

Here are a few drawbacks about using Chromiarch OS instead of Chrome OS or Arch Linux:

* Your device has to be in developer mode. This means that you will have to press Ctrl + D
at every boot and that boot time will be increased by a few seconds (don’t worry, boot
time will still be less than 15 seconds).
* The synaptics driver is used for the touchpad instead of the syntp driver. In
particular, two-finger drag&drop does not work anymore (but two finger scrolling is much
easier with synaptics).
* Screen dimming is disabled, so your screen will not be powerd off anymore when you are
not using your computer (but suspend still works).
* Chrome OS autoupdates will break Chromiarch OS. But this is easy to fix, after an
autoupdate Chrome OS will restart without Arch, you just have to run the script
"chromiarchos_restore", reboot, run "chromiarchos_restore" again and reboot, and this is
fixed.
* The SSD is small. At best you can give 9 or 10 Gib to Arch.
* The keys from the upper row are the functions keys F1 to F10. This means that you will
not have F11 nor F12 and that changing brightness or sound volume will not work in Arch
(unless you remap them in your window manager, but you will loose functions keys this way)
* Arch will run under Chrome OS’ linux kernel, so if you want a custom kernel, you will
need to recompile Chromium OS’ kernel.
* If something goes wrong and your computer does not want to boot, you’re almost screwed.
You can either run recovery mode, but this will erase completely Arch and all your data
(except Chrome OS’ data which is in the cloud), or build Chromium OS to have a bootable
USB key and repair your computer from here (but building Chromium OS is complicated).
You can also perhaps use prebuilt binaries, like Hexxeh’s builds, I guess it should work.

Quick explanation
-----------------

If you are already a Chrome OS hacker, here is a very short outline of what will be done
to make Arch run inside Chrome OS. Don’t worry, a much much more detailed explanation will
follow.

You will switch your device to developer mode, create a new partition for Arch, install
Arch in VirtualBox and copy it into the new partition. Then you will write a new upstart
job (Chrome OS uses upstart) to run a getty chrooted into Arch Linux. This getty will
then start a second X server (in the chroot) on login.

Notes
-----

Before beginning, a few notes:

* This is legal. Unless Apple or Sony, Google let people do whatever they want with their
computer. You will not need to do anything illegal.
* This is reversible. Whatever you do, at any time you can run recovery mode and you will
come back to factory state. This will not void your warranty at all.
* This is not straightforward (well, everything is relative), if you don’t know what a
shell is, it will probably be difficult to follow. But I’ll try to keep the requirements
low, in particular you don’t have to know anything about Chrome OS.

Developer mode
--------------

OK, let’s start.

When you buy a Chromebook, it is in release mode. This means that you don’t have access to
a shell, that your root filesystem (the /) is mounted read-only and cryptographically
signed by Google. So even if you find a way to have a shell and find a way to modify
something, your computer will not boot anymore (unless you also have Google’s private
key). This mode is intended for normal people using Chrome OS normally, to make viruses
very difficult to do.

As you can guess, there is another mode (called developer mode) which will allow you to
disable those protections. You can switch to dev mode with the dev mode switch present on
your device (for the Samsung Chromebook, a photo is available [here](TODO)).

Shut down your computer, flip the dev mode switch and boot your computer. You will see a
screen with a sad computer asking you to press space because Chrome OS is broken. This is
a lie! Chrome OS is not broken, you just have to press Ctrl + D to boot Chrome OS (you
will need to do this every time your computer will boot, this is the drawback of dev
mode). This screen is intended to normal users having accidentally flipped the dev mode
switch. Such a user will most likely either switch off the dev mode switch, run recovery
mode or call technical support, so will not run a potentially insecure operating system.

During the first boot in dev mode, Chrome OS will slowly erase the stateful partition
(more on partitions later) so you will have to wait 5 / 10 minutes. You will lose
everything that was there: list of last open tabs, downloaded files, offlineness of
offline-capable apps, cookies, remembered users and Wi-Fi networks, etc. You will not lose
your data which is in the cloud of course, that’s the whole point of Chrome OS.

You should now be in Chrome OS with the dev mode switch on. You have now access to a full
screen console in tty2 (with Ctrl + Alt + ->, because -> = F2), you have the kernel
messages on tty8 (F8 = mute), and you can have a graphical terminal emulator: press Ctrl +
Alt + T to open crosh and use the "shell" command. You can either connect as root or
connect as chronos and use "sudo". But for now / is still mounted read-only and
verification of cryptographic signatures is still active.

Partitions
----------

Before going on, I should explain the (non standard) partitionning system of Chrome OS.

Unlike many computers, Chrome OS uses GPT partition tables. This is a "new" partitionning
system replacing the old MBR, without restrictions on the number or the size of partitions
(with the MBR you can only have 4 primary partitions, if you want more you need to use
extended and logical partitions). We will use the cgpt tool (present in Chrome OS) to
tweak partitions.

If you run "sudo cgpt show /dev/sda" you will see the partitions present on your drive.
The output should be something like this (size and positions are expressed in 512 bytes
blocks):
TODO

There are a lot of partitions here. Let’s take a look at the more important partitions.

KERN-A (/dev/sda2) and KERN-B (/dev/sda4) are kernel partitions. They contain the Linux
kernel and its signature.  Those partitions are not formatted with any filesystem in order
to make boot faster (the bootloader doesn’t have to know how to parse a filesystem). They
have usually a size of 16 MiB.

ROOT-A (/dev/sda3) and ROOT-B (/dev/sda5) are rootfs partitions. Each block is signed and
the signatures are verified on the fly when blocks are read from the SSD (with a tool
called dm-verity). They have usually a size of 2 GiB.

The kernel and the rootfs are duplicated in order to make autoupdates easy. Let’s suppose
you are running on KERN-A with the ROOT-A rootfs. If there is an autoupdate available,
Chrome OS will download the update directly into KERN-B and ROOT-B, and will change the
priorities of the partitions such that at the next reboot you will be running on KERN-B
and ROOT-B.

Kernel partitions have three flags: the successful flag (0-1), the tries flag (0-15) and
the priority flag (0-15). The BIOS will try to boot the partition with the highest
priority first. The others flags are used to handle a bad update: if a kernel doesn’t
boot, the tries flag is decremented, and when tries = successful = 0, this kernel is not
even tried. If the kernel boots correctly, successful is set to 1 so this kernel will not
be skipped.

Finally the stateful partition (STATE /dev/sda1) is the partition containing everything
that has to be read-write and conserved between autoupdates. This includes the /home
directory, the /var directory and the /usr/local directory (this will be very useful for
us, because scripts in /usr/local/bin will not be removed when there is an autoupdate).
This partition takes all the remaining space (usually about 10.7 GiB).

The other partitions are currently not used.

You can see the currently running rootfs partition with the command "rootdev -s", or just
"rootdev" if rootfs verification is disabled.

Developer mode, part 2
----------------------

We will now replace the BIOS by the dev BIOS which will accept a self-signed kernel and
then remove rootfs verification by the kernel.

To install the dev BIOS, run the command "chromeos-firmwareupdate --mode=todev" (as root)
and reboot (the warning screen will not be the same). Now that you are using the dev
firmware you can remove rootfs verification with the command
"/usr/share/vboot/bin/make_dev_ssd.sh --remove_rootfs_verification" (the command will
actually not work but will tell you the right "--partitions n" argument to add).

If you reboot now, you should have rootfs verification disabled and mounted read-write
(you can test it with "touch /a" as chronos, if you have a permission error it is mounted
read-write, if you have a "read-only filesystem" error if it is not).

Preparing the partitions
------------------------

In order to install Arch on ROOT-C, we need to grow the partition to an acceptable size.
We will steal some space to the stateful partition because 10.7 GiB is way too much for
it. I do not really know how much space you should leave for the stateful partition, this
will likely depend on how much offline webapps you will be using. I’m not using many
webapps and my stateful partition is currently using about 400 MiB (according to df) or
280 MiB (according to du), so I will give 10 GiB to Arch and the rest (806 MiB) to the
stateful partition, I hope this will be enough (autoupdates do not need space, they are
directly applied to the new partition).

The commands are

   cgpt add -i 1  -b 266240  -s 1650688  -l STATE /dev/sda
   cgpt add -i 13 -b 1916928 -s 20971520 -l ARCH  /dev/sda

Note that this only modifies the size of the partitions, not the content. In particular,
the headers of the stateful partition will now report the wrong size, Chrome OS will
notice it and reformat the stateful partition using the previous size. We can avoid that
by wiping the stateful partition, it will be automatically recreated at next boot

   dd if=/dev/zero of=/dev/sda bs=131072 seek=1040 count=6448

Note that we are erasing at a very low level a partition which is currently being used,
this is black magic, you should reboot as soon as possible after the wipe or I guess that
strange things will happen.

Installing Arch Linux
---------------------