Chromiarch OS
=============

Description
-----------

Chromiarch OS is a way to have both Chrome OS and Arch Linux running simultaneously on
a Samsung Chromebook. Chromiarch OS consists mainly of this documentation together with
some useful scripts. A video showing the final result will be uploaded soon.

If you install Chromiarch OS following this guide, your system will look like this:
When you boot your computer, it will boot Chrome OS normally (except that you will need to
press Ctrl + D on boot because of developer mode). You will then have Chrome OS’ X server
on tty1, Chrome OS’ dev console on tty2 and an Arch Linux’ agetty on tty3. Moreover, when
you login on tty3, you can choose to start a second X server (in Arch Linux) which will
start on tty4. You can then switch between Chrome OS and Arch Linux with Ctrl + Alt + F1
and Ctrl + Alt + F4.

Drawbacks
---------

Here are a few drawbacks about using Chromiarch OS instead of Chrome OS or Arch Linux:

* Your device has to be in developer mode. This means that you will have to press Ctrl + D
at every boot and that boot time will be increased by a few seconds (don’t worry, boot
time will still be less than 15 seconds).
* The synaptics driver is used for the touchpad instead of the syntp driver. In
particular, two-finger drag&drop does not work anymore (but two finger scrolling is much
easier with synaptics).
* Screen dimming is disabled, so your screen will not be powered off anymore when you are
not using your computer.
* Playing sound from Chrome OS and Arch simultaneously does not work, I haven’t figured
out how to fix this.
* Chrome OS autoupdates will break Chromiarch OS, but it will take you less than one
minute to restore it.
* The SSD is small. At best you can give 9 or 10 Gib to Arch.
* The keys from the upper row are the functions keys F1 to F10. This means that you will
not have F11 nor F12 and that changing brightness or sound volume will not work in Arch
(unless you remap them in your window manager, but you will loose functions keys this way)
* Arch will run under Chrome OS’ linux kernel, so if you want a custom kernel, you will
need to recompile Chromium OS’ kernel.
* If something goes wrong and your computer does not want to boot, you’re almost screwed.
You can either run recovery mode, but this will erase completely Arch and all your data
(except Chrome OS’ data which is in the cloud), or build Chromium OS to have a bootable
USB key and repair your computer from here (but building Chromium OS is difficult). You
can also try prebuilt binaries, like Hexxeh’s builds, I guess it should work.

Quick explanation
-----------------

If you are already a Chrome OS hacker, here is a very short outline of what will be done
to make Arch run inside Chrome OS. Don’t worry, a much much more detailed explanation will
follow.

You will switch your device to developer mode, create a new partition for Arch, install
Arch in a chroot (on your main computer) and copy it into the new partition. Then you will
write a new upstart job (Chrome OS uses upstart) to run a getty chrooted into Arch Linux.
This getty will then start a second X server (in the chroot) on login.

Notes
-----

Before beginning, a few notes:

* This is legal. Unless some other companies, Google let people do whatever they want with
their computer. You will not have to do anything illegal.
* This is reversible. Whatever you do, at any time you can run recovery mode and you will
come back to factory state. This will not void your warranty at all.
* This is not straightforward (well, everything is relative), if you don’t know what a
shell is, it will probably be difficult to follow. But I’ll try to keep the requirements
low, in particular you don’t have to know anything about Chrome OS.
* This is not just a proof of concept. I’m using this system everyday and find it very
convenient and useful.
* This is only the description of my setup. It is not made to be configurable, it’s made
to do what I want personally. If you want something else, you are of course free to edit
the scripts or adapt the instructions to your needs.

Developer mode
--------------

OK, let’s start.

When you buy a Chromebook, it is in release mode. This means that the machine is
completely locked: you don’t have access to a shell and the root filesystem (the /) is
mounted read-only and cryptographically signed by Google. This mode is intended for normal
people using Chrome OS normally, they won’t be bothered by not having a shell, and this
makes malwares very difficult to do.

As you can guess, there is another mode (called developer mode) which will allow you to
disable those protections. You can switch to dev mode with the dev mode switch present on
your device (for the Samsung Chromebook, a photo is available [here](TODO)).

