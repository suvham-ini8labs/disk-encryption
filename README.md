# Linux Disk Encryption using LUKS (Linux Unified Key Setup)

### Installation 

Install `cryptsetup` package

```sh
apt install cryptsetup
```

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

### Backup and Restore

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