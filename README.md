# Custom iPXE EFI Boot ISO for Ubuntu 24.04

Network boot Ubuntu 24.04 live server over HTTP with static IP support and automatic NIC detection via MAC address (BOOTIF).

## Prerequisites

```bash
apt install liblzma-dev isolinux genisoimage mtools xorriso
```

## Build

### 1. Clone iPXE

```bash
git clone https://github.com/ipxe/ipxe.git
cd ipxe/src
```

### 2. Create `boot.ipxe`

```ipxe
#!ipxe

echo ===== Ubuntu Network Boot =====
echo
echo Detected interfaces:
ifstat
echo
prompt Press any key to continue...

echo -n Interface number (0, 1, 2...): && read nicnum
set nic net${nicnum}

menu Network Configuration
item static    Static IP
item dhcp      DHCP
choose mode && goto ${mode}

:dhcp
dhcp ${nic}
set ip-param ip=dhcp
goto boot

:static
echo -n IP address: && read ip
echo -n Netmask: && read netmask
echo -n Gateway: && read gateway
echo -n DNS: && read dns
ifopen ${nic}
set ${nic}/ip ${ip}
set ${nic}/netmask ${netmask}
set ${nic}/gateway ${gateway}
set ${nic}/dns ${dns}
set ip-param ip=${ip}::${gateway}:${netmask}::::none nameserver=${dns}
goto boot

:boot
imgfree
set base-url http://<YOUR-SERVER-IP>:8000/ubuntu_24.04/amd64
set iso-url http://<YOUR-SERVER-IP>:8000/ubuntu_24.04/ubuntu-24.04-live-server-amd64.iso
kernel ${base-url}/linux initrd=initrd ${ip-param} BOOTIF=01-${${nic}/mac:hexhyp} url=${iso-url} || goto failed
initrd ${base-url}/initrd || goto failed
boot || goto failed

:failed
echo
echo ===== BOOT FAILED =====
echo
prompt Press any key to retry...
goto boot
```

### 3. Build EFI SNP binary

Use `snp.efi` (not `ipxe.efi`) — it uses the UEFI firmware's network stack instead of iPXE native drivers. This avoids freezes on boards with NICs that iPXE doesn't support well (Mellanox ConnectX-4, Intel I226, etc.).

```bash
make bin-x86_64-efi/snp.efi EMBED=boot.ipxe
```

### 4. Package as EFI-bootable ISO

The ISO needs a FAT filesystem image as the EFI boot entry — pointing `-e` directly at the raw `.efi` binary causes UEFI firmware to stall at ~33KB.

```bash
mkdir -p efi_iso/EFI/BOOT
cp bin-x86_64-efi/snp.efi efi_iso/EFI/BOOT/BOOTX64.EFI

dd if=/dev/zero of=efi_iso/efiboot.img bs=1M count=4
mkfs.vfat efi_iso/efiboot.img
mmd -i efi_iso/efiboot.img ::/EFI
mmd -i efi_iso/efiboot.img ::/EFI/BOOT
mcopy -i efi_iso/efiboot.img efi_iso/EFI/BOOT/BOOTX64.EFI ::/EFI/BOOT/BOOTX64.EFI

xorrisofs -o ipxe-efi.iso -e efiboot.img -no-emul-boot efi_iso/
```

### 5. Rebuild after editing `boot.ipxe`

No `make clean` needed — the embed file is baked in at link time:

```bash
make bin-x86_64-efi/snp.efi EMBED=boot.ipxe && \
cp bin-x86_64-efi/snp.efi efi_iso/EFI/BOOT/BOOTX64.EFI && \
mcopy -o -i efi_iso/efiboot.img efi_iso/EFI/BOOT/BOOTX64.EFI ::/EFI/BOOT/BOOTX64.EFI && \
xorrisofs -o ipxe-efi.iso -e efiboot.img -no-emul-boot efi_iso/
```

## HTTP Server

Serve the Ubuntu netboot files and ISO:

```
assets/
├── ubuntu_24.04/
│   ├── amd64/
│   │   ├── initrd        # from netboot tarball (NOT from casper/)
│   │   ├── linux          # from netboot tarball (NOT from casper/)
│   │   └── ...
│   └── ubuntu-24.04-live-server-amd64.iso
└── ipxe-efi.iso
```

```bash
cd assets && python3 -m http.server 8000
```

The `initrd` and `linux` must come from the **netboot tarball** (`ubuntu-24.04.1-netboot-amd64.tar.gz`), not from `casper/` inside the ISO. The casper initrd has no network drivers — it expects the ISO to be locally mounted.

## Key Decisions

| Problem | Solution |
|---------|----------|
| UEFI-only board (no legacy BIOS) | Build `snp.efi`, package in FAT image inside ISO |
| iPXE freezes on multi-NIC boards | Use `snp.efi` (UEFI SNP protocol) instead of `ipxe.efi` (native drivers) |
| `snponly.efi` picks wrong NIC | Use `snp.efi` to see all interfaces |
| EFI ISO stalls at 33KB | Wrap `.efi` in a FAT image (`efiboot.img`) for El Torito |
| iPXE static IP needs `ifopen`/`set` | `dhcp` does this automatically; static requires explicit `ifopen` + `set net0/ip` etc. |
| Linux kernel can't find NIC by name | Pass `BOOTIF=01-${${nic}/mac:hexhyp}` — initramfs matches MAC to device name |
| Casper initrd has no network drivers | Use netboot tarball's `initrd` and `linux` instead |

## Tested Hardware

- **Board**: Asus PRO WS B850M-ACE SE (AM5, UEFI-only)
- **BMC**: ASPEED AST2600 (standard IPMI, virtual media ISO mount)
- **NICs**: Realtek 10GbE + Intel 2.5GbE (integrated) + Mellanox ConnectX-4 (PCIe)
- **Boot NIC**: Mellanox ConnectX-4 via `snp.efi` (shows as `net1` in iPXE, `enp1s0f1np1` in Linux)
