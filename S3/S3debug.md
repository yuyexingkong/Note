# BEST PRACTICE TO DEBUG LINUX* SUSPEND/HIBERNATE ISSUES

BY [ZHANG RUI](https://01.org/users/rzhang) ON SEP 25, 2015

and [CHEN, YU](https://01.org/user/27924)

## 1 OVERVIEW

When talking about Linux* suspend/hibernate, we are referring to the following three supported Linux system sleep states:

- STI (suspend-to-idle) is a generic, pure software, lightweight system sleep state. Together with platform-specific driver enhancements, this state can be used to reach the S0i3 state on Intel® platforms. To enter this state, run the shell command: `echo freeze > /sys/power/state`
- STR (suspend-to-RAM) offers significant power savings as everything in the system except memory is put into a low-power state. Memory should be placed into the self-refresh mode to retain its contents. ACPI platforms are in S3 after suspending to RAM. To enter this state, run the shell command: `echo mem> /sys/power/state`
- Hibernation (hibernate-to-disk) operates similarly to Suspend-to-RAM, but includes a final step that writes memory contents to disk. On resume, this is read and memory is restored to its presuspend state. Usually, the ACPI platform is in S4 state after suspending to disk. To enter this state, run the shell command: `echo disk > /sys/power/state`

For more details about each system low power state, refer to https://www.kernel.org/doc/Documentation/power/states.txt.

For more details about sysfs interfaces for Linux suspend/hibernate, like `/sys/power/state` mentioned above, refer to https://www.kernel.org/doc/Documentation/power/interface.txt.

The Intel Open Source Technology Center (OTC ) PRC Kernel Power team has been working on Linux Power Management subsystem quality improvement for years. Our experience of working on issues raised either via emails in Linux mailing list or via https://bugzilla.kernel.org/ shows the following:

- Low efficiency. This is because problems cannot be locally reproduced for the developer in most cases. Thus we rely on users to provide detailed descriptions of the problem symptoms, the necessary debug information, and some test results if needed. Getting things ready for further debugging may take days or even weeks because of different time zones.
- Extra work. This is because Linux `STI/STR/HTD` are system-level low power states that need hardware, firmware, Linux PM core, and many other kernel components, especially device drivers, to co-work well. This means that a single error in any single component might break system suspend/hibernate. For the problems that are caused by other components, even if we’ve found the root cause, we still need to rely on the component owner to provide a proper fix in most cases.

Thus, this documentation is written to help users do some debug routines so that they can do the following:

- Find the root cause or narrow down the problem if possible.
- Raise the issue properly by providing:
  - A precise description of the issue.
  - Test results of necessary debug routines.
  - Precise components and owners, for example: Linux PM core, drivers, BIOS, etc.

In this document

Chapter 2 introduces some common debug methods that can be used in Chapter 4.

Chapter 3 introduces some typical issues that can break Linux suspend/hibernate, based on our experience of Linux PM Maintenance.

Chapter 4 gives step-by-step instructions based on different symptoms of the problem.

Thus, when users encounter a suspend/hibernate related issue, we recommend that users find the proper section in Chapter 4 that describes the problem encountered and follow the step-by-step instructions of that section directly.

## 2 COMMON SUSPEND/HIBERNATION DEBUGGING METHODS

This lists some common debugging methods that are useful for people who encounter a suspend/hibernate related issue.

2.1 initcall_debug

Adding the `initcall_debug` boot option to the kernel cmdline will trace initcalls and the driver pm callbacks during boot, suspend, and resume. It is useful to check if any specified driver/component fails. Make sure to always enable it when debugging STI/STR/HTD related issues. In general, positive results mean the callback has no error/warning and returns 0 in a reasonable time. This is demonstrated in the following:

```
[ 76.201970] calling 0000:00:02.0+ @ 2298, parent: pci0000:00
[ 76.217006] call 0000:00:02.0+ returned 0 after 14677 usec
```

Negative results means the callback either contains some error/warning, or it takes an unreasonable long time to complete.

2.2 no_console_suspend

Adding the `no_console_suspend` boot option to the kernel cmdline disables suspending of consoles during suspend/hibernate. Once this option is added, debugging messages can reach various consoles while the rest of the system is being put to sleep. This may not work reliably with all consoles, but is known to work with serial and VGA consoles.

2.3 ignore_loglevel

Adding the `ignore_loglevel` boot option to the kernel cmdline prints all kernel messages to the console no matter what the current loglevel is, which is useful for debugging.

2.4 dynamic debug

For STI/STR, adding `dyndbg="file suspend.c +p`" boot option, for HTD, adding the `file swap.c +p;file snapshot.c +p;file hibernate.c +p` boot option enables dynamic debugging. Thus we can gather more information during hibernate.

2.5 serial console

Serial console output can be very useful to track some system hang issues, especially when the VGA console is down or is not able to record the full crash logs. To enable serial console, add `console=ttyS0,115200 and no_console_suspend` to the kernel cmdline, and then use a program like minicom or GNU screen on another machine to set up the serial port also to `115200 8-N-1` to capture the log. For more details, refer to https://www.kernel.org/doc/Documentation/serial-console.txt.

2.6 rtc trace

rtc trace is a feature used to capture the broken driver in the form of file+line during suspend/resume, save it to CMOS SRAM, and restore it during the next boot. To enable this, make sure your kernel is built with `CONFIG_PM_TRACE_RTC=y`, and pm async is disabled by `echo 0 > /sys/power/pm_async`. Refer to https://www.kernel.org/doc/Documentation/power/s2ram.txt for more details.

Test result: The dmesg output after rebooting from the failure `STI/STR/HTD`. Note that the reboot must happen in three minutes, or else the data saved in SRAM will be ruined.

2.7 pm_async

Currently, Linux suspends and resumes all the devices in parallel to save time. Disabling async suspend/resume by `echo 0 > /sys/power/pm_async` can be used to check if the suspend/hibernate failure is caused by some unknown device dependency issue. This can also be used to tune driver suspend/resume latency.

Test result: Check if the problem still exists after disabling async suspend/resume.

2.8 pm_test

`pm_test` is a debugging method that the system can do partially STR/HTD, and it can be used to narrow down which part of code that creates the STI/STR/HTD failure. Refer to [https://www.kernel.org/doc/Documentation/power/basic-pm-debugging.txt](https://www.kernel.org/doc/Documentation/power/basic-pm-debugging.txt to get more details) to get more details.

Test result: Check which test modes have the problem and which test modes don't have the problem. Dmesg output of the failure modes, if possible.

2.9 ACPI wakeup

`/proc/acpi/wakeup` lists the wake up capability of all ACPI devices and references to physical devices if they are available. Poking this file as shown in the following examples can enable/disable a device' wakeup capability. For example, you can disable/enable the "enabled"/"disabled" devices by doing the following:

```
root@linux-machine# cat /proc/acpi/wakeup
Device  S-state   Status   Sysfs node
...
EHC1      S4    *enabled   pci:0000:00:1d.0
...
root@linux-machine# echo EHC1 > /proc/acpi/wakeup
root@linux-machine# cat /proc/acpi/wakeup
Device  S-state   Status   Sysfs node
...
EHC1      S4    *disabled  pci:0000:00:1d.0
...
```

Test result: Disabling and enabling each device's wakeup capability solves the problem.

2.10 acpidump

`acpidump` is a tool that can be used to dump the ACPI tables provided by the BIOS. This is helpful when debugging suspend/hibernate problems that are probably caused by ACPI. To get the tool, just go to the tools/power/acpi/ directory of your kernel source code and run “make”.

2.11 rtcwake

`rtcwake` is a tool that can be used to enter a system sleep state (suspend/hibernate) until a specified wakeup time. And you can easily use it to perform suspend/hibernate stress tests. For example, you can use this simple script to run 1000 STR cycles easily:

```
for i in $(seq 1000); do
    rtcwake –m mem –s 30
done
```

2.12 analyze_suspend

The `analyze_syspend` tool provides the capability for system developers to visualize the activity between suspend and resume, allowing them to identify inefficiencies and bottlenecks. For example, you can use following command to start:

```
./analyze_suspend.py -rtcwake 30 -f -m mem
```

And 30 seconds later the system resumes automatically and generates 3 files in the `./suspend-yymmddyy-hhmmss` directory:

```
mem_dmesg.txt  mem_ftrace.txt  mem.html
```

You can first open the mem.html file with a browser, and then dig into `mem_ftrace.txt` for data details. You can get the analyze_suspend tool via git:

```
git clone https://github.com/01org/suspendresume.git
```

For more details, go to the homepage: https://01.org/suspendresume.

```
Test result: mem_dmesg.txt mem_ftrace.txt mem.html
```

2.13 disk mode

This is used for HTD only. Try `echo {shutdown/reboot} > /sys/power/disk` to skip some platform specific code that may be buggy before running `echo disk > /sys/power/state`. On ACPI platforms, the system actually enters S5 instead of S4 when this is used.

Test result: if the problem still exists when `/sys/power/disk` equals shutdown or reboot.

## 3 TYPICAL SUSPEND/HIBERNATE ISSUES

3.1 Regression

Regressions are when suspend/hibernate used to work well, but fails after upgrading the kernel. In this case, the quickest and most efficient way to find the root cause and solve the problem is to use git bisect to find out which commit introduced the problem and then report the problem to the commit author/component owner.

3.2 Driver specific issues

A broken driver can cause:

- The system to fail to enter suspend (suspend command returns with an error code).
- The system to hang during suspend/resume.
- The system to take an unreasonably long time to suspend/resume.
- The debug trace/error log to generate in dmesg after suspend/resume.
- The driver itself to fail to work.

So when you encounter suspend/hibernate related problems, usually the first thing is to verify if the problem is a driver specific issue. If the system is responsive after resume/suspend fails ( you can reach the system via VGA/serial console, ssh, etc), try to reproduce the problem with the boot options `initcall_debug` and `no_console_suspend`. If there are, do the following:

1. Build the drivers as modules
2. Unload the drivers
3. Suspend and resume the machine
4. Reload the drivers using the following commands: `modprobe -r foo && echo mem > /sys/power/state & modprobe foo`
5. If the drivers are unloadable, then do not build the driver at all via the Kconfig setting.

If the problem cannot be reproduced and the errors/warnings do not reappear after this then this is a driver specific issue.

If the system is not responsive, do the following:

1. Enable serial console and boot with `initcall_debug` and `no_console_suspend`.
2. Check the serial console output for errors/warnings caused by drivers.
3. If you find any errors/warnings, build the drivers as modules.
4. Unload the drivers.
5. If the drivers can't be unloaded at runtime, don't build the drivers at all via the Kconfig setting.
6. Suspend and resume the machine.
7. Reload the drivers: `modprobe -r foo && echo mem > /sys/power/state & modprobe foo`

If the problem can't be reproduced and the errors/warnings don't reappear after these steps, then the drivers are causing the issue.

If the system works well after resuming, but some drivers do not work anymore, do the following:

1. Build the drivers as modules.
2. Unload the drivers
3. Suspend and resume the machine
4. Reload the drivers: `modprobe -r foo && echo mem > /sys/power/state & modprobe foo`

If the problem is not reproducible, this is a driver-specific issue.

For these issues, file a [bug report](https://docs.google.com/document/d/1Fk5KeTcEbWkXUu8uv39wR8B_PDMn5sqJSsxxIRNtrUA/edit#heading=h.111kx3o) to the driver/component owner directly, because these problems show that the buggy drivers/components break Linux suspend/hibernate.

The following sections talks about some typical driver-specific issues that might break suspend/hibernate.

3.2.1 Graphics issue

Whenever the system hangs during suspend/hibernate or the monitor does not display anymore after resuming, try the following steps:

1. Disable the kernel graphics driver you're using. On Intel platforms, set `CONFIG_DRM_I915=n`.
2. Note: Do not use `nomodeset modprobe.blacklist=i915` in the command line, because sometimes the i915 driver might get probed by other components.
3. Try suspending/resuming again.
4. If you see a black screen without graphics after resuming, try to reach the system via serial console or SSH. If the system is still working, this is a graphics issue.
5. If neither serial console nor SSH is available, press Num Lock on your PS2 keyboard. If the Num Lock LED lights up when you press the key, then use the keyboard to type reboot. Although the monitor has no display, if the system reboots successfully, then it is likely that the system is working correctly, and this is a graphics issue.

For graphics issues, file a bug at https://bugs.freedesktop.org/. For example, we isolated the root cause of a suspend/hibernate problem to the graphics driver by the hint from the following lines in dmesg output:

 

```
[  218.176542] ------------[ cut here ]------------
[  218.176550] WARNING: CPU: 1 PID: 221 at drivers/gpu/drm/i915/intel_lrc.c:1100 gen8_init_rcs_context+0x15d/0x160()
[  218.176552] WARN_ON(w->count == 0)
[  218.176553] Modules linked in:
[  218.176556] CPU: 1 PID: 221 Comm: kworker/u16:3 Tainted: G        W      3.18.0-eywa-dirty #30
[  218.176558] Hardware name: Intel Corporation Skylake Client platform/Skylake Y LPDDR3 RVP3, BIOS SKLSE2P1.86C.B060.R01.1411140050 11/14/2014
[  218.176564] Workqueue: events_unbound async_run_entry_fn
[  218.176567]  0000000000000009 ffff88009a96fad8 ffffffff82431683 0000000000000001
[  218.176570]  ffff88009a96fb28 ffff88009a96fb18 ffffffff8111e001 00000000fffe6b2d
[  218.176573]  ffff88009b42f300 0000000000000005 ffff88009b670000 ffff88009b671ce8
[  218.176574] Call Trace:
...
[  218.176657] ---[ end trace f587fddb962240b1 ]---
```

3.2.2 Audio issue

We have not seen audio drivers break suspend/hibernate, but the audio drivers themselves may take a long time to resume, as seen in the following example:

```
[   61.273112] calling  hdaudioC0D2+ @ 1236, parent: 0000:00:1f.3
[   64.295757] snd_hda_intel 0000:00:1f.3: azx_get_response timeout, switching to polling mode: last cmd=0x208f8100
[   65.303916] snd_hda_intel 0000:00:1f.3: No response from codec, disabling MSI: last cmd=0x208f8100
[   66.312147] snd_hda_intel 0000:00:1f.3: azx_get_response timeout, switching to single_cmd mode: last cmd=0x208f8100
[   66.312402] azx_single_wait_for_response: 153 callbacks suppressed
[   66.323406] snd_hda_codec_hdmi hdaudioC0D2: Unable to sync register 0x2f0d00. -5
[   66.323791] snd_hda_codec_hdmi hdaudioC0D2: HDMI: invalid ELD buf size -1
[   66.324179] snd_hda_codec_hdmi hdaudioC0D2: HDMI: invalid ELD buf size -1
[   66.324565] snd_hda_codec_hdmi hdaudioC0D2: HDMI: invalid ELD buf size -1
[   66.324571] call hdaudioC0D2+ returned 0 after 4932188 usecs
```

In this case, raise an [audio bug](https://docs.google.com/document/d/1Fk5KeTcEbWkXUu8uv39wR8B_PDMn5sqJSsxxIRNtrUA/edit#heading=h.111kx3o).

3.2.3 USB issue

Usually, USB issues will not hang the system, but some USB devices may stop working after system resume. If some USB devices (like USB mouse/keyboard/network) stop working after resuming the system while other devices (like a PS/2 keyboard) seem to work fine, check the dmesg/serial log for any USB warnings. If you find a warning, this is probably a USB issue. For example:

```
[ 21.203077] xhci-hcd: probe of xhci-hcd.1.auto failed with error -110
```

In this case, raise a [USB bug](https://docs.google.com/document/d/1Fk5KeTcEbWkXUu8uv39wR8B_PDMn5sqJSsxxIRNtrUA/edit#heading=h.111kx3o).

3.2.4 mmc issue

MMC failures can cause suspending operations, especially suspend-to-disk, to fail. Check for MMC warnings in the dmesg and serial console output during suspending. For example:

```
[90.373541] mmc1: Timeout waiting for hardware interrupt.
[160.493633] mmc1: error -110 during resume (card was removed?)
```

If rootfs is not on MMC, try again with the "modprobe.blacklist=mmc_block" boot option. If rootfs is on MMC, use something like a USB drive with the same kernel/distro and try again with modprobe.blacklist=mmc_block in the command line. If the problem is gone after blacklisting the MMC driver, file a [MMC bug](https://docs.google.com/document/d/1Fk5KeTcEbWkXUu8uv39wR8B_PDMn5sqJSsxxIRNtrUA/edit#heading=h.111kx3o).

3.2.5 i2c issue

We have also seen several errors caused by I2C. For example:

```
calling i2c_designware.0+ @ 2867, parent: 0000:00:15.0
------------[ cut here ]------------
WARNING: CPU: 1 PID: 2867 at drivers/clk/clk.c:845 __clk_disable+0x5b/0x60()
[<ffffffff810751fa>] warn_slowpath_null+0x1a/0x20
[<ffffffff81623a5b>] __clk_disable+0x5b/0x60
[<ffffffff81624267>] clk_disable+0x37/0x50
[<ffffffffa03a9029>] dw_i2c_suspend+0x29/0x40
```

In this case, file an [I2C bug](https://docs.google.com/document/d/1Fk5KeTcEbWkXUu8uv39wR8B_PDMn5sqJSsxxIRNtrUA/edit#heading=h.111kx3o).

## 4 DEBUGGING SUSPEND/HIBERNATE ISSUES

4.1 State not available

4.1.1 Symptom

"freeze"/"mem"/"disk" is not listed in the output of cat `/sys/power/state`, thus the system cannot enter STI/STR/HTD state.

4.1.2 Debugging steps

1. For STI:
   - Check if the kernel is built with `CONFIG_SUSPEND`. If not, set it.
   - Check if the kernel boots with the kernel parameter `relative_sleep_states=1`. If yes, remove it.
2. For STR:
   - Check if the kernel is built with `CONFIG_SUSPEND`. If not, set it.
   - Check the dmesg output after boot for a line like `ACPI: (supports S0 S3 S4 S5)`. If S3 is not listed, it usually means that the platform does not have S3 support. If you are not sure, file a suspend/hibernate bug.
3. For HTD:
   - Check if the kernel is built with `CONFIG_HIBERNATION`. If not, please rebuild with `CONFIG_HIBERNATION=y`.
   - Check the dmesg output after boot for a line like `ACPI: (supports S0 S3 S4 S5)`. If S4 is not listed, it usually means the platform does not have S4 support. If you are not sure, file a suspend/hibernate bug.
4. If the problem still exists after the above tests, file a [suspend/hibernate bug](https://docs.google.com/document/d/1Fk5KeTcEbWkXUu8uv39wR8B_PDMn5sqJSsxxIRNtrUA/edit#heading=h.111kx3o) `[platform] {STI, STR, HTD}: state not available`, together with the dmesg output, [acpidump](https://docs.google.com/document/d/1Fk5KeTcEbWkXUu8uv39wR8B_PDMn5sqJSsxxIRNtrUA/edit#heading=h.26in1rg) output, and kernel config file you're using.

4.2 Failed to suspend/hibernate

4.2.1 Symptom

The command, `echo freeze/mem/disk > /sys/power/state` returns with an error code.

4.2.2 Debugging steps

1. Check if this is a regression.
2. If not, enable initcall_debug and no_console_suspend.
3. If you find some device callback returns an error code instead of 0, that means that the device cannot be suspended, which causes the system suspend/hibernate to yield. In this case, file a bug to the driver/component owner.
4. If not, file a suspend/hibernate bug `[Platform] {STI, STR, HTD}: failed to suspend`, with both the dmesg output after the suspend failure as well as the error code returned.

4.3 System hangs during suspend/hibernate or resume

4.3.1 Symptom

After issuing `echo freeze/mem/disk > /sys/power/state`, the screen goes black like a successful suspend/hibernate, but then system becomes unresponsive, and cannot be woken up to a working state. This may happen during suspend or resume. Alternately, the system goes into a sleep state as expected, but does not respond when resuming, displaying only a black screen or crash log. For example, the fan starts to spin, and any existing power button backlight changes, but the system never goes back to a working state.

4.3.2 Debugging steps

1. Check if this is a regression.
2. If not, check if this is a graphics issue.
3. If not, check if this is a driver specific issue.
4. If the problem still exists, file a suspend/hibernate bug `[Platform] {STI, STR, HTD}: system hangs during suspend/resume`, together with the serial console output/screenshot after the problem was reproduced, the contents of `/var/log/kern.log`, and the test results of disk mode (HTD only), pm_test, pm_async, and rtc trace.

4.4 System reboots during suspend/resume

4.4.1 Symptom

Screen goes dark, and a fresh boot starts without users doing anything.

4.4.2 Steps

1. Check if this is a regression.
2. Check if this is a graphics issue.
3. Check if this is a driver specific issue.
4. If no, file a suspend/hibernate bug `[Platform] {STI, STR, HTD}: system reboot during suspend/resume`, together with the serial console output after the problem is reproduced: /var/log/kern.log, and the test results of disk mode (HTD only), pm_test, pm_async and rtc trace.

4.5 System errors after resume

4.5.1 Symptom

The system can suspend to and resume from a STI/STR/HTD state, but some components or driver may not be functional or might have have errors or warnings in dmesg after resume.

4.5.2 Debugging steps

1. Check if this is a regression.
2. If not, check if this is a driver specific issue.
3. If the problem still exists, file a suspend/hibernate bug `[Platform] {STI, STR, HTD}: xxx broken after resume` together with the dmesg output after resume.

4.6 System wakes up immediately after suspend

4.6.1 Symptom

The system comes back to a working state soon after suspending, without the user doing anything. And dmesg shows the system has done a complete suspend/resume cycle.

4.6.2 Debugging steps

1. Check if this is a regression.
2. Check if this is a driver specific issue by removing any unnecessary peripherals like USB drives, net cables, etc.
3. If not, file a suspend/hibernate bug `[Platform] {STI, STR, HTD}: system wakes up immediately after suspend`, together with acpidump output, dmesg after the immediate resume, the content of `/proc/interrupts` both before and after suspend and test result of ACPI wakeup.

4.7 System cannot be woken up

4.7.1 Symptom

The system does not wake up when you press the power button, use the keyboard, etc.

4.7.2 Debugging steps

1. Check if this is a regression. For STI, check if the driver has its own managed interrupt. If it does, check if enable_irq_wake() is invoked for the interrupt used by the driver. If it is not, add it in driver .probe() or .suspend() callback. For example:

   ```
   static int bu21013_suspend(struct device *dev)
   {
       ...
       if (device_may_wakeup(&client->dev))
           enable_irq_wake(bu21013_data->irq);
       else
           disable_irq(bu21013_data->irq);
       ...
       return 0;
   }
   ```

2. If the problem still exists, file a suspend/hibernate bug for `[Platform] {STI, STR, HTD}: XXX cannot wake up the system`. Make sure to include dmesg, acpidump, and test results of ACPI wakeup along with the bug report.

4.8 Long latency during suspend/resume (STI/STR/Hibernation)

4.8.1 Symptom

The overall/specific driver suspend/resume time is longer than expected/required.

4.8.2 Debugging steps

1. If you have already identified a specific driver that takes too much time to suspend/resume, file a bug to the driver owner.
2. If the overall suspend/resume latency is high, do the following:
   1. Reboot with the initcall_debug and no_console_suspend boot options.
   2. Disable async suspend/resume.
   3. Redo the suspend/resume using analyze_suspend.
   4. Check the test results and find which drivers and components are taking the most time during suspend/resume and report the issues to the driver/component owners, along with the test results of analyze_suspend.

4.9 Suspend/hibernate fails occasionally

4.9.1 Symptom

The symptom can be any of the symptoms listed above, but the difference is that these issues are not 100% reproducible. Sometimes it may take tens or even hundreds of suspend/hibernate cycles to reproduce.

4.9.2 Debugging steps

1. For HTD,
   1. Check the driver used for your swap partition.
   2. If the driver is build as module, build the driver in and check if the problem still exists.
   3. If no, file a suspend/hibernate bug for `[Platform] HTD: hibernate fails when disk driver is built as module`.
   4. If yes, please continue debugging with the following steps.
2. Reboot with initcall_debug, no_console_suspend, ignore_loglevel, dynamic debug, and serial console.
3. Use rtcwake to perform the stress testing.
4. Follow the instructions in the section above that best describes the symptom when suspend/hibernate fails to verify and raise the problem.

## 5 RAISE AN ISSUE

If you have narrowed down a problem, or found the root cause of the problem to be a driver specific issue:

- If you know where to raise the issue for the specified driver, raise it there directly. For example, for graphics issues, file a bug report at https://bugs.freedesktop.org/.
- If you’re not clear where to raise the issue, the preferred way is to subscribe to the subsystem/driver mailing list after checking http://vger.kernel.org/vger-lists.html and raise the problem there, please do CC [Linux-pm@vger.kernel.org](mailto:Linux-pm@vger.kernel.org) when raising the problem.

If it is not a driver specific issue, or you’re not sure what the problem is after debugging, file a bug report at [https://bugzilla.kernel.org/enter_bug.cgi?product=Power%20Management](https://bugzilla.kernel.org/enter_bug.cgi?product=Power Management). Set the Component to “Hibernation/Suspend” and include the debug information and test results required.