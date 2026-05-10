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
### Billentyűzetkiosztás beállítása (alapértelmezetten amerikai – US)
Az elérhető kiosztások listázása;
```
localectl list-keymaps
```
és a kívánt kiosztás betöltése.
```
loadkeys <a kívánt kiosztás neve>
```
### Hálózat Beállítása (WiFi)
Az iwctl paranccsal tudunk felkapcsolódni vezetéknélküli hálózatra.
```
iwctl
```
Először be kell kapcsolnunk az adaptert (ennek általában phy0 a neve, azért érdemes ellenőrizni).
```
adapter list

adapter phy0 set-property Powered on

station wlan0 scan

station wlan0 get-networks

station wlan0 connect "HálózatNeve"

exit
```
Ezután csak be kell írnod a jelszót és kész is vagy.
(_Ha esetleg valami oknál fogva makacskodna a wifi adapter, ezzel a paranccsal unlbock-olni lehet: "rfkill unblock wifi". Ezuán próbáld újra._

Ellenőrizd hogy élő-e a kapcsolat:
```
ping google.com
```

#### Ha távolról, SSH-n keresztül szeretnél bejelentkezni a célgépre [elhagyható]
Állíts be jelszót a root felhasználónak;  
```
passwd
```
majd ellenőrizd, hogy az SSH-szolgáltatás fut-e.
```
systemctl status sshd
```
_Ha nem fut, indítsd el így:_ `systemctl start sshd`.

Egy másik gépről tusz ssh-val csatlakozni, ha esetleg így könnyebb a telepítés. Ezzel a paranccsal meg tudod nézni, mi a géped ip-címe:
```
ip addr show
```
A másik gépedről:
```
ssh root@192.168.1.XXX
```
Itt hozzá kell adnod a kulcsot az ismert host-ok listájához, majd meg kell adnod a jelszót, amit a passwd paranccsal készítettél. A további lépéseket már a másik gépedről tudod majd folytatni.

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
> - Méret: 2048M  
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
BTRFS_OPTS="rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,commit=120"
```
Gyökér subvolume (@) felcsatolása a /mnt-re.
```
mount -o $BTRFS_OPTS,subvol=@ /dev/mapper/arch-linux /mnt
```
Könyvtárak létrehozása a csatolandó subvolume-oknak.
```
mkdir -p /mnt/{boot/efi,home,opt,srv,var/cache,/var/lib/libvirt/images,var/log,var/spool,var/tmp,.snapshots,efi}
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
mount /dev/nvme0n1p1 /mnt/efi
```
### Pacman PGP kulcsok beszerzése
```
pacman-key --init
pacman-key --populate
```
### Alap rendszer telepítése
```
pacstrap /mnt base base-devel linux-zen linux-zen-headers linux-firmware btrfs-progs snapper limine amd-ucode sudo pacman neovim efibootmgr systemd efivar sbsigntools git binutils openssh networkmanager
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
echo "LANG=en_US.UTF-8" >> /etc/locale.conf
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
### Initramfs generálás (mkinitcpio)

A titkosított és titkosítatlan UUIDk mentése változókba
```
CRYPT_UUID=$(blkid -s UUID -o value /dev/nvme0n1p2)
ROOT_UUID=$(blkid -s UUID -o value /dev/mapper/arch-linux)
```
Hozzunk létre a kernel cmdline beállítási fájlt, amiben a kernel betöltési paraméterei vannak
```
cat <<EOL > /etc/kernel/cmdline
rd.luks.name=$CRYPT_UUID=arch-linux root=UUID=$ROOT_UUID rootfstype=btrfs rootflags=subvol=@,compress=zstd:3,relatime rw quiet splash
EOL
```

Az `/etc/mkinitcpio.conf` fájlban ezeken a helyeken kell beállításokat módosítani:
_Hozzák kell adni a btrfs-t a MODULES-hoz._
```
MODULES=(btrfs)
```
_A HOOKS-hoz ezeket kell hozzáadni. (Systemd hoohkokat használunk, természetesen ezek helyett lehet az alap rendszer hookokat is.)_
```
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems resume fsck)
```
Hozzuk létre az UKIk tárolására a Linux mappát az esp-n.
```
mkdir -p /efi/EFI/Linux
```
Generáljuk újra a kerneleket:
```
mkinitcpio -P
```
Ha minden jól ment, akkor létre is hoztuk a zen és a fallback uki-t is! (`ls /efi/EFI/Linux` és ott látnunk kell az `arch-linux-zen.efi` és az `arch-linux-zen-fallback.efi` fájlokat!)

#### Rendszer betöltő telepítése (Limine)
A `pacstrap` paranccsal már telepítettük a limine-t, így most már csak dolgoznunk kell vele. Először is létre kell hoznunk neki egy mappát, és át kell másolnunk a bootloader fájlt:

```
mkdir -p /efi/EFI/arch-limine

cp /usr/share/limine/BOOTX64.EFI /efi/EFI/arch-limine/
```

Létre kell hoznunk egy konfigurációs fájlt ahhoz, hogy az UKI-kat lássa és be is tudja tölteni. (_A systemd-boottal ellentétben, a kernelek keresése nem automatikus!_)

```
nvim /efi/EFI/arch-limine/limine.conf
```
Ezt másoljuk bele a fájlba:
```
timeout: 5

/Arch Linux (Zen UKI)
    protocol: efi_chainload
    image_path: boot():/EFI/Linux/arch-linux-zen.efi

/Arch Linux (Zen Fallback UKI)
    protocol: efi_chainload
    image_path: boot():/EFI/Linux/arch-linux-zen-fallback.efi
```
(Ha a későbbiekben bármi mást - mondjuk efi-shell-t vagy memtestet akarunk hozzáadni, azokat is itt fogjuk tudni megtenni.)

### Frissítsük az EFI Boot Entry-t manuálisan
Listázzuk ki az elérhető EFI entryket:
```
efibootmgr
```
Ha NEM LÁTOD a limine entry-t, akkor hozzá kell adni manuálisan:
```
efibootmgr -c -d /dev/nvme0n1 -p 1 -L "Arch Linux" -l '\EFI\systemd\systemd-bootx64.efi'
efibootmgr -c -d /dev/nvme0n1 -p 1 -L "Arch Linux Limine Boot Loader" -l '\EFI\arch-limine\BOOTX64.EFI' --unicode
```
Az `efibootmgr -o`-val változtatni is tudod a bejegyzések sorrendjét:

```
efibootmgr -o 0000,0001,0002

vagy

efibootmgr -o 0001,0000,0002
```
Mindez attól függ, mit szeretnél bootolni - közvetlenül az UKIt, vagy a Systemd-bootmanagert. :)

