# secure-arch - _Erősített Arch Linux telepítési útmutató._
   - [btrfs](https://btrfs.readthedocs.io/en/latest/): Egy sokoldalú, másolatkészítés-alapú (copy-on-write) fájlrendszer Linuxhoz.  
   - limine bootloader és snapper integráció
   - [encryption](https://gitlab.com/cryptsetup/cryptsetup/): LUKS 2 lemeztitkosítás a sd-crypt kernelmodulra építve.
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
pacstrap /mnt base base-devel linux-zen linux-zen-headers linux-firmware btrfs-progs snapper snap-pac limine amd-ucode sudo pacman neovim efibootmgr systemd efivar sbsigntools git binutils openssh networkmanager
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

#### Limine Pacman hook
Amikor frissül a Limine, akkor automatikus másolja a pacman a helyére a fájlt.

```
nvim /etc/pacman.d/hooks/99-limine.hook
```
Ezt másoljuk bele:
```
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = limine              

[Action]
Description = Deploying Limine after upgrade...
When = PostTransaction
Exec = /usr/bin/cp /usr/share/limine/BOOTX64.EFI /efi/EFI/arch-limine/
```

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
Mindez attól függ, mit szeretnél bootolni - közvetlenül az UKIt, vagy a Limine boot-managert. :)

## Paru AUR helper telepítése
Az Arch User Repository - röviden AUR hasznosságát nem lehet figyelmen kívül hagyni, így telepítsünk fel egy helpert, ami segít az ott lévő alkalmazások telepítésében.

```
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```
Ezek után a `pacman` helyett, használhatjuk a `paru`-t.

# Teljes asztali környezet telepítése és testreszabása: _mesa, pipewire, Hyprland_ használéatával

## Pacman és Paru beállítások
Engedélyeznünk hell a pacman multilib repo-t, és a parunál hogy az alkalmazások telepítésénél részesíste előnybe a tárolókban lévő programokat.
```
nvim /etc/pacman.conf
```
Kommenteld ki ezt a két sort:
```
[multilib]
Include = /etc/pacman.d/mirrorlist
```


_Rendszer telepítése:_
```
paru -S \
mesa lib32-mesa mesa-utils libva-mesa-driver lib32-libva-mesa-driver \
vulkan-radeon lib32-vulkan-radeon vulkan-tools vulkan-validation-layers \
xf86-input-libinput xf86-video-amdgpu pipewire pipewire-alsa pipewire-audio \
pipewire-pulse wireplumber alsa-utils gst-libav sof-firmware gst-plugins-ugly \
gst-plugins-bad gst-plugins-good bluez bluez-utils udiskie gparted \
brightnessctl playerctl grim slurp steam gamemode lib32-gamemode \
lib32-alsa-plugins lib32-libpulse hyprland hyprlock hyprpicker hyprpaper \
polkit hyprpolkitagent dunst wayland-protocols qt5-wayland qt6-wayland gtk3 \
waybar wofi uwsm libnewt wl-clipboard xdg-desktop-portal xdg-desktop-portal-hyprland \
xdg-desktop-portal-gtk xdg-utils xdg-user-dirs dosfstools sysc-greet-hyprland \
man-db man-pages tldr tldr tree fd fzf ripgrep kitty yazi power-profiles-daemon \
noto-fonts noto-fonts-emoji ttf-nerd-fonts-symbols ttf-firacode-nerd \
qutebrowser zen-browser-bin openssh avahi ffmpegthumbnailer ttf-liberation \
zram-generator btop htop cliphist keepassxc bluetui wiremix webcord
```

## Login manager [sysc-greet](https://nomadcxx.github.io/sysc-greet/) (greetd frontend)
A kompozitorunknak megfelelően be kell állítanunk a bejelentkező képernyőt. Mivel már előzőleg telepítettük, nincs más dolgunk, csak átállítani a billentyűzetkiosztást, és engedélyezni a szervízt.

Állítsd át itt a billentyűzeted kiosztását:
```
nvim /etc/greetd/hyprland-greeter-config.conf
```

Szervíz engedélyezése:
```
sudo systemctl enable greetd.service
```

## Power Profiles Daemon és rendszer finomhangolás
A szükséges programot `power-profiles-daemon` már fentebb telepítettük, és csak engedélyeznünk kell.

```
sudo systemctl enable --now power-profiles-daemon
```
---
# Mélyebb rendszer beállítások
## Hasznos `sysctl` paraméterek
Hozzunk létre ehhez egy gyűjtő fájlt a `/etc/sysctl.d/` könyvtárban. nevezzük el `70-system-settings.conf`-nak.
```
nvim /etc/sysctl.d/70-system-settings-conf
```

```
#################################
### Virtuális memória kezelés
# Ez határozza meg hogy a rendszer mennyire aggresszívan használja a swap területet.
# Ha NVME SSD-d van, akkor a 100 nagyon jó érték, főleg ha ZRAM-ot is beállítjuk.
vm.swappiness = 100

# Ez szabályozza, hogy a kernel mennyire szívesen tartja RAM-ban a fájlrendszer-struktúrákat
# (könyvtárak, fájlinformációk - dentry/inode). Alapérték 100. Ez kicsit felgyorsítja a fájlok
# keresését, így kevesebbek kell olvasni a lemezről.
vm.vfs_cache_pressure = 50

# Ez határozza meg hogy a kernel műveletenként hány memória oldalt olvasson be/írjon ki swapból/ba.
# Mivel nincs mozgó alaktrész, nincs felpörgetési idő, ezért oldalanként fog történni a műveletsor.
# Nagyjából 4 KB/page. SSD és zRAM esetében ez optimális.
vm.page-cluster = 0

#################################
### Lemezműveletek (Dirty Pages)
## Ezek olyan adatokat jelentenek, amik már megváltoztak, de még nem íródtak ki a lemezre.
# Amint az írásra váró adatok mennyisége eléri a 256 MB-ot, a folyamat megáll, és kénytelen megvárni,
# amíg az adatok kiíródnak a lemezre. Így a RAM kevésbé lesz tele adattal aminek lemezen a helye,
# és így kisebb eséllyel lesz rendszer fagyás.
vm.dirty_bytes = 268435456

# Ez úgy működik nagyjából mint az előző beállítás, itt viszont a háttéradatok mozgatását szabályozza.
# 65 MB = 67108864 byte
vm.dirty_background_bytes = 67108864

# Ezzel szabályozható hogy a kernel hány másodpercenként ellenőrizze hogy van-e kiírandó adat.
# Az alapértelmezett érték 500, ezen érték emelése csökkenti a lemezműveletek számát, ami energiát
# spórol - nagyon jó laptopoknál.
vm.dirty_writeback_centisecs = 1500

#################################
### Rendszer és biztonság (Kernel)
# Az nmi_watchdog egy "lefagyás figyelő". Én le szoktam kapcsolni - egyel kevesebb folyamat, ami
# eszi az erőforrást. Általában ha stabil a rendszer, nem baj hogy nem megy.
kernel.nmi_watchdog = 0

# Ez a beállítás engedélyezi hogy átlak jogosultságú felhasználók is létre tudjanak hozni rootless
# konténereket/izolációs egységeket pl. docker, bubblewrap, flatpak segítségével.
kernel.unprivileged_userns_clone = 1

# Ez eltűnteni a boot folyamatból az alacsony prioritású üzeneteket. Csak kritikus hibák látszódnak.
kernel.printk = 3 3 3 3

# Biztonsági beállítás, ami megakadályozza hogy felhasználók lássák a kernel memóriácímeit (pointers).
# Megnehezíti a kernel elleni exploitok végrehajtását.
kernel.kptr_restrict = 2

#################################
### Hálózat és fájlrendszer
# Ez megnöveli a bejövő hálózati csomagok várólistáját - jó beállítás, ha gyorsa az internet kapcsolat.
# Több ideje marad a processzornak a feldolgozásra, mielőtt a kernel kiszórná őket.
net.core.netdev_max_backlog = 4096

# Felemeli rendszerszinten az egyszerre megnyitható fájlok maximális számát.
# Így nehezebb kifutni a descriptorokból.
fs.file-max = 2097152
```


A secureboot és apparmor modulok a következő részben kerülnek tárgyalásra.

(obs-gamecapture env OBS_VKCAPTURE=1 LD_PRELOAD="" XKB_DEFAULT_LAYOUT=hu gamescope -w 2560 -h 1440 -W 2560 -H 1440 -f --force-grab-cursor -- %command%)
