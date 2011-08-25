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

If you are already a Chrome OS hacker, here is a very short resume of what will be done to
make Arch run inside Chrome OS. Don’t worry, a much much more detailed explanation will
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
* This is not completely straightforward (well, everything is relative), if you don’t know
what a shell is, it will probably be difficult to follow. But I’ll try to keep the
requirements low, in particular you don’t have to know anything about Chrome OS.

Developer mode
--------------

OK, let’s start.

When you buy a Chromebook, it is in release mode. This means that you don’t have access to
a shell, that your root filesystem (the /) is mounted read-only and cryptographically
signed by Google. So even if you find a way to have a shell and find a way to modify
something, you will not be able to sign your changes (unless you have Google’s private
key) and your computer will not boot anymore. This mode is intended for normal people
using Chrome OS normally, this make viruses very difficult to do.

As you