# Teljes asztali környezet telepítése és testreszabása | _mesa, pipewire, Hyprland_

_az egész rendszer egyben:_
```
pacman -S \
mesa lib32-mesa mesa-utils vulkan-radeon lib32-vulkan-radeon \
vulkan-tools vulkan-validation-layers libva-mesa-driver \
lib32-libva-mesa-driver xf86-input-libinput xf86-video-amdgpu \
\
bluez bluez-utils bluetui qutebrowser zen-browser-bin \
rpcbind openssh \
\
hyprland hyprlock hyprpicker hyprpolkitagent \
wayland wayland-protocols qt5-wayland qt6-wayland \
gtk3 waybar rofi wofi polkit polkit-gnome greetd \
greetd-tuigreet uwsm ttf-nerd-fonts-symbols ttf-fira-code-nerd \
\
tldr tree cliphist fd fzf ripgrep btop btrfs-progs \
snapper xdg-utils xdg-desktop-portal xdg-user-dirs \
xdg-desktop-portal-gtk xdg-desktop-portal-hyprland \
dosfstools \
\
man-db man-pages tldr kitty neovim lazygit \
lua-language-server pyright keepassxc obsidian yazi \
\
gamemode lib32-gamemode steam lib32-alsa-plugins lib32-libpulse \
pipewire pipewire-alsa pipewire-audio pipewire-pulse wireplumber \
wiremix alsa-utils gst-libav gst-plugins-bad gst-plugins-good \
sof-firmware gst-plugins-ugly obs-studio vesktop-bin
```

## Külön modulokban:
**Grafikus meghajtók (mesa, vulkan driverek)**
```
pacman -S \
mesa lib32-mesa mesa-utils vulkan-radeon lib32-vulkan-radeon \
vulkan-tools vulkan-validation-layers libva-mesa-driver \
lib32-libva-mesa-driver xf86-input-libinput xf86-video-amdgpu
```

**Hálózatkezelés és böngészők**
```
pacman -S \
bluez bluez-utils bluetui qutebrowser zen-browser-bin \
rpcbind openssh
```

**Grafikus felület és Ablakkezelés**
```
hyprland hyprlock hyprpicker hyprpolkitagent \
wayland wayland-protocols qt5-wayland qt6-wayland \
gtk3 waybar rofi wofi polkit polkit-gnome greetd \
greetd-tuigreet uwsm ttf-nerd-fonts-symbols ttf-fira-code-nerd
```

