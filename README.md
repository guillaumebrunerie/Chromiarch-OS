Chromiarch OS
=============

     /!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\
     /!\ Alpha version, not tested as is /!\
     /!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\/!\

Description
-----------

Chromiarch OS is a way to have both Chrome OS and Arch Linux running simultaneously on a
Samsung Chromebook. Chromiarch OS consist of this documentation together with some needed
files. A video showing the final result will be available some day.

If you install Chromiarch OS following this guide, your system will look like this:
When you boot your computer, it will boot Chrome OS normally (except that you will need to
press Ctrl + D on boot because of developer mode). You will then have Chrome OS’ X server
on tty1, Chrome OS’ dev console on tty2 and an Arch Linux’ agetty on tty3. Moreover, when
you login on tty3, you can choose to start a second X server (in Arch Linux) which will
start on tty4. You can then switch between Chrome OS and Arch Linux with Ctrl + Alt + F1
and Ctrl + Alt + F4.
There are of course a lot of different setups possible, I leave it up to you to adapt the
instructions here to your needs.

License
-------
This document and the associated files are distributed under the MIT license, see the
`LICENSE` file.
Copyright © 2011, Guillaume Brunerie <guillaume.brunerie@[ens.fr|gmail.com]>

Drawbacks
---------

There are a few drawbacks about using Chromiarch OS instead of Chrome OS or Arch Linux:

General :

* Your device has to be in developer mode. This means that you will have to press Ctrl + D
at every boot and that boot time will be increased by a few seconds (don’t worry, boot
time will still be less than 15 seconds).
* Screen dimming has to be disabled, so your screen will not be powered off
anymore when you are not using your computer in Chrome OS (but if you close the lid it
will suspend as expected), and (at least with Gnome) screen dimming works correctly in the
Arch Linux side.
* I have some problems with the sound because both systems think that they are completely
managing the sound. For example sometimes the sound is not muted even if I just pressed
the “mute” button, or sound still plays in the speakers if there are headphones plugged
in.

In comparison to a true Arch Linux netbook :

* The SSD is very small. At best you can give 9 or 10 Gib to Arch.
* The keys from the upper row are the functions keys F1 to F10. This means that there is
no F11 nor F12 and that changing brightness or sound volume will not work in Arch (you can
of course remap keys as you want in Arch, but there are still not enough keys in
comparison to a regular netbook)
* Arch will run under Chrome OS’ linux kernel, so if you want a custom kernel, you will
need to recompile Chromium OS’ kernel (and this is not the easiest thing to do)
* Chrome OS autoupdates will break the Arch part of Chromiarch OS, but it is easy to
restore (and will take you one or two minutes after each autoupdate).
* If something goes wrong and your computer does not want to boot (or if you are stuck in
guest mode without having access to a shell or tty switching, as happened to me once),
it’s rather difficult to recover your system without loosing your data in Arch Linux.
You cannot boot on a live USB of a random Linux distribution because of Chrome OS’ boot
system. So you can either run recovery mode, but this will erase completely Arch and all
your data (except Chrome OS’ data which is in the cloud), or build Chromium OS to have a
bootable Chromium OS USB key and repair your computer from there (but building Chromium OS
is rather difficult). You can also try prebuilt binaries, like Hexxeh’s builds, I guess it
should work (if shell access is enabled) but I have not tested.

In comparison to a Chrome OS netbook :

* There are some problems with the touchpad (for example middle click with three-finger
click does not work in Chrome OS), but this should be fixed some day (more explanations on
this below)
* Chrome OS autoupdates will use more bandwidth, because delta updates are not available
when you have modified you file system, so it will download full updates each time.

Quick explanation
-----------------

If you are already a Chrome OS/Linux hacker, here is a very short outline of what will be
done to make Arch run inside Chrome OS. Don’t worry, a much much more detailed explanation
will follow.

You will switch your device to developer mode, create a new partition for Arch, install
Arch in a chroot (on your main computer) and copy it into the new partition. Then you will
write a new upstart job (Chrome OS uses upstart) to run a getty chrooted into Arch Linux.
Your shell will then start a second X server (in the chroot) on login.

Notes
-----

