# X1C6 S3 DSDT patch (credit to https://delta-xi.net/#056 and Ranguvar )



Update: The procedure also works for the X1 Yoga Gen3 (thanks to Ran for confirming). This patch is said to work. However, you can always go without a working patch file, see Plan B on how to proceed.

-
If you own a Lenovo X1 6th generation and want to use Linux, you're not going to be happy from the start. The entire range of hardware works splendidly and almost out of the box (let alone the fingerprint reader), but Lenovo decided to make one major change in the new UEFI firmware image. They removed traditional deep sleep (ACPI S3 sleep state) in favor of a new, Microsoft-driven sleep state called Windows Modern Standby, aka Si03. This sleep state doesn't fully turn off all components except for main memory, but puts the devices themselves into an ultra low-power state. This way, much like modern smartphones do, some devices can briefly wake up particular components of the system - most notably communication devices. The idea is, to have an always-connected feature, to e.g. download updates while sleeping or stay connected to a WiFi during sleep. This may or may not be preferable idea, but for me it is not.

On the Intel Core architecture, Linux doesn't deal with Si03 too well (as opposed to Intel Atom CPUs) and suspend energy consumption is around 4 watts on the X1, which is extremely high and won't even give you a single day of suspend time.

Luckly, together with great help from a few hackers from the Arch Linux forums (especially Ranguvar for sorting things out and fiji-flo for the working patch), we could establish a workaround. The idea is to patch the DSDT tables, a part of the ACPI firmware, written in ACPI machine language (a freakish place). Using that patch, sleep energy consumption drops from 4 to 0.15 watts on my machine.

If you follow this guide, no one is responsible for any damage to your hardware or any other kind of harming your machine. However, we didn't hear a single report of such cases as of now, and it works fine for all of us who tried it and reported to us.

The following steps should guide you to how you can do it yourself.

Reboot, enter your BIOS/UEFI. Go to Config - Thunderbolt (TM) 3 - set Thunerbolt BIOS Assist Mode to Enabled. It has also been reported that Security - Secure Boot must be disabled.

Install iasl (Intel's compiler/decompiler for ACPI machine language) and cpio from your distribution.
Get a dump of your ACPI DSDT table.
  # cat /sys/firmware/acpi/tables/DSDT > dsdt.aml
Decompile the dump, which will generate a .dsl source based on the .aml ACPI machine language dump.

  $ iasl -d dsdt.aml
Download the patch and apply it against dsdt.dsl:

  $ patch --verbose < X1C6_S3_DSDT.patch
Note: Hunk 6 may fail due to different specified memory regions. In this case, simply edit the (almost fully patched) dsdt.dsl file, search for and entirely delete the two lines reading solely the word "One". You can look at hunk 6 in the patch file to see how the lines above and below look like if you're unsure.

Plan B: If this does not work (patch is rejected): It has been the case, that certain UEFI settings may lead to different DSDT images. This means that it may be possible that the above patch doesn't work at all with your decompiled DSL. If that is the case, don't worry: Go through the .patch file in your editor, and change your dsdt.dsl by hand. This means locating the lines which are removed in the patch and removing them in your dsl. The patch contains only one section at the end which adds a few lines - these are important and make the sleep magic happen.

Make sure that the hex number at the end of the first non-commented line is incremented by one (reading DefinitionBlock, should be around line 21). E.g., if it was 0x00000000 change it to 0x00000001. Otherwise, the kernel won't inject the new DSDT table.

Recompile your patched version of the .dsl source.

  $ iasl -ve -tc dsdt.dsl
There shouldn't be any errors. When recompilation was successful, iasl will have built a new binary .aml file including the S3 patch. Now we have to create a CPIO archive with the correct structure, which GRUB can load on boot (much like initrd is loaded). We name the final image acpi_override and copy it into /boot/.

  $ mkdir -p kernel/firmware/acpi
  $ cp dsdt.aml kernel/firmware/acpi
  $ find kernel | cpio -H newc --create > acpi_override
  # cp acpi_override /boot
We yet have to tell GRUB to load our new DSDT table on boot in its configuration file, usually located in /boot/grub/grub.cfg or something similar. Look out for the GRUB menu entry you're usually booting, and simply add our new image to the initrd line. It should look somewhat like that (if your initrd line contains other elements, leave them as they are and simply add the new ACPI override):

  initrd   /acpi_override /initramfs-linux.img
Note: You will need to do this again when your distribution updates the kernel and re-writes the GRUB configuration. I'm looking for a more automated approach, but was too lazy to do it so far.

Moreover, GRUB needs to boot the kernel with a parameter setting the deep sleep state as default. The best place to do this is /etc/default/grub, since that file is not going to be overwritten when the GRUB config becomes regenerated. Simply add mem_sleep_default=deep to the GRUB_CMDLINE_LINUX_DEFAULT configuration option. It should look somewhat like that:

  GRUB_CMDLINE_LINUX_DEFAULT="quiet mem_sleep_default=deep"
Reboot.

If everything worked, you shouldn't see any boot errors and the kernel will confirm that S3 is working. The output of the following commands should look the same on your machine:

  $ dmesg | grep ACPI | grep supports"
    [    0.213668] ACPI: (supports S0 S3 S4 S5)
  $ cat /sys/power/mem_sleep
    s2idle [deep]
In most setups, simply closing the lid will probably trigger deep sleep. If you're using a systemd-based distribution (most of which are), you can also verify if it works on the command line:

  $ systemctl suspend -i
Once again, many thanks to Ranguvar for the great collaboration on the Arch forums, and to fiji-flo for managing to hack the first fully working patch. Also, to whomever wrote the article on DSDT patching in the glorious Arch Wiki. And the entire Arch community in general, you're wonderful.