**Rendszereszközök/Segédprogramok**
```
pacman -S \
tldr tree cliphist fd fzf ripgrep btop btrfs-progs \
snapper xdg-utils xdg-desktop-portal xdg-user-dirs \
xdg-desktop-portal-gtk xdg-desktop-portal-hyprland \
dosfstools
```

**Fejlesztői és Termelékenységi Eszközök**
```
pacman -S \
man-db man-pages tldr kitty neovim lazygit \
lua-language-server pyright keepassxc obsidian yazi
```

**Játékok, multimédia, kommunikáció**
```
pacman -S \
gamemode lib32-gamemode steam lib32-alsa-plugins lib32-libpulse \
pipewire pipewire-alsa pipewire-audio pipewire-pulse wireplumber \
wiremix alsa-utils gst-libav gst-plugins-bad gst-plugins-good \
sof-firmware gst-plugins-ugly obs-studio vesktop-bin
```


---

## Login manager TUI-greet (greetd frontend)
Állítsuk be a greetd konfigurációs fájlját hogy lássa az összes wayland session-t.

```
nvim /etc/greetd/config.toml

vt = 1

[default_session]
command = "tuigreet -w 80 --sessions /usr/share/wayland-sessions"

user = "greeter"
```
Engedélyezzük a greetd szervizt
```
systemctl enable greetd.service
```
### Ha kettő vagy több monitort használsz
Észre fogod venni, hogy csak egy frame buffer van mind a kettő (vagy több) monitorodra és elég bután néz ki az,  
hogy a 2/4K-s monitorodon nem jó a loginmanager mérete. Ezt a `ddcutil` és `fbset` programok  
használatával tudod kiküszöbölni. Szép megoldás? Nem. De határozottan működik.  
_Ha két monitorod van, akkor egyet lekapcsolunk a bejelentkezésig. Ez készeríti majd a frame buffert hogy  
az elérhető monitoron a legnagyobb felbontást használja. Nekem egy HD és egy 2K-s monitorom van,  
ezen demonstrálom mit kell tenni._

**1. Ha eddig nem tetted meg, telepítsd a szükséges programokat**
```
pacman -S ddcutil fbset
```
**2. Szerezzünk jogosulságokat az `i2c`-hez**
```
usermod -aG i2c $USER
```
**2.1. Nézzük meg hogy melyik monitor a display 1 vagy dispaly 2 (stb).**
```
ddcutil detect
```
**3. Hozzunk létre egy scriptet, ami standby módba teszi az egyik monitort login előtt**  
_!!Nálam ez a HD lesz!!_
```
nvim /usr/local/bin/m-prelogin.sh

#!/bin/sh

# Monitor 1 (HD) - standby
ddcutil setvcp D6 04 --display 1

# Fő monitor felbontása
fbset -xres 2560 -yres 1440

```
Tegyük futtathatóvá a scriptet
```
chmod +x /usr/local/bin/m-prelogin.sh
```
Hozzunk létre hozzá egy systemd unitot
```
nvim /etc/systemd/system/m-prelogin.service

[Unit]
Description=Set monitor standby and resolution before login
After=multi-user.target
Before=graphical.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/m-prelogin.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
Tegyük aktívvá:
```
sudo systemctl enable m-prelogin.service
```
**4. Hozzunk létre egy scriptet ami visszakapcsolja a monitort login után**
_Ezt úgy állítottam be, hogy lekérdezze ki jelentkezik be -így ha több felhazsnáló is  
van a gépen, mindenkinél működni fog._
```
nvim /usr/local/bin/m-postlogin.sh

#!/bin/bash
ddcutil setvcp D6 01 --display 1

```
Tegyük futtathatóva:
```
chmod +x /usr/local/bin/m-postlogin.sh
``` 
Hozzunk létre egy systemd unitot ehhez is.
```
nvim ~/.config/systemd/user/m-postlogin.service

[Unit]
Description=Enable second monitor after login
After=graphical-session.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/m-postlogin.sh

[Install]
WantedBy=default.target
```
Engedélyezzük:
```
systemctl --user daemon-reexec
systemctl --user daemon-reload
systemctl --user enable m-postlogin.service

```
Nézd meg, hogy a Linger=yes szerepel-e a loginctl-ben.
```
loginctl show-user $USER

ha nem:

sudo loginctl enable-linger $USER
```
A secureboot és apparmor modulok a következő részben kerülnek tárgyalásra.

(obs-gamecapture env OBS_VKCAPTURE=1 LD_PRELOAD="" XKB_DEFAULT_LAYOUT=hu gamescope -w 2560 -h 1440 -W 2560 -H 1440 -f --force-grab-cursor -- %command%)
