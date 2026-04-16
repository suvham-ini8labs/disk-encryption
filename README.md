# Linux Disk Encryption using LUKS (Linux Unified Key Setup)

## Encrypting with existing data 

First we will `cryptsetup` package

```sh
apt install cryptsetup
```

We will add data to our disk before encrypting

```sh
mkdir test_dir
touch test_dir/test_data
echo "Hi" > test_dir/test_data
```

Unmount the device
```sh
umount /mnt/nvme0n2
```

Check the filesystem for any errors.

```sh
fsck -n /dev/nvme0n2
```

Output

```sh
fsck from util-linux 2.39.3
e2fsck 1.47.0 (5-Feb-2023)
/dev/nvme0n2: clean, 17/1966080 files, 167459/7864320 blocks
```

Generate a key file

```sh
dd if=/dev/urandom bs=512 count=1 of=disk.key
chmod 400 disk.key
```
Run encryption in place using the key file.

```sh
cryptsetup reencrypt \
  --encrypt \
  --reduce-device-size 32M \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --key-file disk.key \
  --pbkdf argon2id \
  /dev/nvme0n2
```

Output

```sh
WARNING!
========
This will overwrite data on LUKS2-temp-eb9451f6-d2b7-4030-a8e6-dc44aadbb25f.new irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
Warning: keyslot operation could fail as it requires more than available memory.
Finished, time 03m34s,   29 GiB written, speed 143.3 MiB/s
```

Data on the last <size> sectors will be lost since data will be shifted to make space for the luks header.
It is reccomended to add more space / have no data on the last sectors. 

```sh
 --reduce-device-size size
    This means that the last size sectors on the original device
    will be lost, and data will be effectively shifted by the
    specified number of sectors.

    It could be useful if you added some space to the underlying
    partition or logical volume (so the last size sectors contains
    no data).

	LUKS2: Initialize LUKS2 reencryption with data device size
        reduction (currently, only encryption mode is supported). The
        last size sectors on the original plaintext device is used for
        temporarily storing the original first data segment. The
        former first data segment is replaced with LUKS2 header (half
        the size value), and plaintext data is shifted backwards
        (again half the size value) while being encrypted.``
```

We will also use a passphrase as our second way to access the disk.
We can enter the passphrase on the prompt.
```sh
cryptsetup luksAddKey --key-file disk.key /dev/<block-device-name>
```

This command will create a new device mapper, which is a virtual block device on which we can create a filesystem.
If we write on this mapper, kernel will encrypt and store it in the physical block device on the fly and similar for read operations.

```sh
cryptsetup luksOpen --key-file disk.key /dev/nvme0n2 nvme0n2-mapper
```

Now if we run `fsck` command it will throw an error, since cryptsetup created a luks header but the fs is not aware of it

```sh
e2fsck -f /dev/mapper/nvme0n2-mapper
```

Output

```sh
e2fsck 1.47.0 (5-Feb-2023)
The filesystem size (according to the superblock) is 7864320 blocks
The physical size of the device is 7860224 blocks
Either the superblock or the partition table is likely to be corrupt!
Abort<y>? no
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/mapper/nvme0n2-mapper: 17/1966080 files (0.0% non-contiguous), 167459/7864320 blocks
````

We need to shrink the filesystem using `resize2fs` command.

```sh
resize2fs /dev/mapper/nvme0n2-mapper 
```

Output

```sh
resize2fs 1.47.0 (5-Feb-2023)
Resizing the filesystem on /dev/mapper/nvme0n2-mapper to 7860224 (4k) blocks.
The filesystem on /dev/mapper/nvme0n2-mapper is now 7860224 (4k) blocks long.
```

Now we can mount our mapper device

```sh
mount /dev/mapper/nvme0n2-mapper /mnt/nvme0n2
```

Access the filesystem

```sh
root@suvham-jumpserver:/mnt/data# ls
total 28
drwx------ 2 root root 16384 Apr 14 13:36 lost+found
drwxr-xr-x 2 root root  4096 Apr 15 06:27 test_dir1
drwxr-xr-x 2 root root  4096 Apr 15 06:27 test_dir2
drwxr-xr-x 2 root root  4096 Apr 15 06:27 test_dir3
root@suvham-jumpserver:/mnt/data# cat test_dir1/test_data 
Hi from device
```

We will now make changes persistent

We will now enable our encrypted disk mounting at boot time with the key.
We will add this line in `/etc/crypttab` file

```sh
# <target name>	<source device>		<key file>	<options>
nvme0n2_mapper /dev/nvme0n2 /home/azureuser/.luks/disk.key luks
```

We will add this line in `/etc/fstab` file to mount the device.

```sh
/dev/mapper/nvme0n2_mapper /mnt/nvme0n2 ext4 defaults 0 2
```

We can verify our luks device

```sh
cryptsetup luksDump /dev/nvme0n2
```

Output

```sh
LUKS header information
Version:       	2
Epoch:         	3842
Metadata area: 	16384 [bytes]
Keyslots area: 	16744448 [bytes]
UUID:          	eb9451f6-d2b7-4030-a8e6-dc44aadbb25f
Label:         	(no label)
Subsystem:     	(no subsystem)
Flags:       	(no flags)