Shut down your computer, flip the dev mode switch and boot your computer. You will see a
warning screen with a sad computer asking you to press space because Chrome OS is
broken. This is a lie! Chrome OS is not broken, you just have to press Ctrl + D to boot
Chrome OS (you will need to do this every time your computer will boot, this is the
drawback of dev mode). You can also wait 30 seconds, it will boot automatically (and your
computer will beep after 20 seconds, this is normal). This screen is intended to normal
users having accidentally flipped the dev mode switch. Such a user will most likely either
switch the dev mode switch off immediately, run recovery mode or call technical support,
so will not run a potentially insecure operating system.

During the first boot in dev mode, Chrome OS will slowly erase the stateful partition
(more on partitions in the next section) so you will have to wait 5 / 10 minutes. You will
lose everything that was there: list of last open tabs, downloaded files, offlineness of
offline-capable apps, cookies, remembered users and Wi-Fi networks, etc. You will not lose
any data which is stored in the cloud of course, that’s the whole point of Chrome OS.

You should now be in Chrome OS with the dev mode switch on. The only difference with
earlier is that you have now access to a (root) shell. You can access it either via the
dev console in tty2 (accessible with Ctrl + Alt + ->, because -> = F2, and you can come
back to X with Ctrl + Alt + <-) or via crosh: press Ctrl + Alt + T in Chrome and use the
"shell" command. You can either connect as root or connect as chronos and use "sudo". But
for now / is still mounted read-only and verification of cryptographic signatures is still
active.

Partitions
----------

Before going on, I should explain the (non standard) partitionning system of Chrome OS.

Unlike most computers, Chrome OS uses GPT partition tables. This is a "new" partitionning
system replacing the old MBR, without restrictions on the number or the size of partitions
(with the MBR you can only have 4 primary partitions, if you want more you need to use
extended and logical partitions). We will use the cgpt tool (present in Chrome OS) to
look at and modify partitions.

If you run "sudo cgpt show /dev/sda" you will see the partitions present on your drive.
The output should be something like this (size and positions are expressed in 512 bytes
blocks):
TODO

There is a lot of partitions here. Let’s take a look at the more important partitions.

KERN-A (/dev/sda2) and KERN-B (/dev/sda4) are the kernel partitions. They contain the
Linux kernel and its signature, not formatted with any filesystem in order to make boot
faster (the bootloader doesn’t have to parse a filesystem). They have usually a size of 16
MiB.

ROOT-A (/dev/sda3) and ROOT-B (/dev/sda5) are rootfs partitions containing most of the
Chrome OS operating system (except the kernel and the modifiables parts). These partitions
are not signed as a whole, but each block is signed and the signatures are verified on the
fly when blocks are read from the SSD (with a tool called dm-verity). Checking the
signature of a 2 GiB filesystem at boot would take too much time. They have usually a size
of 2 GiB.

The kernel and the rootfs are duplicated in order to make autoupdates easy. Let’s suppose
you are running on KERN-A with the ROOT-A rootfs. If there is an autoupdate available,
Chrome OS will download the update into the KERN-B and ROOT-B partitions, and when the
update is over it will change the priorities of the partitions such that at the next boot
you will be running on KERN-B and ROOT-B.

Kernel partitions have three flags: the successful flag (0-1), the tries flag (0-15) and
the priority flag (0-15). The BIOS will try to boot the partition with the highest
priority first. The others flags are used to handle a bad update: if a kernel doesn’t
boot, the tries flag is decremented, and when tries = successful = 0, this kernel is not
even tried. If the kernel boots correctly, successful is set to 1 so this kernel will not
be skipped.

Finally the stateful partition (STATE, /dev/sda1) is the partition containing everything
that has to be read-write and conserved between autoupdates. This includes the /home
directory, the /var directory and the /usr/local directory (this will be very useful for
us, because scripts in /usr/local/bin will not be removed after an autoupdate).  This
partition takes all the remaining space (usually about 10.7 GiB).

The other partitions are currently not used by Chrome OS.

You can see the current rootfs partition with the command "rootdev -s", or just "rootdev"
if rootfs verification is disabled. For example if the output of "rootdev -s" is
"/dev/sda5", you are running with the ROOT-B rootfs and (necessarily) the KERN-B kernel.

Finally the BIOS is also non standard. There is first a read-only BIOS, containing
Google’s public key and code for the recovery mode. The only way to modify the read-only
BIOS is to physically open your computer. This will disable the write protection and void
your warranty. After the read-only BIOS there are two (also for the sake of autoupdates)
read-write BIOSes whose job is to choose the right kernel, check that the signature of the
kernel is correct and boot the kernel.

