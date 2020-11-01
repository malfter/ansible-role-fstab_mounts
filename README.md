# ansible-role-drive_mounts

This Ansible role was created by me in order to easier manage a couple of
different programs who all need to create entries in `/etc/fstab` in a specific
order, and are therefore somewhat dependent on each other. This role can create
the following types of mounts:

- Boot drives
- Normal drives
- Encrypted drives
  - With options for loading the decryption keys from a remote host.
- Pooled drives (using [mergerfs][1] for pooling).


### Important Note
This Ansible role is not really "automatic", since it is necessary to perform
a few manual steps before this role can be run (like creating a filesystem on
the drives). So this is designed more as a method of "record keeping" which
drives that are present on each system, and where they are mounted. By version
managing your configs you can easily track the hardware changes made to your
systems.

At the end of this README there are some guides for the manual steps that are
necessary, so you may know what is going on at a more detailed level.



# Installation

This repository does not have any dependencies by itself, so just move into
your `roles/` folder and run the following:

```bash
git clone git@github.com:JonasAlfredsson/ansible-role-drive_mounts.git drive_mounts
```

If you would like to download any updates for this role in the future, you may
use the following command from within the previously cloned folder:

```bash
git pull
```

When the configuration is complete you may then just include this role in your
main playbook like this:

```yaml
- hosts: all
  name: Configure all drive mounts
  roles:
    - drive_mounts
```



# Usage

There are four "types" of mounts which can be managed with this role, and they
will all have their own section that explains how to properly configure them.
However, the only section that is required to be completed is the
"[boot drives](#boot-drives)" one, since otherwise you may end up in a state
where the computer won't be able to boot properly.

