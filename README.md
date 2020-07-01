*Gidon Rémi*  
*UBS - Master CSSE 2019/2020*

*Sécurité avancée des systèmes d'exploitation*  
*Cas d'une console de jeux*  
*Tutoriel mise en place du système*

# Installation

1. Download image
```bash
wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-10.3.0-amd64-netinst.iso
```

2. Flash image
```bash
# Plug USB pendrive
# check it's device with dmesg
# and replace /dev/sdX by the device found
dd if=debian-10.3.0-amd64-netinst.iso of=/dev/sdX bs=1M status=progress
```

3. Connect the Up-Squared to a screen with HDMI or your laptop using UART
4. Plug USB pendrive in Up-Squared and boot it
5. Start `expert graphical installation`
6. User: `ossec`, password: `ossec`
7. Partitions:
    - boot vfat ESP partition 500MB, boot flag
    - rootfs ext4 partition 10GB, mounted on /
    - swap partition 5GB
    - let left space available (~15GB) for later configuration
8. Install only targeted drivers & modules

# Bootloader

We will create our own chain of trust from the `.efi` loader to the kernel. In order to achieve such chain of trust we will have to sign each component (efi loader, grub, initrd and the linux kernel).  

## Update EFI keystore

1. Create a work environment, mine is in my home
```bash
mkdir /home/ossec/keys
cd ~/keys
```

2. First thing first, we need keys to sign or encrypt anything. Replace `CN` (Name) and `O` (Organization) with the ones of your choice.
```bash
CN=ossec
O=upboard
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=$CN PK, O=$O/"  -keyout PK.key  -out PK.crt  -days 3650 -nodes -sha256
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=$CN KEK, O=$O/" -keyout KEK.key -out KEK.crt -days 3650 -nodes -sha256
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=$CN db, O=$O/"  -keyout db.key  -out db.crt  -days 3650 -nodes -sha256
```

3. From these `.key` and `.crt` files we will create the `.esl` and `.auth` files required by UEFI.
```bash
GUID=$(uuidgen --random)
echo $GUID > GUID.txt
cert-to-efi-sig-list -g $GUID PK.crt PK.esl
sign-efi-sig-list -g $GUID -k PK.key -c PK.crt PK PK.esl PK.auth
cert-to-efi-sig-list -g $GUID KEK.crt KEK.esl
sign-efi-sig-list -g $GUID -a -k PK.key -c PK.crt KEK KEK.esl KEK.auth
cert-to-efi-sig-list -g $GUID db.crt db.esl
sign-efi-sig-list -g $GUID -a -k KEK.key -c KEK.crt db db.esl db.auth
```

4. Then copy those keys to the ESP, where it's reachable from the BIOS, to update the keystore.
```bash
# copy keys in ESP to be able to reach them from BIOS
# we will create our own boot entry in /ossec
mkdir -p /boot/efi/EFI/ossec/keys
cp *.auth /boot/efi/EFI/ossec/keys
```

5. Backup actual EFI keys
In order to be able to restore the system in case we brick it during developement
```bash
efi-readvar -v PK -o old_PK.esl
efi-readvar -v KEK -o old_KEK.esl
efi-readvar -v db -o old_db.esl
efi-readvar -v dbx -o old_dbx.esl
```

6. Update keystore in the BIOS
- `reboot now`
- During boot, press ESC to enter BIOS.
- When prompt asks for password enter `upassw0rd`
- Navigate to `Boot>Secure boot>Keys`
- Update `PK`, `KEK` and `db` with their `.auth` counterparts
- Activate secure boot (`Secure boot: [Enabled]`)
- Save & Exit

We have create our **root of trust** ! That's the first link of the chain of trust that holds the role of verifying the first component (which will be our `.efi` loader).

## Grub

The next link is our bootloader. For the sake of simplicity we will use Grub2, which is by default with the distribution and supports secure boot.  
We won't only enable secure boot by verying `initrd` and `vmlinuz` with grub, we will also harden it with a password and protect our boot configuration against modifications.