Data segments:
  0: crypt
	offset: 16777216 [bytes]
	length: (whole device)
	cipher: aes-xts-plain64
	sector: 512 [bytes]

Keyslots:
  0: luks2
	Key:        512 bits
	Priority:   normal
	Cipher:     aes-xts-plain64
	Cipher key: 512 bits
	PBKDF:      argon2id
	Time cost:  17
	Memory:     607898
	Threads:    4
	Salt:       03 21 8f ff b1 c0 28 11 9b 12 3b e2 3e 7a ed ce 
	            93 08 e5 98 f4 7e 1c a7 d7 d2 27 62 82 84 d9 c5 
	AF stripes: 4000
	AF hash:    sha256
	Area offset:32768 [bytes]
	Area length:258048 [bytes]
	Digest ID:  0
  1: luks2
	Key:        512 bits
	Priority:   normal
	Cipher:     aes-xts-plain64
	Cipher key: 512 bits
	PBKDF:      argon2id
	Time cost:  17
	Memory:     616832
	Threads:    4
	Salt:       50 e1 f5 01 e8 8d 3c 84 e4 6b d3 1c 8f 1b 75 91 
	            b8 6d da 08 28 54 a4 3e f6 2b dd be 34 2e a0 04 
	AF stripes: 4000
	AF hash:    sha256
	Area offset:290816 [bytes]
	Area length:258048 [bytes]
	Digest ID:  0
Tokens:
Digests:
  0: pbkdf2
	Hash:       sha256
	Iterations: 289023
	Salt:       62 f3 c3 3a a7 a4 04 87 a6 33 3a e3 97 73 a1 b9 
	            f8 8f fa 6f a2 a8 80 78 ff e8 c2 55 42 7b 21 5a 
	Digest:     2b da 2d 77 fd 65 de 9f ac 27 40 f9 d5 22 a0 56 
	            43 ab de 04 f9 e8 58 ad b9 b0 ee 10 a9 fe 88 df 
```

## Backup and Restore

We will now create a backup of the luks header as it contains important information to encrypt and decrypt the disk.
The luks header is located at the very begining of the disk, any tampering to the header can make the disk unaccessible
even if provided with correct key.

```sh
cryptsetup luksHeaderBackup /dev/DEVICE --header-backup-file /path/to/backupfile
```

Store the backup file somewhere remote(not on the same disk).

We can now restore the luks header.

```sh
cryptsetup luksHeaderRestore /dev/DEVICE --header-backup-file /path/to/backup_header_file
```

## Undo operation

Unmount the device

```sh
umount /data
```

We can now remove the device mapper

```sh
cryptsetup luksClose data_crypt
```

We can now decrypt our disk
```sh
cryptsetup reencrypt --decrypt --resilience checksum --header nvme0n2-decryption-header /dev/nvme0n2
```

We can check if the device is luks encrypted

```sh
cryptsetup isLuks /dev/nvme0n2 && echo "encrypted" || echo "not encrypted"
```

We can check if the filesystem is corrupted.

```sh
e2fsck -f /dev/nvme0n2
```
Grow the filesystem on the disk to maximum size(consume the empty header blocks).
```sh
resize2fs /dev/nvme0n2
```

```sh
resize2fs 1.47.0 (5-Feb-2023)
Resizing the filesystem on /dev/nvme0n2 to 7864320 (4k) blocks.
The filesystem on /dev/nvme0n2 is now 7864320 (4k) blocks long.
```

Check contents of the device

```sh
mount /dev/sdb1 /data
ls /data
```

```sh
root@suvham-jumpserver:/mnt# df -h nvme0n2/
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n2     30G   24K   28G   1% /mnt/nvme0n2
```

### Encrypt empty disk 

We now will create a key for the encryption.This key is not used to encrypt the data on the disk.
It is used to encrypt the master key, which is then used to encrypt the data.

```sh
dd if=/dev/urandom bs=512 count=1 of=disk.key
chmod 400 disk.key
```

We will now encrypt the disk using the key we created earlier.

```sh
cryptsetup luksFormat \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --hash sha256 \
  --pbkdf argon2id \
  --key-file disk.key \
  /dev/<block-device-name>