Developer mode, part 2
----------------------

We will now replace the (read-write) BIOS by the dev BIOS which will accept a self-signed
kernel and then remove rootfs verification by the kernel.

To install the dev BIOS, run the command "chromeos-firmwareupdate --mode=todev" (as root)
and reboot (the warning screen will not be the same). Now you should be using the dev
firmware and you can remove rootfs verification with the command
"/usr/share/vboot/bin/make_dev_ssd.sh --remove_rootfs_verification" (the command will
actually not work because it is dangerous to touch the partition you are not currently
using, but it will tell you the right "--partitions n" argument to add).

If you reboot now, you should have rootfs verification disabled and mounted read-write.
You can check it with rootdev (should give you a /dev/sda*), mount, or try to create a
file /a ("touch /a" as chronos will give you a permission error if / is mounted read-write
and a "read-only filesystem" error if it is not).

Preparing the partitions
------------------------

In order to install Arch, we need to create a new partition. I will create it as
/dev/sda13 with label ARCH. We will steal some space to the stateful partition because
10.7 GiB is way too much for it. I do not really know how much space you should leave for
the stateful partition, this will likely depend on how many offline webapps you will be
using and how many files you will download. I’m not using many webapps and my stateful
partition is currently using 350 MiB (according to df) or 280 MiB (according to du), so I
will give 9 GiB to Arch and the rest (1830 MiB) to the stateful partition, I hope this
will be enough (autoupdates do not need temporary space, they are directly applied to the
new partitions).

The commands are

   cgpt add -i 1  -b 266240  -s 3747840  -l STATE /dev/sda
   cgpt add -i 13 -b 4014080 -s 18874368 -l ARCH  /dev/sda

Note that this only modifies the size of the partitions, not the content. In particular,
given that the headers of the stateful partition will now report the wrong size, Chrome OS
will notice it and reformat the stateful partition using the previous size. We can avoid
that by wiping the stateful partition, it will be automatically recreated at next boot

   dd if=/dev/zero of=/dev/sda bs=131072 seek=1040 count=14640

Note that we are erasing at a very low level a partition which is currently being used,
this is black magic, you should reboot as soon as possible after the wipe or I guess that
strange things will happen.

If everything went well, during the reboot you will see the message "Autorepair in
progress", you will need to reconfigure (again) your Wi-Fi network and your Google
account, and then the partitions will look like this:
TODO

Installing Arch Linux
---------------------

We are ready to install Arch Linux in our new ARCH partition. But don’t be too optimistic,
the official installer won’t work because Chrome OS’ BIOS can’t boot on a live USB (it can
only boot on a USB stick containing a Chromium OS image).

You have several options here. The first one is described [here](TODO) for Ubuntu, but
this will work similarly for Arch Linux. Basically you will install Arch Linux in
Virtualbox, extract the raw partition, transfer onto it your Chromebook, and dd the
partition directly in the ARCH partition. If you want to do this, note that you don’t need
a Chromium OS chroot, you can just copy the cgpt tool from your Chromebook to your main
computer.

