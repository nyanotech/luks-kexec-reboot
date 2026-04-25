# luks-kexec-reboot

Script to do a reboot of a headless luks-encrypted linux machine using kexec.

This script generates a keyfile, crafts an initramfs that unlocks your
disks using that keyfile, and then boots into that initramfs via kexec.

This way, key material only ever lives in memory, and is never written to disk.

## Supported Configurations

Right now, this thing only supports the configuration I made it for:

- **an initramfs that will read `/etc/crypttab` from inside the initramfs**
  - Arch linux with the `sd-encrypt` `mkinitcpio` hook does this
- every luks volume is in `/etc/crypttab` or `/etc/crypttab.initramfs`
  (no depending on `rd.luks` or similar params in kernel cmdline)
- **systemd-boot (optional)** - bootctl is used to autodiscover the existing
  boot configuration, but it's also possible to feed in your kernel, initramfs,
  etc manually

Untested, but might work:

- luks volumes that you normally open using a keyfile
- having luks1 volumes in your crypttab

Won't work (unimplemented):

- volumes you unlock with something other than a passphrase or keyfile (eg,
  tpm)
- signing for kexec secure boot

**Do not use this** if you are:

- using the 7th keyslot on any of your luks devices. Currently, this script
  always overwrites the 7th keyslot, rather than implementing some way to clear
  the created keyslots after reboot.

## Dependencies

- Root.
- Tools: `cryptsetup`, `cpio`, `kexec` (kexec-tools), `systemd-ask-password`,
  `bootctl`, `blkid`, `jq`, `lsblk`.
- Kernel that can kexec things
  - and hasn't had that functionality disabled for security
  - and can load unsigned initramfs

## What it does

- makes a temp working dir in a tmpfs
- generates a random keyfile in the working dir
- take `/etc/crypttab` or `/etc/crypttab.initramfs`, make a copy, for every
  entry in the copy, if it doesn't have a keyfile listed, list ours as the
  keyfile
- append the keyfile and modified crypttab to a copy of the initramfs
- prompts once for your luks passphrase
- overwrites keyslot number 7 on every luks volume in `/etc/crypttab` or
  `/etc/crypttab.initramfs` to use the newly-made keyfile
- load up the modified initramfs, and the discovered kernel, cmdline, etc
  into kexec
- remove the tmpfs dir we did all this in

## Usage

```
sudo luks-kexec-reboot               # use systemd-boot default entry
sudo luks-kexec-reboot --dry-run     # parse + build, don't touch LUKS/kexec
sudo luks-kexec-reboot --entry arch.conf
sudo luks-kexec-reboot --kernel /boot/vmlinuz-linux \
                      --initrd /boot/amd-ucode.img \
                      --initrd /boot/initramfs-linux.img \
                      --cmdline "$(cat /proc/cmdline)"

systemctl kexec        # trigger the reboot
kexec -u               # or cancel the staged image
```

## Acknowledgements

This is more or less a reimplementation of
[keyexec](https://github.com/flowztul/keyexec), whose idea I'm
stealing.