```

We will also use a passphrase as our second way to access the disk.

```sh
cryptsetup luksAddKey --key-file disk.key /dev/<block-device-name>
```

This command will create a new device mapper, which is a virtual block device on which we can create a filesystem.
If we write on this mapper, kernel will encrypt and store it in the physical block device on the fly and similar for read operations.

```sh
cryptsetup luksOpen --key-file disk.key /dev/nvme0n2 nvme0n2_mapper
```

Create a filesystem

```sh
mkfs.ext4 /dev/mapper/nvme0n2_mapper
```

We will now mount the mapper

```sh
mkdir -p /mnt/nvme0n2
mount /dev/mapper/nvme0n2 /mnt/nvme0n2
```

Output:

```sh
nvme0n2          259:5    0   30G  0 disk  
└─nvme0n2_mapper 252:0    0   30G  0 crypt /mnt/nvme0n2
```

We will now make changes persistent

We will now enable our encrypted disk mounting at boot time with the key.
We will add this line in `/etc/crypttab` file

```sh
# <target name>	<source device>		<key file>	<options>
nvme0n2_mapper /dev/nvme0n2 /home/azureuser/.luks/disk.key luks
```

We will add this line in `/etc/fstab` file to mount the device.

```sh
/dev/mapper/nvme0n2_mapper /mnt/nvme0n2 ext4 defaults 0 2
```

We can now verify disk is encrypted

We can use `luksDump` to get information about the luks header.

```sh
cryptsetup luksDump /dev/nvme0n2
```

Output: 

```sh
LUKS header information
Version:       	2
Epoch:         	4
Metadata area: 	16384 [bytes]
Keyslots area: 	16744448 [bytes]
UUID:          	2cd85cb4-dda6-498e-a4eb-c17369eb8b25
Label:         	(no label)
Subsystem:     	(no subsystem)
Flags:       	(no flags)

Data segments:
  0: crypt
	offset: 33554432 [bytes]
	length: (whole device)
	cipher: aes-xts-plain64
	sector: 4096 [bytes]

Keyslots:
  0: luks2
	Key:        512 bits
	Priority:   normal
	Cipher:     aes-xts-plain64
	Cipher key: 512 bits
	PBKDF:      argon2id
	Time cost:  15
	Memory:     678602
	Threads:    4
	Salt:       cd d3 a1 7b 8f b2 63 55 9b 57 e0 d6 a7 b9 1e 38 
	            ed 55 b3 74 6e 3c aa bc 48 5d d8 49 f5 56 00 04 
	AF stripes: 4000
	AF hash:    sha256
	Area offset:32768 [bytes]
	Area length:258048 [bytes]
	Digest ID:  0
  1: luks2
	Key:        512 bits
	Priority:   normal
	Cipher:     aes-xts-plain64
	Cipher key: 512 bits
	PBKDF:      argon2id
	Time cost:  15
	Memory:     681622
	Threads:    4
	Salt:       a4 fa a5 36 33 d1 b2 fa ea 8c 69 b7 1c 53 bf 09 
	            2c 71 ff 0a df ed 23 b6 8b 34 58 03 41 2a 79 ea 
	AF stripes: 4000
	AF hash:    sha256
	Area offset:290816 [bytes]
	Area length:258048 [bytes]
	Digest ID:  0
Tokens:
Digests:
  0: pbkdf2
	Hash:       sha256
	Iterations: 289982
	Salt:       12 21 b1 b4 68 d1 d2 fe 02 f9 a7 69 fc 4d 01 5e 
	            c7 00 27 61 c5 35 ef 6a 1f 27 53 dc ff 62 23 4e 
	Digest:     99 c7 41 b4 06 65 85 29 3d bc fe 6a cc 49 f0 88 
	            f9 ce a8 e1 e3 cd 7f a1 c7 63 e1 aa 60 46 40 46 
```

```sh
cryptsetup status nvme0n2_mapper
```

Output:

```sh
/dev/mapper/nvme0n2_mapper is active and is in use.
  type:    LUKS2
  cipher:  aes-xts-plain64
  keysize: 512 bits
  key location: keyring
  device:  /dev/nvme0n2
  sector size:  4096
  offset:  65536 sectors
  size:    62849024 sectors
  mode:    read/write
```


## Resources

- [Encrypting existing data on a block device using LUKS2 - Red Hat](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/security_hardening/encrypting-block-devices-using-luks_security-hardening#encrypting-existing-data-on-a-block-device-using-luks2_encrypting-block-devices-using-luks)