I will not use this method here but something much lighter: the mkarchroot tool. You will
need to have another computer running Arch Linux with the devtools package installed.
If your main computer is running Arch Linux, an easier solution is to use the mkarchroot
tool (in the devtools package). On your main Arch Linux computer, create a minimal Arch
Linux chroot with the following command: `mkarchroot /path/to/somewhere/archroot base`
(note that Chrome OS’ kernel is 32 bits, so if you are on Arch64 you will need a custom
`pacman.conf` including 32 bits repositories.

Once you have a working Arch chroot, you can compress it:
`tar czvf rootfs.tgz -C /path/to/somewhere/archroot .`, transfer it onto your Chromebook
(through the network or with an USB stick), make a new ext4 file system on the ARCH
partition (or whatever filesystem you want): `mkfs.ext4 /dev/sda13` and untar the Arch
chroot in the new partition:
`mkdir /tmp/ARCH; mount /dev/sda13 /tmp/ARCH; tar xzvf rootfs.tgz -C /tmp/ARCH .`

Entering the chroot
-------------------

To enter your new chroot (in your Chromebook) issue the following commands:

   mkdir /tmp/chroot
   mount /dev/sda13 /tmp/chroot
   mount -o bind   /dev   /tmp/chroot/dev
   mount -t devpts devpts /tmp/chroot/dev/pts
   mount -t tmpfs  devshm /tmp/chroot/dev/shm
   mount -t proc   proc   /tmp/chroot/proc
   mount -t sysfs  sys    /tmp/chroot/sys
   chroot /tmp/chroot /bin/bash



X server
--------

We have a working Arch chroot in the Chromebook, we can now install the X server inside
this chroot. Don’t forget to install the package xf86-video-intel for the graphic card.
You can install the window manager / desktop environment you want and run the X server
with `startx`.

It should work, except for a few things: the touchpad, the sound, direct rendering and
the webcam.

Sound, direct rendering and the webcam
--------------------------------------

The problem is actually very tricky. Playing sound requires write access to the
`/dev/snd/*` device nodes. The nodes are owned by `root:audio` with permissions
`rw-rw----` and you are in the `audio` group, so you should be able to play sound, right?

Well, no. The Linux kernel doesn’t actually knows what the `audio` group is, internally
groups are numbers and the mapping between numbers and names is in the `/etc/group` file.
And guess what? The audio group doesn’t correspond to the same number in Chrome OS and
Arch Linux, so the device nodes are not seen owned by the `audio` group in Arch Linux.

This is easy to fix, just edit the `/etc/group` file in Arch Linux to change the `audio`
number to 18.

The webcam and direct rendering suffer from the same problem with the `video` group. Just
change the number corresponding to the `video` group to 27 and it will work.

Touchpad
--------

Now we come to the touchpad.

Here is a little explanation of the problem: for some unknown reason, Chrome OS doesn’t
use the open source synaptics driver but the closed source syntp driver (which is not even
available to regular Linux users). This driver doesn’t support `/dev/input/*` devices, it
needs a raw access to the device, so there is a udev rule at
`/lib/udev/rules.d/75-synaptics.rules` which change the touchpad mode to serio_raw mode.
But when the touchpad is in serio_raw mode, the synaptics driver in Arch can’t see it, so
it doesn’t work.

The solution will simply be to remove the offending udev rule. The touchpad will stay in
evdev mode and synaptics will work. Fortunately, there is already a config file for
synaptics in Chrome OS (at `/etc/X11/xorg.conf.d/50-touchpad-synaptics.conf`) so the
touchpad will also use synaptics in Chrome OS.

If after removing the udev rule and rebooting, the touchpad is behaving strangely, you may
need to reboot once again. It seems to be a bug in the touchpad firmware but I’m not sure.

Internet
--------

In order to have Internet in the chroot, you have to copy the /etc/resolv.conf file.

Re-enabling the OOM killer
--------------------------

The OOM (Out of memory) killer is a component of the Linux kernel used to kill some
process when the system runs out of memory. You can tweak the likeliness of some process
to be killed by changing the value of `/proc/<pid>/oom_score_adj` (this should be a value
between -1000 and 1000, the higher the value is the higher the chances are that this
process will be killed first, if the value is -1000 the process will never be OOM killed).

Chrome OS is extensively using this feature (for example the foreground tab will have a
higher oom_score_adj), and by default every daemon has OOM killing disabled. This means
that every process you will run in Arch will never be choosed by the OOM killer.

So we write 0 (the usual default value) into `/proc/$$/oom_score_adj` and every Arch
process will inherit this value.

Putting all this together
-------------------------

So far so good, we can now write a script which will automatically mount the ARCH
partition. This script is available as `chromiarchos_run` and should be put in the
`/usr/local/bin` directory in Chrome OS.

I check that the chroot directory doesn’t exist and that you are root. Then I create the
directory, mount the root partition, the /dev, the /etc/resolv.conf and the root
filesystem of Chrome OS (in order to be able to access it from Arch, this is completely
optional).

Then I enter the chroot (with something called `jchroot`, see later) and run the
`/usr/local/bin/chromiarchos_sysinit` script. This script will do some initialisation and
then start agetty. When agetty stops, the scripts returns and I can kill every program
still running in Arch, unmount the partitions and remove the directory (note that `/tmp`
in Chrome OS is mounted as tmpfs, so it is automatically cleared anyway after a reboot).