Before beginning, a few notes:

* This is legal. Unless some other companies, Google let people do whatever they want with
their computer. You will not have to do anything illegal.
* This is reversible. Whatever you do, at any time you can run recovery mode and you will
come back to factory state. This will not void your warranty at all.
* This is not straightforward (well, everything is relative), if you don’t know what a
shell is, it will probably be too difficult to follow. But I’ll try to keep the
requirements low, in particular you don’t have to know anything about Chrome OS.
* This is *not* just a proof of concept. I’m using this system everyday and find it very
convenient and useful.
* This is only the description of my system. It is not made to be configurable, it’s made
to do what I want. If you want something else, you are of course free to edit the scripts
or adapt the instructions to your needs.

In the examples,

    (co) # This is the Chrome OS shell
    (al) # This is the Arch Linux shell on your main computer
    (ca) # This is the chrooted Arch Linux shell on your Chromebook

A `#` in the prompt indicates that the command should be run as root, and a `$` indicates
that it should be run as an unprivileged user (`chronos` on Chrome OS, `username` in the
chrooted Arch Linux shell on the Chromebook).

Developer mode
--------------

OK, let’s start.

When you buy a Chromebook, it is in release mode. This means that the machine is
completely locked: you don’t have access to a shell and the root filesystem (the /) is
mounted read-only and cryptographically signed by Google. This mode is intended for normal
people using Chrome OS normally, they won’t be bothered by not having a shell, and this
makes malwares very difficult to do.

