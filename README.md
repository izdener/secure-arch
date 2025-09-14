# secure-arch - _Erősített Arch Linux telepítési útmutató._
   - [btrfs](https://btrfs.readthedocs.io/en/latest/): Egy sokoldalú, másolatkészítés-alapú (copy-on-write) fájlrendszer Linuxhoz.  
   - [encryption](https://gitlab.com/cryptsetup/cryptsetup/): LUKS 2 lemeztitkosítás a dm-crypt kernelmodulra építve.
   - [zram](https://www.kernel.org/doc/html/v5.9/admin-guide/blockdev/zram.html): RAM tömörítés a memória megtakarítása érdekében.
   - [hyprland](https://hypr.land/): Modern, látványos Wayland-kompozitor.
   - [secure boot](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot): Biztonsági funkció, amely csak megbízható, aláírt szoftverek elindítását engedélyezi rendszerindításkor.
   - [apparmor](https://wiki.archlinux.org/title/AppArmor): Az AppArmor egy kötelező hozzáférés-vezérlési (MAC) rendszer, amely a Linux Security Modules (LSM) keretrendszerre épül.
## Arch telepítő eszköz létrehozása
- A hivatalos telepítési útmutatóra folyamatosan hívatkozni fogok, itt a [linkje](https://wiki.archlinux.org/title/Installation_guide).
- Szerezd be a telepítőképet innen: [Arch Linux letöltés](https://archlinux.org/download/)
- Ellenőrizd a letöltött Arch ISO fájl digitális aláírását (lásd az Arch telepítési útmutató 1.2-es pontját).
- Írd ki az ISO fájlt egy USB meghajtóra - én a [Ventoy](https://www.ventoy.net/en/index.html)t ajánlom.
- Majd az elkészült USB eszközt helyezd be a gépbe amire telepíteni szeretnél, és bootold be róla az Arhc linuxot.
## Rendszer előkészítése Arch ISO-val
### Ha távolról, SSH-n keresztül szeretnél bejelentkezni a célgépre [elhagyható]
Állíts be jelszót a root felhasználónak;  
```
passwd
```
majd ellenőrizd, hogy az SSH-szolgáltatás fut-e.
```
systemctl status sshd
```
_Ha nem fut, indítsd el így:_ `systemctl start sshd`.
### Billentyűzetkiosztás beállítása (alapértelmezetten amerikai – US)
Az elérhető kiosztások listázása;
```
localectl list-keymaps
```
és a kívánt kiosztás betöltése.
```
loadkeys <a kívánt kiosztás neve>
```
### Csatlakozás az internethez 
Használhatod az `iwctl` segédprogramot Wi-Fi kapcsolathoz;  
```
ping -c 2 archlinux.org
```
_Ellenőrizd a kapcsolatot nézd meg az IP-címedet:_ `ip addr show` _— ezután már készen állsz az SSH kapcsolatra is._
### Időzóna beállítása
Időzónák listázása;
```
timedatectl list-timezones
```
időzóna beállítása;
```
timedatectl set-timezone RÉGIÓ/VÁROS
```
NTP (időszinkronizálás) engedélyezése.
```
timedatectl set-ntp true
```

### Lemez particionálása
Jelenlegi partíciók listázása: `lsblk -f`.
- Töröld a céllemez meglévő partícióit (FIGYELEM: minden adat elvész!).  
Hozz létre két új partíciót:
> 💡 Megjegyzés: Az Arch hivatalos telepítési útmutatója javasolja a swap partíció létrehozását, de helyette a zram sokkal inkább kedvezőbb, mivel nem használja el az nvme-ssd-ket.  
- EFI partíció = 1024 MB  
- Fő partíció = a fennmaradó hely (vagy ahogy az adott eset megkívánja - _Btrfs fájlrendszer nem igényel előre meghatározott partícióméreteket, mivel al-alrészeket (subvolumes) használ, amelyek viselkedésükben hasonlítanak a partíciókhoz, de nem igényelnek fizikai felosztást_)
Ezt a műveletet `parted`, `fdisk`, vagy mint itt, `cfdisk` használatával is elvégezhetjük.
`cfdisk /dev/nvme0n1`:  
> Válassz GPT-t (ha még nincs), válaszd: gpt
> Hozd létre az ESP partíciót: New
> - Méret: 1024M  
> - Típus: válaszd EFI System
> 
> Hozd létre a root (LUKS) partíciót: New
> - Méret: a maradék teljes lemez
> - Típus: hagyhatod alapértelmezettnek (Linux filesystem), mivel úgyis titkosítani fogjuk
> 
> Írd ki a partíciós táblát: Write
> - Gépelj be: yes
> 
> Végül: Quit
#### A fő partíció előkészítése, formázása
Titkosítás beállítása;
```
cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
```
a titkosított partíció megnyitása;
```
cryptsetup open --allow-discards --persistent /dev/nvme0n1p2 arch-linux
```
partíció formázása BTRFS-sel;
```
mkfs.btrfs /dev/mapper/arch-linux
```
partíció felcsatolása a telepítéshez:
```
mount /dev/mapper/arch-linux /mnt
```
Subvolume-ok létrehozása:
```
btrfs subvolume create  /mnt/@
btrfs subvolume create  /mnt/@home
btrfs subvolume create  /mnt/@opt
btrfs subvolume create  /mnt/@srv
btrfs subvolume create  /mnt/@cache
btrfs subvolume create  /mnt/@images
btrfs subvolume create  /mnt/@log
btrfs subvolume create  /mnt/@spool
btrfs subvolume create  /mnt/@tmp
btrfs subvolume create  /mnt/@snapshots
```
> 💡 Megjegyzés: A subvolume-ok elnevezése tetszőleges, de ügyelj arra, hogy később ezekre fogunk hivatkozni például a snapper használatakor.

A partíció leválasztása:
```
umount /mnt
```


_Csatolási lehetőségek elmentése többször felhasználható bash alias-ban._
```
BTRFS_OPTS="rw,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,commit=120"
```
Gyökér subvolume (@) felcsatolása a /mnt-re.
```
mount -o $BTRFS_OPTS,subvol=@ /dev/mapper/arch-linux /mnt
```
Könyvtárak létrehozása a csatolandó subvolume-oknak.
```
mkdir -p /mnt/{boot/efi,home,opt,srv,var/cache,/var/lib/libvirt/images,var/log,var/spool,var/tmp,.snapshots,media/extra-data}
```
Subvolume-ok felcsatolása
```
mount -o $BTRFS_OPTS,subvol=@home /dev/mapper/arch-linux /mnt/home
mount -o $BTRFS_OPTS,subvol=@srv /dev/mapper/arch-linux /mnt/srv
mount -o $BTRFS_OPTS,subvol=@opt /dev/mapper/arch-linux /mnt/opt
mount -o $BTRFS_OPTS,subvol=@images /dev/mapper/arch-linux /mnt/var/lib/libvirt/images
mount -o $BTRFS_OPTS,subvol=@cache /dev/mapper/arch-linux /mnt/var/cache
mount -o $BTRFS_OPTS,subvol=@log /dev/mapper/arch-linux /mnt/var/log
mount -o $BTRFS_OPTS,subvol=@tmp /dev/mapper/arch-linux /mnt/var/tmp
mount -o $BTRFS_OPTS,subvol=@snapshots /dev/mapper/arch-linux /mnt/.snapshots
mount -o $BTRFS_OPTS,subvol=@spool /dev/mapper/arch-linux /mnt/var/spool
```
#### EFI partició előkészítése, formázása
ESP formázása fat32-vel;
```
mkfs.fat -F32 /dev/nvme0n1p1
```
ESP partíció csatolása az efi könyvtárra.
```
mount /dev/nvme0n1p1 /mnt/boot/efi
```
### Pacman PGP kulcsok beszerzése
```
pacman-key --init
pacman-key --populate
```
### Alap rendszer telepítése
```
pacstrap /mnt base base-devel linux-zen linux-zen-headers linux-firmware btrfs-progs amd-ucode sudo pacman neovim dracut efibootmgr systemd-boot sbsigntools git binutils openssh networkmanager
```
Hozzuk létre az fstab-ot.
```
genfstab -U /mnt >> /mnt/etc/fstab
```

## Beállítások a chroot környezetben
Lépjünk be az új rendszerünkbe.
```
arch-chroot /mnt
```
Állítsunk be root jelszót
```
passwd
```
Állítsuk be az időt - most már nem csak az ISO-n.
```
ln -sf /usr/share/zoneinfo/RÉGIÓ/VÁROS /etc/localtime

hwclock --systohc
```
Állítsuk be a rendszer nyelvét.
_Kommenteld ki a nyelve(ke)t amit használni szeretnél._
```
nvim /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" >> /etc/locale.conf`
```
Állítsuk be a billentyűzet kiosztást.
```
echo "KEYMAP=hu" >> /etc/vconsole.conf
```
Állítsuk be a gép nevét.
```
echo "GÉP-NEVE" >> /etc/hostname
```
### Felhasználó(k) hozzáadása, és wheel
Saját felhasználó hozzáadása.
```
useradd -m A_TE_NEVED
passwd A_TE_NEVED
```
Felhasználó hozzáadása a wheel csoporthoz - sudo
```
EDITOR=nvim visudo

   %wheel	ALL=(ALL) ALL # kommenteld ki ezt a sort

usermod -aG wheel A_TE_NEVED
```
### Alap systemd szervizek elindítása
```
systemctl enable NetworkManager # a kapitalizáció fontos!
systemctl enable fstrim.timer
```
### Initramfs generálás (dracut)
Adjuk hozzá a kernel modulokat és beállításokat egy fájlhoz.   
_A_ `base.conf` _név mellett döntöttem, mert nekem ez logikus._
```
nvim /etc/dracut.conf.d/base.conf

hostonly=yes
add_dracutmodules+=" crypt btrfs "
uefi=yes
```
Generáljuk újra az initramfs-t.
```
dracut --force
```
#### Rendszer betöltő telepítése (systemd-boot)
```
bootctl install
```
A titkosított és titkosítatlan UUIDk mentése változókba
```
CRYPT_UUID=$(blkid -s UUID -o value /dev/nvme0n1p2)
ROOT_UUID=$(blkid -s UUID -o value /dev/mapper/arch-linux)
```
Kernel boot entry készítése
```
cat <<EOL > /boot/loader/entries/arch.conf
title   Arch Linux (linux-zen)
linux   /vmlinuz-linux-zen
initrd  /initramfs-linux-zen.img
options rd.luks.name=$CRYPT_UUID=cryptroot root=UUID=$ROOT_UUID rootflags=subvol=@ rw
EOL
```
loader.conf szerkesztése
```
cat <<EOL > /boot/loader/loader.conf
default arch
timeout 3
editor no
EOL
```

### Teljes asztali környezet telepítése - mesa, pipewire, Hyprland
```
pacman -S \
mesa lib32-mesa vulkan-radeon libva-mesa-driver \
xf86-video-amdgpu pipewire pipewire-audio \
pipewire-alsa pipewire-pulse wireplumber \
alsa-utils wiremix greetd tuigreet \
hyprland xdg-desktop-portal-hyprland xdg-desktop-portal \
xdg-desktop-portal-gtk wlroots wayland wayland-protocols \
qt5-wayland qt6-wayland gtk3 xdg-utils hyprlock hyprpicker \
wofi waybar polkit polkit-gnome elogind dbus uwsm kitty \
ripgrep tldr man-db man-pages bluez bluez-utils \
fcitx5 fcitx5-im fcitx5-configtool fcitx5-gtk fcitx5-qt \
ttf-firacode-nerd \
journalctl-tui bluetui btop
```

A secureboot és apparmor modulok a következő részben kerülnek tárgyalásra.
