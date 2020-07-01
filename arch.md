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