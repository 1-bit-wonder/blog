---
title: How to use Yubikey with WSL2 via USB passthrough (or how I compiled my first custom Linux kernel)
date: "2022-10-07"
---

<!-- # How to use YubiKey with WSL2 via USB passthrough

## (or how I compiled my first custom Linux kernel) -->

As someone who tends to be fairly paranoid when it comes to online security, I like the idea of using a hardware-based authentication device to store keys safely for things like code signing and SSH access. 

Unfortunately, this turned out to be a bit painful to get working with WSL. After struggling for days on end to follow any tutorial with success, eventually giving up for a few months and going back and forth between a dual-boot of OpenSUSE, I finally decided to take a crack at it again.

With existing tutorials being seemingly useless, I had to figure out a method that didn't involve forwarding the gpg/ssh-agent from Windows to WSL using a .exe from an unmaintained project on GitHub. Here's what I came up with:   

### Before we get started, some requirements:

- YubiKey 5 * Series or later
- Windows 11
- WSL2 Version >= 0.67.6 running Ubuntu 20.04 (other distro/version may also work, I haven't tested)

### Getting USB passthrough set up

USB passthrough works via [usbipd-win](https://github.com/dorssel/usbipd-win) which allows for sharing locally connected USB devices to other machines, including Hyper-V guests and WSL2.

First, we must install the windows package, I used the Windows Package Manager.

```powershell
winget install usbipd
```

But you can also find an installer for the latest release [here](https://github.com/dorssel/usbipd-win/releases/latest).

Onto the WSL side, running `uname -a` from within WSL should report a kernel version of 5.10.60.1 or later. Now let's install the user space tools for USB/IP on Linux and a database of USB hardware identifiers.

```bash
$ sudo apt install linux-tools-virtual hwdata
$ sudo update-alternatives --install /usr/local/bin/usbip usbip `ls /usr/lib/linux-tools/*/usbip | tail -n1` 20
```

### udev

It's recommended to configure udev rules to allow non-root users to access the device. Yubico provides some rules for their cards [here](https://github.com/Yubico/libfido2/blob/main/udev/70-u2f.rules) that you can copy to your /etc/udev/rules.d folder.

```bash
$ sudo wget https://github.com/Yubico/libfido2/blob/main/udev/70-u2f.rules > /etc/udev/rules.d/70-u2f.rules
```

Once udev rules are in place, we need to restart WSL. From a PowerShell do

```powershell
wsl --shutdown
```

### Checking usbip for info about devices on the Windows host

These utility commands can be ran from command line with either usbip on WSL or usbpid on Windows. Just note, the usage varies slightly, so check the docs. I like to run the commands from within WSL. First let's get the host IP address from /etc/resolv.conf.

```bash
$ cat /etc/resolv.conf
# This file was automatically generated by WSL. To stop automatic generation of this file, add the following entry to /etc/wsl.conf:
# [network]
# generateResolvConf = false
nameserver 172.29.176.1
```

 Grab the IP and use it with the remote flag for listing exportable USB devices on the host.

```bash
$ usbip list -r 172.29.176.1
Exportable USB devices
======================
 - 172.29.176.1
       1-11: Yubico.com : Yubikey 4 OTP+U2F+CCID (1050:0407)
           : USB\VID_1050&PID_0407\6&20381000&0&11
           : (Defined at Interface level) (00/00/00)
           :  0 - Human Interface Device / Boot Interface Subclass / Keyboard (03/01/01)
           :  1 - Human Interface Device / No Subclass / None (03/00/00)
           :  2 - Chip/SmartCard / unknown subclass / unknown protocol (0b/00/00)
```

### Installing YubiKey utilities and attaching the device

Let's install the YubiKey manager command line tool from the package repository.

```bash
$ sudo apt install yubikey-manager
```

We can see what commands are available to us with `ykman`

```bash
Usage: ykman [OPTIONS] COMMAND [ARGS]...

  Configure your YubiKey via the command line.

  Examples:

    List connected YubiKeys, only output serial number:
    $ ykman list --serials

    Show information about YubiKey with serial number 0123456:
    $ ykman --device 0123456 info

Options:
  -v, --version
  -d, --device SERIAL
  -l, --log-level [DEBUG|INFO|WARNING|ERROR|CRITICAL]
                                  Enable logging at given verbosity level.
  --log-file FILE                 Write logs to the given FILE instead of standard error; ignored unless --log-level
                                  is also set.
  -r, --reader NAME               Use an external smart card reader. Conflicts with --device and list.
  -h, --help                      Show this message and exit.

Commands:
  config   Enable/Disable applications.
  fido     Manage FIDO applications.
  info     Show general information.
  list     List connected YubiKeys.
  mode     Manage connection modes (USB Interfaces).
  oath     Manage OATH Application.
  openpgp  Manage OpenPGP Application.
  otp      Manage OTP Application.
  piv      Manage PIV Application.
```

Attach the device with usbip from Linux so we can use this utility to get more info about it. We can get both the IP for the remote host, as well as the bus id from the `usbip list` command we ran above.

```bash
$ sudo usbip attach -r 172.29.176.1 --busid=1-11
```

If all goes well, we should be able to query the card now with `ykman`

```bash
$ ykman list
YubiKey 5 NFC [OTP+FIDO+CCID] Serial: XXXXXXXX

ykman info
Device type: YubiKey 5 NFC
Serial number: XXXXXXXX
Firmware version: 5.4.3
Form factor: Keychain (USB-A)
Enabled USB interfaces: OTP+FIDO+CCID
NFC interface is enabled.

Applications    USB     NFC
OTP             Enabled Enabled
FIDO U2F        Enabled Enabled
OpenPGP         Enabled Enabled
PIV             Enabled Enabled
OATH            Enabled Enabled
FIDO2           Enabled Enabled
```

As one my main use cases for the YubiKey was to authenticate to GitHub and other services via SSH, I'm interested in using FIDO2 for storing resident keys. Let's try to get some info about what keys I have already stored.

```bash
$ ykman fido list
Traceback (most recent call last):
  File "/usr/bin/ykman", line 11, in <module>
    load_entry_point('yubikey-manager==3.1.1', 'console_scripts', 'ykman')()
  File "/usr/lib/python3/dist-packages/ykman/cli/__main__.py", line 273, in main
    cli(obj={})
  File "/usr/lib/python3/dist-packages/click/core.py", line 764, in __call__
    return self.main(*args, **kwargs)
  File "/usr/lib/python3/dist-packages/click/core.py", line 717, in main
    rv = self.invoke(ctx)
  File "/usr/lib/python3/dist-packages/click/core.py", line 1137, in invoke
    return _process_result(sub_ctx.command.invoke(sub_ctx))
  File "/usr/lib/python3/dist-packages/click/core.py", line 1137, in invoke
    return _process_result(sub_ctx.command.invoke(sub_ctx))
  File "/usr/lib/python3/dist-packages/click/core.py", line 956, in invoke
    return ctx.invoke(self.callback, **ctx.params)
  File "/usr/lib/python3/dist-packages/click/core.py", line 555, in invoke
    return callback(*args, **kwargs)
  File "/usr/lib/python3/dist-packages/click/decorators.py", line 17, in new_func
    return f(get_current_context(), *args, **kwargs)
  File "/usr/lib/python3/dist-packages/ykman/cli/fido.py", line 112, in list_creds
    controller = ctx.obj['controller']
  File "/usr/lib/python3/dist-packages/ykman/cli/util.py", line 127, in __getitem__
    self.resolve()
  File "/usr/lib/python3/dist-packages/ykman/cli/util.py", line 124, in resolve
    self._objects[k] = f()
  File "/usr/lib/python3/dist-packages/ykman/cli/__main__.py", line 194, in resolve_device
    dev = _run_cmd_for_single(ctx, subcmd.name, transports, reader)
  File "/usr/lib/python3/dist-packages/ykman/cli/__main__.py", line 132, in _run_cmd_for_single
    return descriptor.open_device(transports)
  File "/usr/lib/python3/dist-packages/ykman/descriptor.py", line 96, in open_device
    for drv in _list_drivers(transports):
  File "/usr/lib/python3/dist-packages/ykman/descriptor.py", line 164, in _list_drivers
    for dev in open_fido():
  File "/usr/lib/python3/dist-packages/ykman/driver_fido.py", line 97, in open_devices
    for dev in CtapHidDevice.list_devices(descriptor_filter):
  File "/usr/lib/python3/dist-packages/fido2/hid.py", line 135, in list_devices
    for d in hidtransport.hid.Enumerate():
  File "/usr/lib/python3/dist-packages/fido2/_pyu2f/linux.py", line 183, in Enumerate
    for hidraw in os.listdir('/sys/class/hidraw'):
FileNotFoundError: [Errno 2] No such file or directory: '/sys/class/hidraw'
Exception: ykman exited with 1
[tty 4], line 1: ykman fido list
```

Uh oh, something's not quite right...

### But what does it all mean?

Looking at the traceback, we can see the ykman script fails due to not being able to find `'/sys/class/hidraw'`. Because USB support in WSL is quite new, it seems [not all](https://github.com/microsoft/WSL/issues/7686#issuecomment-1027437615) of the HID drivers have been enabled yet in the kernel. So what does this mean for us?

### Giving you the keys to the Lamborghini 

As a budding Linux script kiddie, I was always awestruck reading accounts of people compiling their own kernels for system optimization and support for esoteric hardware. Back then it seemed like a task fit only for the most capable practitioners of the *nix dark arts. 

Today, thanks to the wonderful folks working on open source projects at Microsoft and elsewhere, it is ridiculously simple to get a copy of their kernel source, modify and load it into WSL with no fear of bricking your whole system. Let's do that now. First we need to install some build dependencies and pull in the kernel source.

```bash
$ sudo apt install build-essential flex bison libssl-dev libelf-dev
$ git clone https://github.com/microsoft/WSL2-Linux-Kernel.git
```

Once the source finishes downloading, `cd` into the directory and create a copy of Microsoft's build config to the root so we have a backup to revert to should things go awry. 

```bash
$ cp Microsoft/config-wsl .config
```

We have a couple options available to us for modifying the config. You can edit the config manually in your text editor, I opted to use the graphical configuration editor included with `make`, which in this case was a bit overkill because I only needed to change one line. For brevity, let's do it the easy way. Open up the `.config` file in an editor. 

```bash
$ nano .config
```

Find the line that says

```
# CONFIG_HIDRAW is not set
```

And replace with

```
CONFIG_HIDRAW=y
```

Now it's time to compile our kernel! Run `make` and wait...

```bash
$ make
...
...
...
Kernel: arch/x86/boot/bzImage is ready  (#1)
```

If you see the above message, give yourself a pat on the back for a job well done, you just compiled a Linux kernel! Let's copy that image to our Windows filesystem so that we can tell WSL where to load it from. 

```bash
$ cp arch/x86/boot/bzImage /mnt/c/Users/<your_windows_user>/custom-wsl-kernel
```

### systemd

While it may not be strictly necessary, as Microsoft does have it's own way of managing what services start up at boot, systemd has become the standard across most distros these days. I recommend this step, because this is what worked for me. Systemd support is also a [new feature](https://devblogs.microsoft.com/commandline/systemd-support-is-now-available-in-wsl/) as of WSL2 version 0.67.6. You can check your version by running `wsl --version` and `wsl --update` if you are behind. Enabling systemd is as easy as creating or editing your `/etc/wsl.conf`. Add these lines:

```
[boot]
systemd=true
```

### One last thing

With that out of the way, we can shut down WSL again to take care of a few final things on Windows. 

```powershell
wsl --shutdown
```

The only thing left to do now is edit our `.wslconfig` to tell the system where our kernel is. Open up `C:\Users\<your_user>\.wslconfig` and add

```
[wsl2]
kernel=C:\\Users\<your_user>\custom-wsl-kernel
```

That's it!

### And now for the moment of truth...

Time to open up WSL and test it out. Let's try running our `ykman` utility again to check out what it knows about FIDO on our card.

```bash
# Attach the YubiKey again
$ sudo usbip attach -r 172.29.176.1 --busid=1-11

# Check what we get
$ ykman fido list
Enter your PIN:
ni@nunya.com (login.microsoft.com)
openssh (ssh:)
$ ykman fido info
PIN is set, with 8 tries left.

```

It works! Feel free to do a little victory dance. Now you can [generate/add](https://www.yubico.com/blog/github-now-supports-ssh-security-keys/) your own resident keys and use FIDO2 from your WSL guests. 
