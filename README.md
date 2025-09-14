# secure-arch - _Erősített Arch Linux telepítési útmutató._
   - [btrfs](https://btrfs.readthedocs.io/en/latest/): Egy sokoldalú, másolatkészítés-alapú (copy-on-write) fájlrendszer Linuxhoz.  
   - [encryption](https://gitlab.com/cryptsetup/cryptsetup/): LUKS 2 lemeztitkosítás a dm-crypt kernelmodulra építve.
   - [zram](https://www.kernel.org/doc/html/v5.9/admin-guide/blockdev/zram.html): RAM tömörítés a memória megtakarítása érdekében.
   - [hyprland](https://hypr.land/): Modern, látványos Wayland-kompozitor.
   - [secure boot](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot): Biztonsági funkció, amely csak megbízható, aláírt szoftverek elindítását engedélyezi rendszerindításkor.
   - [apparmor](https://wiki.archlinux.org/title/AppArmor): Az AppArmor egy kötelező hozzáférés-vezérlési (MAC) rendszer, amely a Linux Security Modules (LSM) keretrendszerre épül.

## Első lépés: Arch telepítő eszköz létrehozása
  1. A hivatalos telepítési útmutatóra folyamatosan hívatkozni fogok, itt a [linkje](https://wiki.archlinux.org/title/Installation_guide).
  2. Szerezd be a telepítőképet innen: [Arch Linux letöltés](https://archlinux.org/download/)
  3. Ellenőrizd a letöltött Arch ISO fájl digitális aláírását (lásd az Arch telepítési útmutató 1.2-es pontját).
  4. Írd ki az ISO fájlt egy USB meghajtóra - én a [Ventoy](https://www.ventoy.net/en/index.html)t ajánlom.
  5. Majd az elkészült USB eszközt helyezd be a gépbe amire telepíteni szeretnél, és bootold be róla az Arhc linuxot.

## Második lépés: rendszer előkészítése Arch ISO-val
**0. [elhagyható] Ha távolról, SSH-n keresztül szeretnél bejelentkezni a célgépre:**  
Állíts be jelszót a root felhasználónak, majd ellenőrizd, hogy az SSH-szolgáltatás fut-e:  
```
passwd
systemctl status sshd
```
_ha nem, indítsd el ezzel: `systemctl start sshd`_.

**1. Billentyűzetkiosztás beállítása (alapértelmezetten amerikai – US):**  
Az elérhető kiosztások listázása, és a kívánt kiosztás betöltése:
```
localectl list-keymaps

loadkeys <a kívánt kiosztás neve>
```

**2. Csatlakozás az internethez:**  
Használhatod az `iwctl` segédprogramot Wi-Fi kapcsolathoz;  
```
ping -c 2 archlinux.org
```
_Ellenőrizd a kapcsolatot nézd meg az IP-címedet:_ `ip addr show` _— ezután már készen állsz az SSH kapcsolatra is._

**4. Időzóna beállítása:**  
Időzónák listázása, időzóna beállítása, NTP (időszinkronizálás) engedélyezése.
```
timedatectl list-timezones
timedatectl set-timezone RÉGIÓ/VÁROS
timedatectl set-ntp true
```

**5. Lemez particionálása:**  
Jelenlegi partíciók listázása: `lsblk`. Töröld a céllemez meglévő partícióit (FIGYELEM: minden adat elvész!).  
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

**6. lépés: A fő partíció előkészítése, formázása**
Titkosítás beállítása, a titkosított partíció megnyitása;  
partíció formázása BTRFS-sel, és a partíció felcsatolása a telepítéshez:  
```
cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
cryptsetup open --allow-discards --persistent /dev/nvme0n1p2 arch-linux

mkfs.btrfs /dev/mapper/arch-linux
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

Subvolume-ok csatolása megfelelő opciókkal:
```
btrfsopts="relatime,ssd,compress=zstd:3,space_cache=v2,discard=async"
```


**7. EFI partició előkészítése, formázása**
EFI formázása fat32-vel, boot könyvtár létrehozása, partíció csatolása
```
mkfs.fat -F32 /dev/nvme0n1p1
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```