Furthermore, as stated in the introduction, this role requires you to manually
prepare the disks (formatting/encrypting them) so that you can obtain their
unique identifier ([UUID](#obtain-the-uuid)) and enter it here in the Ansible
configuration. If you need help creating these drives there are some nice
step-by-step guides available in the [Preparation Steps](#preparation-steps)
section.

If you know what you are looking for you can jump directly the the desired
section, otherwise just continue reading.

- [Boot Drives](#boot-drives)
- [Normal Drives](#normal-drives)
- [Encrypted Drives](#encrypted-drives)
  - [The `cryptdisks` Mount](#the-cryptdisks-mount)
- [Pooled Drives](#pooled-drives)
  - [Install `mergerfs`](#install-mergerfs)

> NOTE: Some extra information about Ansible variables may be found
        [here](#ansible-variables-location).


## Boot Drives
Before doing anything else to the system you are trying to apply this role to
we need to define the `drives_boot` variable. SSH to the destination server and
type the following:

```bash
cat /etc/fstab
```

This is necessary since we need to find the entries which were there since the
installation of the system. For me it looked like this:

```
#   <file system>                       <mount point> <type> <options>          <dump> <pass>
# / was on /dev/sda1 during installation
UUID=88f0236b-620a-456e-a4a6-1b5c84996b5c  /           ext4   errors=remount-ro    0      1
# swap was on /dev/sda5 during installation
UUID=b8d210eb-d186-405f-bc42-a55cbe069a27  none        swap   sw                   0      0
```

This means we need to create the following Ansible variable before creating
entries for any other mounts (the order is important here):

```yaml
drives_boot:
  - uuid: "88f0236b-620a-456e-a4a6-1b5c84996b5c"
    mount_point: "/"
    type: "ext4"
    options: "errors=remount-ro"
    dump: 0
    pass: 1
  - uuid: "b8d210eb-d186-405f-bc42-a55cbe069a27"
    mount_point: "none"
    type: "swap"
    options: "sw"
    dump: 0
    pass: 0
```

This is the minimal configuration necessary in order to use this role, and here
all the individual variables needs to be explicitly set. The drives defined here
are also placed in the top of the `fstab` file, in order to be mounted first.

> If your `fstab` file is located somewhere else you can also define the
  `fstab_path` variable to point to the correct location. Look in the
  [`defaults/main.yml`](./defaults/main.yml) file for more "global" variables.


## Normal Drives
After you have added any "boot" drives, you may then add additional "normal"
drives to `fstab`. The procedure is very similar to the previous
"[Boot Drives](#boot-drives)" step, except there are now some default values
present, which means you can write a more compact list if you want. Any field
that is not marked with `# Required` may be left out of your configuration if
you are fine with the defaults.

```yaml
drives_normal:
  - uuid:  # Required
    mount_point:  # Required
    type: "ext4"
    options: "defaults,nofail"
    dump: 0
    pass: 2
    comment: ""
```

> Help for obtaining the UUID may be found [here](#obtain-the-uuid).

These drives are in the third place in the `fstab` file, after the
[boot drives](#boot-drives) and the [cryptdisks mount](#the-cryptdisks-mount).


## Encrypted Drives


This role can also handle mounting encrypted partitions/drives as well. The
configuration options are a bit different from the previous
["normal" drives](#normal-drives), and this is because the devices will first
have to be "unlocked" by [`cryptsetup`][10] before it can be mounted as usual
([simple visualization](#mount-it)).

Here are the options for encrypted drives, where the defaults are entered and
any field which is not marked with `# Required` may be left out of your
configuration if you are fine with the defaults.

> IMPORTANT: Here the user defined `label` field needs to be unique, just like
             the UUID.

```yaml
drives_encrypted:
  - label:  # Required
    uuid:  # Required
    mount_point:  # Required
    crypttab:
      key_file:  # Required
      options: "luks,nofail"
      comment: ""
    fstab:
      type: "ext4"
      options: "defaults,nofail"
      dump: 0
      pass: 2
      comment: ""
```

> Help for obtaining the UUID may be found [here](#obtain-the-uuid)

These drives are in the fourth place in the `fstab` file, after the
the [cryptdisks mount](#the-cryptdisks-mount) and the
["normal" drives](#normal-drives).

### The Cryptdisks Mount

If the `key_file` (in the settings above) is located on a USB key or network
drive, which needs to be mounted **before** anything else can be mounted, you
will have to specify the `CRYPTDISKS_MOUNT` within the `/etc/default/cryptdisks`
file. This can be done by just writing this:

```yaml
cryptdisks_mount:
  mount_point: "/mnt/usb/drive"
  uuid: "37033980-8380-476d-a851-840815c328b1"
  type: "ext4"
  options: "defaults,noexec"
  dump: 0
  pass: 2
```

or if it is on a Samba network drive which needs credentials:

```yaml
cryptdisks_mount:
  mount_point: "/mnt/network/drive"
  net_path: "//192.168.0.1/secret_keys"
  net_credentials:
    username: "user"
    password: "user_smb_pass"
    path: "/root/.smbcredentials"
  type: "cifs"
  options: "vers=3.11,uid=root,gid=root,_netdev,noexec"
  dump: 0
  pass: 0
```

This mount point is quite important, so it is in the second place in the
`fstab` file, right after the [boot drives](#boot-drives).


## Pooled Drives
"Pooling" drives means that you make multiple separate disks look like a single
one for the computer. Important to know that "pooling" is **not** RAID, since
this method basically only places single whole files on the different drives
available in the pool.

The program making this possible is [`mergerfs`][1], and first of all you will
need to define which version of it you want to be installed. Choose one from
the list [here][17], and define it like this:

```yaml
mergerfs_version: "2.29.0"
```

There are a ton of [options][18] available for `mergerfs`, so I suggest you
read up on them for best experience. However, I have supplied some sane
defaults here that most people could live with. Any field that is not marked
with `# Required` may be left out of your configuration if you are fine with
the defaults.

```yaml
drives_pooled:
  - name: ""
    options: "defaults,allow_other,use_ino"
    dump: 0
    pass: 0
    mount_point:  # Required
    branches: []  # Required
```

The `branches` variable is a list of strings that correspond to mount points
that have been defined in either the [normal](#normal-drives) or the
[encrypted](#encrypted-drives) drives sections.

#### Example:
```yaml
drives_pooled:
  - name: "first_pool"
    mount_point: "/mnt/pool"
    branches:
      - "/mnt/disk1"
      - "/mnt/disk2"
      - "/mnt/disk3"
```

These entries are located at the end of the `fstab` file, so that it is possible
to add any of the previous types of mounts to a pool.



# Preparation Steps

In this section you will find step-by-step guides on how to prepare the raw
drives in order for them to be usable by the different mounting options
supported by this role.

These steps might not be necessary for you to do, or you have another preferred
way of doing it, which is why these steps should be considered more of general
suggestions on how to do these things. If you want more advanced options you
will need to do some extra research by yourself.

- [Create Normal Drive](#create-normal-drive)
  1. [Create Partition Table](#create-partition-table)
  2. [Create Partition](#create-partition)
  3. [Create Filesystem](#create-filesystem)
  4. [Mount It](#mount-it)
- [Create Encrypted Drive](#create-encrypted-drive)
  1. [Create a Keyfile](#create-a-keyfile)
  2. [Encrypt Device/Partition](#encrypt-devicepartition)
  3. [Unlock Encrypted Device](#unlock-encrypted-device)
  4. [Create Normal Filesystem](#create-normal-filesystem)
  5. [Mount It](#mount-it-1)


## Create Normal Drive

1. [Create Partition Table](#create-partition-table) - Only once per drive.
2. [Create Partition](#create-partition) - As many partitions as you want.
3. [Create Filesystem](#create-filesystem) - Once per partition on the drive.
4. [Mount It](#mount-it) - Once per partition on the drive.

In order to be able to perform these steps you will need to find the physical
device you want to use located under `/dev/`. We will then launch [`parted`][9],
which can create both partition tables as well as partitions. So begin with
issuing the starting command:

```bash
sudo parted -a optimal /dev/sdX
```

You should now be inside the "`parted`" command prompt, so any commands made
here will be preceded by "`(parted)`" to indicate that we are still there.

> If you need to completely wipe the drive beforehand you could run the
  following command: \
  `sudo dd if=/dev/zero of=/dev/sdX bs=4M status=progress`

### Create Partition Table
A drive needs a partition table before any partitions can be made. This will
only have to be done once per drive, irregardless how many partitions you intend
to have on the drive later.

Unless you intend to install the drive in a computer that is >15 years old you
should use the "`gpt`" option, otherwise you may use "`msdos`" which will
install a [Master Boot Record][6] instead of the [GUID Partition Table][7].

```bash
(parted) mklabel gpt
```

### Create Partition
Now it is time to actually create a partition. It will be much easier if you
already know how many partitions you want on each drive, so you don't have to
fiddle around with this when there is important data on the drive.

In the first example I will just create a single partition which spans the
entire drive. It is [good practice][8] to leave a little room in the beginning
and the end of the drive for alignment purposes.

```bash
(parted) mkpart primary 1 -1
```

Or you can create two partitions.

```bash
(parted) mkpart primary 1 50%
(parted) mkpart primary 50% -1
```

> The "primary" name has no special meaning for GPT, but
  [is important in the case you are using MBR][11].

It is then possible to check so that the partitions are aligned before
continuing. Here I test both the first and the second partition.

```bash
(parted) align-check optimal 1
(parted) align-check optimal 2
```

In both these cases it should return "`# aligned`".

You may now quit `parted`.

```bash
(parted) quit
```

### Create Filesystem
Now you can create a filesystem on the partition. If you are using Linux you
most likely want to go with the `ext4` filesystem. Here we begin with a
bog standard method of doing it for the first partition.

```bash
sudo mkfs.ext4 /dev/sdX1
```

But for the second partition we don't want to [reserve][20] any of the space to
the super user, and we also want to reduce the number of [inodes][19] to only 1
per 4MiB.

```bash
sudo mkfs.ext4 -m 0 -T largefile4 /dev/sdX2
```

These are advanced options that you should read more into before using them, but
as a quick summary the `largefile4` option frees up some space on the drive if
you only intend to store few very large files (like a movie collection). The
`-m` option change the limit of how much a "normal" user may fill the disk, so
without this the last 5% of the drive would only be usable by "root". These
two options should remain untouched on the OS drive, otherwise you may lock up
the system.

### Mount It
To test if you can use the newly created filesystem you may now mount it.

```bash
sudo mount /dev/sdX1 /mnt/first-drive
```

Try going into the desired mount point and create a file, just to be certain
everything works as intended.



## Create Encrypted Drive
You can either target a raw drive, and make it use all the space just like a
single big encrypted partition, or you can target an already existing partition
(that was formed according to the steps in
[Create Normal Drive](#create-normal-drive)) which allows you to mix both
un-encrypted and encrypted partitions on the same drive. Choose the one you
feel best suite your usecase and then continue with this guide.

1. [Create a Keyfile](#create-a-keyfile) - One for each encrypted device.
2. [Encrypt Device/Partition](#encrypt-devicepartition) - As many encrypted devices you want.
3. [Unlock Encrypted Device](#unlock-encrypted-device) - Once per encrypted device.
    - (Optional) [Securely Format a Drive](#securely-format-a-drive) - Once per encrypted device.
4. [Create Normal Filesystem](#create-normal-filesystem) - Once per encrypted device.
5. [Mount It](#mount-it-1) - Once per encrypted device.

> For an enjoyable experience it is important that your system supports
  [hardware accelerated encryption](#test-for-hardware-acceleration) (AES-NI).


### Create a Keyfile
In order to encrypt/decrypt a drive we will first need a password or a key. I
suggest you use a key, since that is more secure, and it can be easily generated
from the following command:

```bash
dd if=/dev/urandom bs=512 count=1 > /path/to/first-drive.key
```

The default max size of a key file is 8 KiB, but there is no point in using a
key which is larger than LUKS's "master key" that is the one being unlocked by
the key file. The maximum master key size for LUKS is 512 bits, but it defaults
to [256 bits][13].

### Encrypt Device/Partition
Now we can encrypt the entire block device

```bash
sudo cryptsetup luksFormat -v -y --key-file=/path/to/first-drive.key /dev/sdX
```

> If you intend to use the entire dirve you will **not** need to create a
  [partition table](#create-partition-table) beforehand.

or we can encrypt an already existing partition

```bash
sudo cryptsetup luksFormat -v -y --key-file=/path/to/first-drive.key /dev/sdX2
```

Both of these operastions are of course destructive, so any existing data on
the drive/partition will be lost.

### Unlock Encrypted Device
You now have to unlock the encrypted device/partition in order to be able to
use it in a way the rest of the computer can understand.

```bash
sudo cryptsetup luksOpen --key-file=/path/to/first-drive.key /dev/sdX first-drive
```

Here you need to change the X to either the device (or the partition) that
you have [just created](#encrypt-devicepartition). The name "`first-drive`", at
the end of the command, needs to be a unique name since this will be used as
sort of an "UUID" of the decrypted device.

> If you are really adamant about security you should proably take a quick look
  at the [Securely Format a Drive](#securely-format-a-drive) section now.

### Create Normal Filesystem
After you have unlocked the encrypted part you may take a look at the
[Create Filesystem](#create-filesystem) section under the
[Create Normal Drive](#create-normal-drive) chapter again, since right now the
mapped point will behave just like a normal device partition. Otherwise just
use the following command to create a standard filesystem on it:

```bash
sudo mkfs.ext4 /dev/mapper/first-drive
```

### Mount It
To test if you can use the newly created (and encrypted) filesystem you may
now mount it.

```bash
sudo mount /dev/mapper/first-drive /mnt/first-drive
```

The importance of the unique drive name is because `crypttab` will first mount
a "encrypted filesystem" to a point in `/dev/mapper/`, which then `fstab` will
use to mount to the "correct" location later. So something like this is what is
happening:


```
                           crypttab
            UUID     --------> | ----> /dev/mapper/first-drive
     (encrypted device)        |      (un-encrypted identifier)
```
```
                             fstab
 /dev/mapper/first-drive  ---> | --->  /mnt/first-drive
(un-encrypted identifier)      |     (desired mount point)
```

Try going into the desired mount point and create a file, just to be certain
everything works as intended.



# Additional Info

This section contains additional information that may be good to know even when
you are not in the process of creating normal/encrypted drives.


## Obtain the UUID
In order to obtain the UUID ([universally unique identifier][16]) for the drives
connected to your system you can simply use the following command, which will
list all available/mountable devices:

```bash
sudo blkid
```

In my case it looked something like this.

```
/dev/sda1: UUID="88f0236b-620a-456e-a4a6-1b5c84996b5c" TYPE="ext4"
/dev/sda5: UUID="b8d210eb-d186-405f-bc42-a55cbe069a27" TYPE="swap"
```

However, if the drive you are looking for doesn't have any (mountable)
partitions on it you will first need to create them manually. A detailed guide
for how to produce a fully functional "normal" drive may be found in the
[Create Normal Drive](#create-normal-drive) section of the
[Preparation Steps](#preparation-steps). Afterwards you should be able
to find the newly created partition by running the command above again.

The same goes for an encrypted partition. After having completed the
[Encrypt Device/Partition](#encrypt-devicepartition) step, in the
[Create Encrypted Drive](#create-encrypted-drive) guide, you should be able to
find a UUID associated with the newly formed encrypted device.


## Test for Hardware Acceleration
If you intend to have encrypted drives attached to your system, you should
probably make sure that your CPU has support for hardware acceleration
([AES-NI][15]) and that [it is enabled][12]. To quickly check if everything is
working as expected you can run the following command:

```bash
openssl speed -evp aes-256-cbc
```

If the numbers returned from that test is lower than 500 MB/s something is
either wrongly configured, or you don't have support for hardware acceleration.
In that case I would strongly suggest you get a more modern CPU in order to
get a good experience with encrypted drives.


## Securely Format a Drive
If you have a new drive, or are re-purposing an old un-encrypted drive, you may
want to format it in such a way that the entire drive is filled with random
data. This is an important step if you are trying to achieve
"[plausible deniability][14]" for you encrypted drives, but should probably be
done anyways since it will erase any traces of old data that was there before.

The following command expects you to have completed the
[Create Encrypted Drive](#create-encrypted-drive) guide until the point where
this section is mentioned in the
[Unlock Encrypted Device](#unlock-encrypted-device) sub-section. Because
when the drive/partition has been unlocked, and it exists under `dev/mapper/`,
you should run the following command to fill it with random data.

```bash
sudo dd if=/dev/zero of=/dev/mapper/first-drive bs=4M status=progress
```

Even if the input stream of the `dd` command is just zeroes, the encryption
algorithm will produce encrypted (i.e. it looks random) data that is written to
the drive. It is also possible to achieve the same result with the following
command (notice we target the device and not the decrypted mount point now):

```bash
sudo dd if=/dev/urandom of=/dev/sdX bs=4M status=progress
```

However, this is usually slower than just spewing zeroes through the hardware
accelerated encryption algorithm.

If you are trying to create an encrypted drive you should now start that guide
over again from the [beginning](#create-encrypted-drive), so you get a
completely new encryption key afterwards. This is a paranoid security step
taken so there is absolutely no connection between the "encrypted zeroes" and
your real data.



## Ansible Variables Location
I prefer to add all the necessary variables, used by this role, in the
`host_vars/<hostname>` file, since these variables are usually unique for each
host. This way it is easy to separate the information on a host by host basis,
without creating any conflicts between them.

An important thing to remember is that Ansible will overwrite, and not merge,
hashes/dictionaries (like `drives_boot` or `drives_normal`) if there are two
with the same name. You can therefore not have a part of them be defined in the
`group_vars/` and then other parts in the `host_vars/`. If you do not like this
behavior you may look into setting [`hash_behaviour = merge`][2], but be aware
that this is not a very [good solution][3]. Instead you should probably look
into the [`combine`][5] filter or the [`merge_vars`][4] action plugin.



[1]: https://github.com/trapexit/mergerfs
[2]: https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-hash-behaviour
[3]: https://medium.com/uptime-99/3-things-ive-learned-about-ansible-the-hard-way-bae341524a86
[4]: https://pypi.org/project/ansible-merge-vars/
[5]: https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#combining-hashes-dictionaries
[6]: https://en.wikipedia.org/wiki/Master_boot_record
[7]: https://en.wikipedia.org/wiki/GUID_Partition_Table
[8]: https://rainbow.chard.org/2013/01/30/how-to-align-partitions-for-best-performance-using-parted/
[9]: https://man7.org/linux/man-pages/man8/parted.8.html
[10]: https://linux.die.net/man/8/cryptsetup
[11]: https://wiki.archlinux.org/index.php/Parted#Partition_schemes
[12]: https://www.cyberciti.biz/faq/how-to-find-out-aes-ni-advanced-encryption-enabled-on-linux-system/
[13]: https://askubuntu.com/a/653986
[14]: https://blog.linuxbrujo.net/posts/plausible-deniability-with-luks/
[15]: https://en.wikipedia.org/wiki/AES_instruction_set
[16]: https://en.wikipedia.org/wiki/Universally_unique_identifier
[17]: https://github.com/trapexit/mergerfs/releases/
[18]: https://github.com/trapexit/mergerfs/#options
[19]: https://www.howtogeek.com/465350/everything-you-ever-wanted-to-know-about-inodes-on-linux/
[20]: https://unix.stackexchange.com/a/7965
