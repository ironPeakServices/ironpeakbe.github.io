---
date: "2020-10-05T09:56:54+02:00"
title: "Crouching T2, Hidden Danger"
layout: "blog"
draft: false
---

**Let's talk about that thing nobody's talking about.
Let's talk about that vulnerability that's completely exposing your macOS (and iPad Pro) devices while news agencies and Apple (!) are declining to act and report about the matter.
Oh, and did I mention it's unpatcheable?
Settle in buckaroo, we're in for a wild ride.**

Skip to [#security-issues](#security-issues) for the technical mumbo-jumbo.

## Preface

### Intel vs Silicon

This blog post only applies to macOS systems with an Intel processor and the secondary T2 chip.
Apple silicon systems will run completely on a set of ARM processors designed by Apple and thus will use a different boot topology.
Because of this it's possible Apple silicon systems will not be impacted by this vulnerability, but this is yet to be seen.
And besides... let's hope it's fixed by then. :-)

### So about this T2 thing

In case you are using a recent macOS device, you are probably using [the embedded T2 security chip](https://support.apple.com/en-us/HT208862) or *Secure Enclave Processor* (SEP).
This is a custom ARM processor designed by Apple and based on the A10 ARM processor found in iphones.
It performs a predefined set of tasks for macOS such as audio processing, handling I/O, functioning as a [Hardware Security Module](https://en.wikipedia.org/wiki/Hardware_security_module) for e.g. Apple KeyChain, hardware accelerating media playback & cryptographic operations and **ensuring the operating system you are booting is not tampered with**.
The T2 chip runs its own firmware called *bridgeOS*, which can be updated when you install a new macOS version. (ever notice the screen flickering? that's the display driver being interrupted.)

### The macOS boot sequence

So let's focus on the boot image verification on macOS. What exactly happens when you press that power button?
[There's also a visual representation for any *conaisseurs*](https://eclecticlightdotcom.files.wordpress.com/2018/08/bootprocess.png).

1. The press of the power button or the opening of the lid triggers the System Management Controlle (SMC) to boot.

2. The SMC performs a Power-On-Self-Test (POST) to detect any EFI or hardware issues such as bad RAM and possibly redirect to Recovery.

3. After those basic sanity checks, the T2 chip is initialized and I/O connectors are setup. (USB, SATA, NVMe, PCIe, ...)

4. The next boot disk is selected and a disk encryption password is asked if enabled to mount [APFS](https://en.wikipedia.org/wiki/Apple_File_System) volumes.

5. `/System/Library/CoreServices/boot.efi` is located on your System APFS volume and [depending on your secure boot settings](https://support.apple.com/en-us/HT208330) is validated.

6. *boot.efi* is ran which loads the Darwin kernel *(throwback to BSD)* (or Boot Camp if booting Microsoft Windows) & IODevice drivers. If a kernel cache is found in `/System/Library/PrelinkedKernels/perlinkedkernel`, it will use that.

7. Any User Approved Kernel Extensions are initialized & added to the kernel space. *This will go away with System Extensions*.

### macOS security features

So Apple has a couple of tricks up its sleeve to limit the attack surface of any potential security vulnerabilities. A small summary of related measures since macOS Big Sur on Intel processors:

- *Signed System Volume* (SSV): a read-only `/System` partition so the base install of macOS (including the kernel) cannot be tampered with thanks to *System Integrity Protection* (SIP).

- *System Extensions*: a move to away from Kernel Extensions, getting external code out of the Kernel framework-wise.

- *Secure Boot*: verifies the signature validity of the operating system on disk.

- *Filesystem seals*: every byte of data is compared to a hash in the filesystem metadata tree, recursively verifying integrity.

### Apple marketing

As you probably all already know, Apple pushes forward privacy & security as important weapons in todays world of technology.
They tout their devices as highly secure and vouch to handle your personal data using a privacy-centric approach.
While there have been mistakes made in the past (who can blame them?), Apple has been generally quick to fix any security issues that were disclosed to [their responsible disclosure program](https://support.apple.com/en-gb/HT201220) or in public.

## Security issues

### The core problem

The mini operating system on the T2 (*SepOS*) suffers from a security vulnerable also found in the iPhone X since it contains an A10 processor.
Using the [checkm8 exploit](https://checkm8.info) originally made for iPhones, the checkra1n exploit was developed to build a semi-thetered exploit.
This can be used to e.g. circumvent activation lock, allowing stolen iPhones or macOS devices to be reset and sold on the black market.

Normally the T2 chip will exit with a fatal error if it is in DFU mode and it detects a decryption call, but thanks to the [blackbird vulnerability](https://github.com/windknown/presentations/blob/master/Attack_Secure_Boot_of_SEP.pdf) by team Pangu, we can completely circument that check and do whatever we please. 

### Debugging vulnerability

Apple left a debugging interface open in the T2 security chip shipping to customers, allowing anyone to enter Device Firmware Update (DFU) mode without authentication.
An example cable that can be used to perform low-level CPU & T2 debugging is the [BONOBO JTAG/SWD DEBUG CABLE](https://shop.lambdaconcept.com/home/37-bonobo-debug-cable.html). See an example [in this Twitter post](https://twitter.com/h0m3us3r/status/1280432544731860993).

Using this method, it is possible to create an USB-C cable that can automatically exploit your macOS device on boot. **(!)**

### Impact

Once you have access on the T2, you have full `root` access and full kernel execution privileges since the kernel is rewritten before execution.
Good news is that if you are using FileVault 2 as disk encryption, attacks still cannot decrypt your disks.
They can however inject a keylogger in the T2 firmware since it manages keyboard access, storing your password for retrieval.

A firmware password does not mitigate this issue since it requires keyboard access, and thus needs the T2 chip to run first.

## Exploitation

```bash
# install devtools
$ xcode-select --install

# check the script & install homebrew
$ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"

# install packages
$ brew install libplist automake autoconf pkg-config openssl libtool llvm libusb

# git clone, autogen.sh, make & make install
# https://github.com/sbingner/ldid
# https://github.com/libimobiledevice/libusbmuxd
# https://github.com/libimobiledevice/libimobiledevice
# https://github.com/libimobiledevice/usbmuxd

# Run checkra1n and wait for T2 boot. It will stall when complete.
...

# Unplug and replug the usb connection. Checkra1n should now send the overlay.
...

# Bring up a proxy to dropbear
$ iproxy 2222 44 &

# Connect to T2 & enjoy
$ ssh -p 2222 root@127.0.0.1
```

## And more to come

I have sources that say more news is on the way coming weeks. I quote my source: *be afraid, be very afraid*.

## Reaching out

I've reached out to Apple concerning this issue on numerous occasions, even doing the dreaded cc *tcook@apple.com* to get some exposure.
Since I did not receive a response for weeks, I did the same to numerous news websites that cover Apple, but no response there as wel.
In hope of raising more awareness (and an official response from Apple), I am hereby disclosing almost all of the details.

## So what now as...

### ...a user

If you suspect your system to be tampered with, use Apple Configurator to reinstall bridgeOS on your T2 chip described [here](https://mrmacintosh.com/how-to-restore-bridgeos-on-a-t2-mac-how-to-put-a-mac-into-dfu-mode/). If you are a potential target of state actors, verify your SMC payload integrity using .e.g. [rickmark/smcutil](https://github.com/rickmark/smcutil) and **don't** leave your device unsupervised.

### ...a mac sysadmin

Contact your Apple rep & wait for official news from Apple. Don't use the T2 chip for any credentials for now. (such as MFA)
Raise awareness to your users to not leave their device lingering.

### ...a security professional

Wait for a fix, keep an eye on the checkra1n team and be prepared to replace your macOS system.
Be angry at news websites & Apple for not covering this issue, despite numerous attempts from me and others to get them to report about this.

<br>
<br>
<br>
<br>

<br>
<br>

## TL;DR

**TL;DR: all recent macOS devices & the iPad Pro are no longer safe to use if left alone, even if you have them powered down.**

- The root of trust on macOS is inherently broken
- They can decrypt your FileVault2 volumes
- They can alter your macOS installation


## References

- Big thanks to the checkra1n team and specifically [Rick Mark](https://github.com/rickmark/) to bring this to light.
- Checkra1n website: [checkra1n](https://checkra.in/)
- checkra1n t2 OS replacement: [checkra1n/pongoOS](https://github.com/checkra1n/pongoOS)
- smcutil: [rickmark/smcutil](https://github.com/rickmark/smcutil/)
- [How to restore your T2 chip firmware](https://support.apple.com/guide/apple-configurator-2/revive-or-restore-mac-firmware-apdebea5be51/mac)
- First (and only?) article about it: [yalujailbreak.net - T2 Security Chip Jailbreak](https://yalujailbreak.net/t2-security-chip-jailbreak/)