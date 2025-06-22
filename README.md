# NixOS Time Machine

This is my NixOS configuration for a macOS Time Machine compatible backup server. I use it on my old HP Data Vault NAS to extend its lifetime (see my [blog post](https://blog.matoutech.dev/nixos-time-machine/) for the full story). So you should be able to use this as a template for pretty much any system that can run NixOS given enough storage for backups.

## Partition Layout / RAID Setup

My System has two 1 TB and two 1.5 TB HDDs for storage as well as an external SSD for system files. NixOS is installed to the SSD according to the [manual](https://nixos.org/manual/nixos/stable/#sec-installation-manual) but the bootloader lives on the HDD in the first slot because the BIOS boots from it by default. You can get the device ID for `boot.loader.grub.device` in your NixOS config by using `ls -la /dev/disk/by-id` and checking where the IDs are pointing to in combination with the output from `lsblk`.

All HDDs are formatted as MBR with a single partition of type `0xDA` (non-FS data) each. I decided to create two RAID 1 arrays using the information provided by `lsblk`.

```bash
mdadm --create --verbose --level=1 --metadata=1.2 --raid-devices=2 /dev/md/[array name] /dev/[first drive] /dev/[second drive]
```

Depending on your hardware and needs, you might choose a different RAID setup. Just remember to put the output of `mdadm --detail --scan` in your `configuration.nix` under `boot.swraid.mdadmConf`.

Both arrays have an ext4 file system with stride and stripe width optimized for the RAID setup.

```bash
mkfs.ext4 -L [volume name] -E stride=[stride],stripe-width=[stripe width] /dev/md/[array name]
```

The stride is typically 128. The stripe width is the stride times the number of disks in the array, excluding parity disks. You can find further information on the [ArchWiki](https://wiki.archlinux.org/title/RAID#Calculating_the_stride_and_stripe_width).

## Samba Share

The mount points for the file system must be owned by the group `sambashare` and it should have full permissions. Additionally, it is a good idea to set the `setgid` bit of the directories so that all new files and subdirectories will inherit group ownership.

```bash
chown :sambashare [mount point]
chmod 2775 [mount point]
```

When renaming the SMB share, replace `timemachine` with the new share name in the following line of the config under `services.avahi.extraServiceFiles`:

```xml
<txt-record>dk0=adVN=timemachine,adVF=0x82</txt-record>
```

Please also make sure to set an SMB password in order to access the samba share with your user after you've applied the configuration.

```bash
smbpasswd -a [user name]
```