As you can guess, there is another mode (called developer mode, or dev mode) which will
allow you to disable those protections. You can switch to dev mode with the dev mode
switch present on your device (for the Samsung Chromebook, a photo is available
[here](http://www.chromium.org/chromium-os/developer-information-for-chrome-os-devices/
samsung-series-5-chromebook).

Shut down your computer, flip the dev mode switch and boot your computer. You will see a
warning screen with a sad computer asking you to press space because the verification of
Chrome OS is disabled. This is expected, and you should now *not* press Space but Ctrl + D
instead (you will need to do this every time your computer will boot, this is the main
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
earlier is that you have now access to a shell. You can access it either via the dev
console in tty2 (accessible with `Ctrl + Alt + ->`, because `-> = F2`, and you can come
back to X with `Ctrl + Alt + <-`) or via crosh: press `Ctrl + Alt + T` in Chrome and use
the "shell" command. You can either connect as root or connect as chronos and use "sudo".
But for now / is still mounted read-only and verification of cryptographic signatures is
still active.

Partitions
----------

Before going on, I should explain the (unusual) partitionning system of Chrome OS.

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

To install the dev BIOS, run the following command:

    (co) # chromeos-firmwareupdate --mode=todev

and reboot (the warning screen will not be the same). Now you should be using the dev
firmware and you can remove rootfs verification with the command

    (co) # /usr/share/vboot/bin/make_dev_ssd.sh --remove_rootfs_verification

(the command will actually not work because it is dangerous to touch the partition you are
not currently using, but it will tell you the right `--partitions n` argument to add).

If you reboot now, you should have rootfs verification disabled and mounted read-write.
You can check it with rootdev (should give you a /dev/sda*), mount, or try to create a
file /a (`touch /a` as chronos will give you a permission error if / is mounted read-write
and a "read-only filesystem" error if it is not).

Preparing the partitions
------------------------

In order to install Arch, we need to create a new partition. I will create it as
/dev/sda13 with label ARCH. We will steal some space to the stateful partition because
10.7 GiB is way too much for it. I do not really know how much space you should leave for
the stateful partition, this will likely depend on how many offline webapps you will be
using and how many files you will download. I’m not using many webapps and my stateful
partition is currently using 350 MiB (according to `df`) or 280 MiB (according to `du`),
so I will give 9 GiB to Arch and the rest (1830 MiB) to the stateful partition, I hope
this will be enough (autoupdates do not need temporary space, they are directly applied to
the new partitions).

The commands are

    (co) # cgpt add -i 1  -b 266240  -s 3747840  -l STATE        /dev/sda
    (co) # cgpt add -i 13 -b 4014080 -s 18874368 -l ARCH -t data /dev/sda

Note that this only modifies the size of the partitions, not the content. In particular,
given that the headers of the stateful partition will now report the wrong size, Chrome OS
will notice it and reformat the stateful partition using the previous size. We can avoid
that by wiping the stateful partition, it will be automatically recreated at next boot

    (co) # dd if=/dev/zero of=/dev/sda bs=131072 seek=1040 count=14640

Note that we are erasing at a very low level a partition which is currently being used,
this is black magic, you should reboot as soon as possible after the wipe or I guess that
strange things will happen.

If everything went well, during the reboot you will see the message "Autorepair in
progress", you will need to reconfigure (again) your Wi-Fi network and your Google
account, and then the partitions will look like this:
TODO

Let’s create an ext4 file system on the new partition and mount it at /tmp/ARCH:

    (co) # mkfs.ext4 /dev/sda13
    (co) # mkdir /tmp/ARCH
    (co) # mount /dev/sda13 /tmp/ARCH

Installing Arch Linux
---------------------

We are ready to install Arch Linux in our new ARCH partition. But don’t be too optimistic,
the official installer won’t work because Chrome OS’ BIOS can’t boot on a live USB (it can
only boot on a USB stick containing a Chromium OS image).

You have several options here. The first one is described [here](TODO) for Ubuntu, but
this will work similarly for Arch Linux. Basically you will install Arch Linux in
Virtualbox, extract the raw partition, transfer onto it your Chromebook, and `dd` the
partition directly in the ARCH partition. If you want to do this, note that you don’t need
a Chromium OS chroot, you can just copy the cgpt tool from your Chromebook to your main
computer.

I will not use this method here but something more lightweight: the mkarchroot tool. You
will need to have another computer running Arch Linux with the devtools package installed.
Choose where you want to create this chroot, for example `~/archroot`, and run

    (al) # mkarchroot ~/archroot --arch=i686 base

Note that the kernel of Chrome OS is 32-bits, so if your main Arch Linux computer is
64-bits you will have to build a 32-bits chroot (using the `--arch` option of pacman).

If you want to save some place, you can remove the packages from base that you don’t need,
for example `linux` (we will use Chrome OS’ kernel), utilities for fancy filesystems or
networking tools. To keep it simple, I’ll install the whole base group.

Now you can compress your chroot and put it on a USB stick:

    (al) # tar czvf /media/MyUSBstick/rootfs.tgz -C ~/archroot .

And decompress it on the ARCH partition you made earlier.

    (co) # tar xzvf /media/MyUSBstick/rootfs.tgz -C /tmp/ARCH .

Alternatively you can transfer it via SSH:

    (co) # ssh username@myarchlinuxcomputer tar cz -C ~/archroot . | tar xzv -C /tmp/ARCH .


You can now enter your new chroot and look around to see if everything looks fine:

    (co) # mount -o bind /dev /tmp/ARCH/dev   # We want the device nodes
    (co) # mount -t devpts devpts /tmp/ARCH/dev/pts  # We want pseudoterminals
    (co) # mount -t proc proc /tmp/ARCH/proc  # We want informations about the processes
    (co) # mount -t sysfs sys /tmp/ARCH/sys   # We want informations about the system
    (co) # cp /etc/resolv.conf /tmp/ARCH/etc/resolv.conf  # We want Internet
    (co) # chroot /tmp/ARCH /bin/bash

(Note: the copy of `/etc/resolv.conf` to `/tmp/ARCH/etc/resolv.conf` is temporary.
       Indeed, this file is not supposed to be static, but is updated by flimflam (the
       network manager daemon) when you connect to another network. In fact (because the
       root file system is supposed to be read only), this is a symbolic link to the real
       `/etc/resolv.conf` which is in `/var/run/flimflam/resolv.conf`. So the permanent
       solution will be to have `/etc/resolv.conf` in Arch Linux to be a symbolic link to
       `/media/chromeos/var/run/flimflam/resolv.conf`, when the root file system of Chrome
       OS will be mounted at `/media/chromeos`.)

Configuring Arch Linux
----------------------

You should be in the Arch Linux chroot on you Chromebook. Note that we bypassed the normal
installation procedure, so you should configure your system as you would do in a normal
install (cf the official Arch Linux install guide for further explanations). More
precisely, you will want to configure:

- `/etc/rc.conf` : You may want to configure everything in the LOCALIZATION section (and
  you probably want the HARDWARE_CLOCK variable set to something like "donottouch",
  because Arch should not touch the hardware clock at all), the HOSTNAME variable (but see
  below the drawbacks of this and a workaround) and the DAEMONS array. Everything else
  will not be used. Note that the keyboard keymap and fonts will also be used in the
  Chrome OS dev console on tty2.
- `/etc/hosts` : Just put here the HOSTNAME you configured in `/etc/rc.conf`.
- `/etc/locale.gen` : Uncomment the locales you need and run `locale-gen`.

You should not need to configure the other files.

Now you should set a root password:

    (ac) # passwd

And create a non-privileged user:

    (ac) # adduser username   # Choose the audio, video and wheel groups as additional
                              # groups, for example

You can install more packages:

    (ac) # pacman -S base-devel  # Everyone wants this package
    (ac) # pacman -S sudo        # This one too
    (ac) # pacman -S xorg-server xorg-xinit xorg-utils xorg-xserver-utils  # Install Xorg
    (ac) # pacman -S xf86-video-intel  # The Chromebook has an Intel graphic card
    (ac) # pacman -S xf86-input-synaptics  # There is a touchpad

In order to test the X server, we will install the following three packages used in the default
`.xinitrc`.

    (ac) # pacman -S xorg-twm xorg-xclock xterm

Now let’s test it:

    (ac) # su username
    (ac) $ startx -- :1

We have to specify a different display `:1` because otherwise it will conflict with Chrome
OS’ X server.

It should start a minimal environment with three xterm windows and one xclock. You will
notice that the touchpad does not work, but we’ll fix that later.

Now install a desktop environment / window manager:

    (ac) $ sudo pacman -S gnome gnome-extras
    (ac) $ echo "exec gnome-session" > ~/.xinitrc

If you restart X now, you will notice other things that do not work: the sound, the
webcam, the hardware accelerated graphics (direct rendering), and still the touchpad.

Sound, direct rendering and webcam
----------------------------------

This problem is kind of tricky. Playing sound requires write access to the `/dev/snd/*`
device nodes. These nodes are owned by `root:audio` with permissions `rw-rw----` and you
are in the `audio` group, so you should be able to play sound, right?

Well, no. The Linux kernel doesn’t actually know what the `audio` group is. Internally
groups are numbers and the mapping between numbers and names is in the `/etc/group` file.
And guess what? The audio group doesn’t correspond to the same number in Chrome OS and
Arch Linux, so the device nodes are owned by the `audio` group in Chrome OS but not
anymore in Arch Linux.

This is easy to fix, just edit the `/etc/group` file in Arch Linux to change the `audio`
number to 18 (there is no conflict, no group has the number 18 in Arch Linux).

There is the same problem with the `video` group (used for the webcam and the hardware
accelerated graphics), so you should also change the number corresponding to the `video`
group to 27 and it will work.

Touchpad (TODO)
--------

(NB (added later): On the dev channel, syntp does not exists anymore, it has been replaced
by another driver called cmt that should be nicer, so it is very possible that the
touchpad will work out of the box now. If you are on stable channel, you will probably
still need to do what’s described in this section until cmt becomes the default for
everyone)

Now we come to the touchpad.

For some unknown reason, Chrome OS doesn’t use the open source synaptics driver but a
closed source driver called "syntp" (which is not even available to regular Linux users).
This driver doesn’t support devices in evdev mode (ie with a device node in `/dev/input/`),
it needs raw access to the device, so there is a udev rule at
`/lib/udev/rules.d/75-synaptics.rules` putting the touchpad into serio_raw mode.  But when
the touchpad is in serio_raw mode, the synaptics driver in Arch can’t see it, so it
doesn’t work.

To solve the problem we will simply remove the udev rule. The touchpad will stay in evdev
mode and synaptics will work. Of course syntp will not work anymore, but synaptics is also
installed on Chrome OS and there is already a config file for synaptics in Chrome OS (at
`/etc/X11/xorg.conf.d/50-touchpad-synaptics.conf`) so the touchpad will also work in
Chrome OS using synaptics (this config file may be only present in the R15 version of
Chrome OS)

    (co) # mv /etc/X11/xorg.conf.d/50-touchpad-synaptics.conf{,.bak}

After rebooting, Chrome OS and Arch Linux should both have the touchpad using Synaptics
and working. If the touchpad has a strange behaviour, try to reboot one more time (I don’t
know why this is needed sometimes).

Xorg Config
--------

By default, Xorg relies on udev to determine input devices on startup
(keyboard, trackpad, etc). Unfortunately, X does not detect input devices on
startup in the chroot. (Probably because of different udev versions between
ChromeOS and Arch). I use a minimal xorg.conf file to manually specify the
keyboard and trackpad. Copy this file to /etc/X11/xorg.conf. Different
chromebooks may need different /dev/input paths for the keyboard and trackpad.
Use /var/log/Xorg.0.log from ChromeOS to determine the paths to use for your
chromebook. Hotplugging new input devices after startup does seem to work even
after manually specifying these input devices.

Hostname
--------

You may notice others problems, for example the brightness and power keys that do not work
anymore. This is due to the `powerd` daemon crashing because the hostname of the computer
changed after the boot. You should either not change the hostname in Archlinux (the
hostname is `localhost` by default and cannot be customized) or change the hostname
directly in the init scripts of Chrome OS (`/sbin/chromeos_start` and `/etc/hosts`).

We will now make the system more user-friendly by making it start everything needed
automatically at boot.

Upstart rule
------------

Chrome OS uses upstart as init system, and all the config files are in `/etc/init`. You
can find the new `chromiarchos.conf` upstart job together with this documentation. This
file defines a new upstart job starting and stopping with the `system-services` meta-job
(this means that the job will start at boot, after the normal Chrome OS startup, so that
we do not delay the login prompt of Chrome OS). The `post-start` snippet is taken from
`tty2.conf`, it disables screen blanking for the console `tty3`. Finally the rule starts
the `chromiarchos-run` script (described in the next section) that you should put in
`/usr/local/bin` such that it will stay there between auto-updates.

Entering the chroot
-------------------

Let’s look at the `chromiarchos-run` script.

I first change `stdout` and `stderr` to `/dev/tty3` and check that the chroot directory
doesn’t exists and that we are root, or bad things could happen. Then I modify the config
of the OOM killer (see next section for explanations) and I mount `/` and `/dev`,
and the root of Chrome OS at `/media/chromeos` (such that I can access Chrome OS’ system
files within Arch Linux, you should create the `/media/chromeos` directory first), and the
`/etc/resolv.conf` is linked from `/media/chromeos` (this file is needed to have Internet
in Arch). Then I enter the chroot and run the `chromiarchos-init` command, that should be
installed in the chrooted Arch Linux. We’ll see the rest later.

Re-enabling the OOM killer
--------------------------

The OOM (Out Of Memory) killer is a component of the Linux kernel that will kill some
process when the system runs out of memory. You can tweak the likeliness of some process
to be killed by changing the value of `/proc/<pid>/oom_score_adj` (this should be a value
between -1000 and 1000, the higher the value is the higher the chances are that this
process will be killed first, if the value is -1000 the process will never be killed by
the OOM killer).

Chrome OS is extensively using this feature (such that the current tab will only be killed
in last resort, for example), and by default every daemon has OOM killing disabled. This
has the consequence that every process you will run in Arch will never be choosed by the
OOM killer, which is not necessarily a good feature.

So we write 0 (the default value) into `/proc/$$/oom_score_adj` (where `$$` is the
pid of `chromiarchos-run`) so that every Arch process will inherit this value.


Starting Arch Linux
-------------------

The `chromiarchos-init` script is a replacement for the init system. It calls the scripts
`chromiarchos-sysinit` and `chromiarchos-multi` (that are just the corresponding
`/etc/rc.xxx` files where I removed what is not needed), then it starts `agetty`, and
finally call the `chromiarchos-shutdown` script when `agetty` stops.