Create an environment for grub configuration.
```bash
mkdir ~/build/grub
cd ~/build/grub
```

Grub doesn't support `.esl` and `.auth` files previously generated for the UEFI, only GPG keys, so let's [create our own GPG key](https://help.github.com/en/github/authenticating-to-github/generating-a-new-gpg-key).
```bash
apt-get install gpg
# RSA key type, 4096 key bits
gpg --full-generate-key
gpg --list-secret-keys --keyid-format LONG
# should see your newly created key
GPGID=#TODO add your GPG key ID, the one marked as ultimately trusted
```

### Password

As previously explained we'll add a password to protect our boot entry.
```bash
GRUB_USER="ossec" #TODO change with yours
GRUB_PASSWORD="ossec" #TODO change with yours
GRUB_PASSWORD_HASH=$(echo -e "$GRUB_PASSWORD\n$GRUB_PASSWORD" | grub-mkpasswd-pbkdf2 | grep -o "grub.*")
```

### Init configuration

We will build grub in `standalone` mode which means that all files will reside under the ESP partition.  
For grub to be able to load those files, we need a first config file (`grub.init.cfg`) located into the efi loader `.efi` that will discover the partition where the needed configuration is stored.  

1. To locate the partition, we will use it's unique id (`fsuuid`).
```bash
# list the partitions
lsblk -f
# copy the efi fsuuid
EFI_UUID=$(lsblk -f | grep -i efi | grep -E -o "[A-Z0-9]{4}-[A-Z0-9]{4}")
```

2. Create `grub.init.cfg` as follow
```bash
# Enforce that all loaded files must have a valid signature.
set check_signatures=enforce
export check_signatures
# Password protection
set superusers="$GRUB_USER"
export superusers
password_pbkdf2 $GRUB_USER $GRUB_PASSWORD_HASH
# Locate configuration
search --no-floppy --fs-uuid --set=root $EFI_UUID
configfile /grub.cfg
# Without this we provide the attacker with a rescue shell if he just presses
# <return> twice.
echo "grub.init.cfg did not boot the system, rebooting the system in 10 seconds."
sleep 10
reboot
```

### Standalone build

1. Sign the first config file (`grub.init.cfg`)
```bash
gpg --yes --default-key $GPGID --detach-sign grub.init.cfg
```

2. Build grub in standalone mode
```bash
# signature key used
gpg --export $GPGID > ../keys/gpg.key
# build grub
grub-mkstandalone \
    -d /usr/lib/grub/x86_64-efi \
    -O x86_64-efi \
    --modules "part_gpt fat ext2 configfile verify gcry_sha512 gcry_rsa password_pbkdf2 echo normal linux linuxefi all_video search search_fs_uuid reboot sleep" \
    --pubkey ../keys/gpg.key \
    --output grubx64.efi \
    "boot/grub/grub.cfg=grub.init.cfg" \
    "boot/grub/grub.cfg.sig=grub.init.cfg.sig" \
    -v
```

3. Sign grub `.efi` loader with UEFI keys and place it under our ESP boot entry
```bash
sbsign --key ../keys/db.key --cert ../keys/db.crt grubx64.efi
# copy our efi loader into ESP
cp grubx64.efi.signed /boot/efi/EFI/ossec/grubx64.efi
```

### Config file

Now that we have our `grubx64.efi` loader ready, let's configure it to load the next component which is the `kernel` and the `initrd` to boot.

1. As soon as it's loaded, our kernel will search a `rootfs` to mount. We will give the `fsuuid` of our `rootfs` partition where `ubuntu` already is installed.
```bash
# find the rootfs fsuuid, assuming it's the first ext4 partition on the system
ROOT_UUID=$(lsblk -f | grep -i ext4 | tr -s ' ' | cut -d' ' -f3)
```

2. Create `grub.cfg`
```bash
#video mode settings
set timeout_style=menu
set timeout=2
set gfxmode=auto
set gfxpayload=keep
terminal_output gfxterm
#--unrestricted to allow any user to boot but only root to modify
menuentry 'ossec with Linux' --unrestricted {
    echo   'Loading ossec Linux ...'
    linux  /vmlinuz root=UUID=$ROOT_UUID ro
    echo   'Loading ossec initial ramdisk ...'
    initrd /initrd.img
}
```

<!--
## Kernel build

1. Download sources
```bash
mkdir ~/build/kernel/
cd ~/build/kernel/
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.4.29.tar.xz
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.4.29.tar.sign
unzx linux-5.4.29.tar.xz
gpg --locate-keys torvalds@kernel.org gregkh@kernel.org
gpg --verify linux-5.4.29.tar.sign
tar xvf linux-5.4.29.tar
```

2. We could harden the kernel with recommandations from [KSP](https://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project/Recommended_Settings#x86_64) or [CLIP-OS](https://docs.clip-os.org/clipos/kernel.html#configuration) and check it against [config hardenened check](https://github.com/a13xp0p0v/kconfig-hardened-check).
```bash
# copy current configuration (change kernel version in the header)
cp -v /boot/config-$(uname -r) hardened.config
git clone https://github.com/a13xp0p0v/kconfig-hardened-check.git
./kconfig-hardened-check/bin/kconfig-hardened-check.py -c ./linux-5.4.27/hardened.config
# don't forget to generate the .config
make menuconfig
# load hardened.config and generate
```

3. Build the kernel
```bash
cd linux-5.4.27
make -j $(nproc)
make modules_install
make install
```
-->

### Sign for Grub

```bash
cp /boot/vmlinuz-$(uname -r) /boot/efi/vmlinuz
cp /boot/initrd.img-$(uname -r) /boot/efi/initrd.img
cp grub.cfg /boot/efi

gpg --default-key $GPGID --detach-sign /boot/efi/grub.cfg
gpg --default-key $GPGID --detach-sign /boot/efi/vmlinuz
gpg --default-key $GPGID --detach-sign /boot/efi/initrd.img
```

## Boot entry

1. Create a boot entry (`ossec` here) in the ESP and update boot order to put it first.
```bash
efibootmgr -c -d /dev/mmcblk0 -L "ossec" -l '\EFI\ossec\grubx64.efi'
# change boot order to ossec only
efibootmgr -o 0001
```

2. Update boot priority in BIOS
- `reboot now`
- During boot, press ESC to enter BIOS.
- When prompt asks for password let it empty and hit `Enter`
- Navigate to `Boot>UEFI Hard Disk Drive BSS Properties`
- Set our `ossec` boot entry first (a.k.a `Boot Option #1 [ossec]`)
- Save & Exit

1. You should see a grub menu with our `ossec` entry, select it. The following should appear on the screen before loading the `rootfs`:
```raw
Loading ossec Linux...
Loading ossec initial ramdisk ...
```

# Partitions

We already have the ESP partition, swap partition and rootFS created during installation procedure.  
We only miss particular partitions for the need of our system (proprietary gaming station).

`/mnt/data` will contain user and system configuration, system and applications logs but also secrets used to verify applications.  
That will be the only block-level encrypted partition on the system. We will use `dm-crypt` and `LUKS` to enable the encryption/decryption automatically.  
The dilema here is to create a root of trust to store key for this partition, as the Up-Squared supports TPM integration we could setup a TPM to retrieve key before mounting this partition (this is not done in this tutorial because out of scope).  

The second additional partition that our system will need is `/mnt/apps`, it will contains all the games encrypted and signed, ready to be launched.  
Every game (represented as a folder) is encrypted and every component loaded into the system (executables) are signed.  The partition is mounted as `read-only` and `no-user` to avoid runtime modifications.  

Our end goal will be:
```bash
lsblk -f
NAME        FSTYPE  LABEL   UUID    AVAIL   MOUNTPOINT
mmcblk0
└─mmcblk0p1 vfat    ESP     XXX     500M    /boot/efi
└─mmcblk0p2 ext4    ROOTFS  XXX     10G     /
└─mmcblk0p1 swapfs  SWAP    XXX     5G      [SWAP]
└─mmcblk0p1 luks            XXX
│ └─data    ext4    DATA    XXX     5G      /mnt/data
└─mmcblk0p1 ext4    APPS    XXX     10G     /mnt/apps
```

1. Create the partition scheme needed
```bash
# data partition
parted -s /dev/mmcblk0 mkpart ext4 15GB 20GB
# apps partition
parted -s /dev/mmcblk0 mkpart ext4 20GB 30GB
```

2. Format the partitions according to our needs
```bash
mkfs.ext4 -L APPS /dev/mmcblk0p5
mkdir /mnt/apps
mount /dev/mmcblk0p5 /mnt/apps
```

## System

dm-verity ?

## Data

1. Create the key for the `data` partition
```bash
mkdir /etc/ossec
dd if=/dev/urandom of=/etc/ossec/data.key bs=512 count=8
cryptsetup --verbose luksFormat /dev/mmcblk0p4 /etc/ossec/data.key
```

2. Setup the encrypted partition with `dm-crypt`
```bash
cryptsetup -v --key-file /etc/ossec/data.key luksOpen /dev/mmcblk0p4 data
ls -l /dev/mapper/data
mkfs.ext4 -L DATA /dev/mapper/data
mkdir /mnt/data
mount /dev/mapper/data /mnt/data
```

3. Move `/var/log` to `/mnt/data/log`
```bash
rsync -aqxP /var/log /mnt/data
rm -rf /var/log
ln -s /var/log /mnt/data/log
```

4. Setup DAC
```bash
# dashboard user for console user interactions
adduser --no-create-home dashboard
mkdir /mnt/data/{secret/{apps,efi},dashboard,log}
cp -r /home/ossec/keys /mnt/data/secret

chown -R ossec /mnt/data/secret
chgrp -R ossec /mnt/data/secret
chown -R dashboard /mnt/data/dashboard
chgrp -R ossec /mnt/data/dashboard

chmod 700 /mnt/data/secret
chmod -R 600 /mnt/data/secret/*.key

chmod 750 /mnt/data/dashboard
```

## Apps

1. Setup DAC
```bash
chown -R ossec /mnt/apps
chgrp -R ossec /mnt/apps
chmod -R 700 /mnt/data/secret
mkdir /mnt/apps/app1
``` 

2. Applications encryption
Using `fs-crypt` we can encrypt a folder and still operate on it transparently.  
We could use other manual solutions that does not operate transparently.
```bash
tune2fs -O encrypt /dev/mmcblk0p5
fscrypt setup
fscrypt setup /mnt/apps

dd if=/dev/urandom of=/mnt/data/secret/apps/app1.key bs=1 count=32
# use raw key app1.key
fscrypt encrypt app1
echo "# App1 " > app1/app1.md
```

3. Application authentication
Using `fs-verity` we can authenticate binaries of an application transparently.  
Otherwise we could use `gpg` as before to sign and verify files manually.  
We disabled the `ro` option from `/etc/fstab` for `/mnt/apps` because `fs-verity` does not operate on read-only filesystems.
```bash
cd ~
sudo apt-get install libssl-dev make gcc
git clone https://git.kernel.org/pub/scm/linux/kernel/git/ebiggers/fsverity-utils.git
cd fsverity-utils
make
make install
```

We need to add key signatures to fs-verity keyring
```bash
cd /mnt/data/secret/app1
openssl req -newkey rsa:4096 -subj "/CN=$CN, O=$O/" -nodes -keyout app1.sign.key -x509 -out app1.crt
openssl x509 -in app1.crt -out app1.der -outform der
keyctl padd asymmetric '' %keyring:.fs-verity < app1.der
# verify all files
sysctl fs.verity.require_signatures=1
```
Then format our partition so it supports `verity` (also check kernel configuration for `CONFIG_FS_VERITY=m` and `CONFIG_FS_VERITY_BUILTIN_SIGNATURES=1`) and finally sign our file.
```bash
tune2fs -O verity /dev/mmcblk0p5
cd /mnt/apps/app1
fsverity sign app1.md app1.md.sig --key=/mnt/data/secret/apps/app1.sign.key --cert=/mnt/data/secret/apps/app1.crt
fsverity enable app1.md --signature=app1.md.sig
```

## Automatic mounts

To avoid opening LUKS partion and remounting `data` and `apps` every boot, we can rely on `/etc/fstab` and `/etc/crypttab`.

Modify `/etc/crypttab` to add `data`
```
data    /dev/mmcblk0p4      /etc/ossec/data.key       luks
```

Modify `/etc/fstab` to add `apps` and `data`
```bash
# debian installation partitioning
/dev/mmcblk0p2      /           ext4    errors=remount-ro                   0   1
/dev/mmcblk0p1      /boot/efi   vfat    umask=0077                          0   1
/dev/mmcblk0p3      none        swap    sw                                  0   0
# equivalent as above on Debian systems
# cp /usr/share/systemd/tmp.mount /etc/systemd/system/ && systemctl enable tmp.mount
tmpfs               /tmp        tmpfs   mode=1777,nosuid,nodev              0   0
# luks partition
/dev/mapper/data    /mnt/data   ext4    rw,nosuid,noexec,nodev,nouser,auto  0   2
# apps partition
/dev/mmcblk0p5      /mnt/apps   ext4    rw,nodev,nouser,auto                0   2
```

# Production setup

Clean our workspace

```bash
# remove keys used to configure secure boot
rm -rf /boot/efi/EFI/ossec/keys
rm -rf /home/ossec/keys

# delete debian boot entry
efibootmgr -b 3 -B
rm -rf /boot/efi/debian
rm -rf /boot/grub
```

# Filesystem architecture

```
/
├── boot
│   └── efi 700 root
│       ├── EFI
│       │   └── grubx64.efi signed
│       ├── grub.cfg
│       ├── grub.cfg.sig
│       ├── initrd.img
│       ├── initrd.img.sig
│       ├── vmlinuz
│       └── vmlinuz.sig
├── mnt
│   ├── data 700 ossec crypted
│   │   ├── secret
│   │   │   ├── apps
│   │   │   │   ├── app1.key
│   │   │   │   └── app1.crt
│   │   │   ├── efi
│   │   │   │   ├── PK.key
│   │   │   │   ├── KEK.key
│   │   │   │   └── db.key
│   │   │   └── vmlinuz.sig
│   │   ├── dashboard 740 dashboard
│   │   └── log 744 root
│   └── apps 700 ossec
│       └── app1 crypted
│           ├── app1.bin
│           └── app1.sig
├── var
│   └── log -> /mnt/data/log 777 root
├── dev
│   └── mmcblk0
│       ├── mmcblk0p1   vfat    ESP     500M    /boot/efi
│       ├── mmcblk0p2   ext4    ROOTFS  10G     /
│       ├── mmcblk0p1   swapfs  SWAP    5G      [SWAP]
│       ├── mmcblk0p1   luks
│       │   └── data    ext4    DATA    5G      /mnt/data
│       └── mmcblk0p1   ext4    APPS    10G     /mnt/apps
├── etc
│   └── ossec 700 ossec
│       └── data.key 400 ossec
├── tmp tmpfs
├── home
├── media
├── opt
├── proc
├── root
├── run
├── sys
├── srv
└── usr
```

# Sources

[Secure Boot with GRUB 2 and signed Linux images and initrds](https://ruderich.org/simon/notes/secure-boot-with-grub-and-signed-linux-and-initrd)  
[Linux: Secure Boot](http://www.fit-pc.com/wiki/index.php/Linux:_Secure_Boot)  
[How to boot Linux using UEFI with Secure Boot ?](https://ubs_csse.gitlab.io/secu_os/tutorials/linux_secure_boot.html)  
[Create a Luks encrypted partition on Linux Mint](https://blog.tinned-software.net/create-a-luks-encrypted-partition-on-linux-mint/)  
[Automount a luks encrypted volume on system start](https://blog.tinned-software.net/automount-a-luks-encrypted-volume-on-system-start/)  
[fs-verity](https://github.com/ebiggers/fsverity)  
[fs-crypt](https://github.com/google/fscrypt)