# MyGentoo
 Guia para instalar Gentoo con un sistema de archivos btrfs, y systemd

# Guia Gentoo con systemd
En esta instalacion usaremos cualquier distro de linux, ya sea linux mint, fedora, etc, debido a que la iso minimal de gentoo no trae soporte para mi placa de wifi "Realtek Semiconductor RTL8852AE 802.11ax PCIe Wireless Network Adapter" (por el momento).

Arrancaremos como normalmente lo hariamos desde el liveusb de nuestra distro.
Abriremos una terminal y ejecuataremos el comando #-sudo su-#.

## Primero que nada verifiquemos si utilizamos bios o uefi para nuestro arranque, que dara muchos fallos al iniciar
``` 
ls /sys/firmware/efi 
```

## Verifiquemos primero si nuestra hora, dia y mes estan formateados correctamente

```
date
```

Por ejemplo, para ajustar la fecha y hora a las 13:16 horas del 3 de octubre del 2021, ejecute:

``` 
date 100313162021 
```

## Crearemos nuestro esquema de particion
Crearemos un esquema de archivos con btrfs, asi que aqui nos llevaremos tiempo

```
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
nvme0n1     252:0    0   238G  0 disk
├─nvme0n1p1 259:1  252:1    0 600M  0 /boo/efi  >sistema efi<
├─nvme0n1p2 259:2    0     1G  0 part /boot     >Sistema de ficheros linux<
└─nvme0n1p3 259:3    0 236.9G  0 part /home	>Sistema de ficheros linux<
                                      /

```

## Utilizaremos cfdisk
``` cfdisk /dev/nvme0n1< (si no indicamos el disco, se hara dentro del almacenamiento del usb) ```

## Dando formato a los volumenes creados
```
mkfs -t vfat -F 32 /dev/nvme0n1p1< (Es la particion de arranque efi)
mfks -t btrfs -L boot /dev/nvme0n1p2
mkfs -t btrfs -L btrfsroot /dev/nvmen01p3
```

### Aqui es donde crearemos nuestro punto de montaje
``` mkdir -p /mnt/gentoo ```

### Montamos a continuacion
``` mount /dev/nvmen01p3 /mnt/gentoo ```

### Creamos boot
``` 
mkdir -p /mnt/gentoo/boot 
mount /dev/nvmen01p2 /mnt/gentoo/boot	
```

### Crearemos los subvolumenes
```
cd /mnt/gentoo
btrfs subvol create root
btrfs subvol create home
btrfs subvol create srv
btrfs subvol create var
```

### Desmontamos luego de crearlos

```
cd ..
umount -l gentoo
```

# El comando que utilizaremos a continuacion se explicara, debido a que estoy guiandome de muchos lados:

- mount: Este comando se utiliza para montar un sistema de archivos en un directorio específico.

 - -o: Esta opción se utiliza para especificar las opciones de montaje adicionales.

 -   defaults: Esta opción indica que se deben utilizar las opciones de montaje predeterminadas.

  -  noatime: Esta opción desactiva la actualización del tiempo de acceso a los archivos cada vez que se accede a ellos, lo que puede ayudar a mejorar el rendimiento.

   - compress=lzo: Esta opción habilita la compresión LZO en el sistema de archivos montado, lo que permite comprimir los datos para ahorrar espacio de almacenamiento.

   - autodefrag: Esta opción habilita la desfragmentación automática en el sistema de archivos, lo que ayuda a mantener un rendimiento óptimo.

   - subvol=root: Esta opción especifica que se debe montar el subvolumen llamado "root" dentro del sistema de archivos Btrfs.

   - /dev/vda4: Esta es la ruta de la partición que se va a montar. En este caso, la partición /dev/vda4 se montará.

   - /mnt/gentoo: Este es el directorio en el cual se va a montar la partición /dev/vda4. En este caso, se montará en el directorio /mnt/gentoo.

### Empezamos a montar
```
mount -o defaults,noatime,compress=lzo,autodefrag,subvol=root /dev/nvme0n1p3 /mnt/gentoo

```

### Creamos los directorias para luego montarlos
```
cd /mnt/gentoo
mkdir srv home root var boot
```
### Montamos
```
mount -o defaults,relatime,compress=lzo,autodefrag,subvol=home /dev/nvme0n1p3 /mnt/gentoo/home
mount -o defaults,relatime,compress=lzo,autodefrag,subvol=srv /dev/nvme0n1p3 /mnt/gentoo/srv
mount -o defaults,relatime,compress=lzo,autodefrag,subvol=var /dev/nvme0n1p3 /mnt/gentoo/var
mount -o defaults,relatime /dev/nvmen1p2 /mnt/gentoo/boot

mount -o defaults,noatime /dev/nvme0n1p1 /mnt/gentoo/boot/efi
```
	
### Verifiquemos si estamos en el directorio /mnt/gentoo
```
pwd
```
Hecho esto procederemos a descargar el tarball de gentoo, el cual contiene toda las herramientas para seguir instalando gentoo, en este caso descargaremso el tarball con systemd, ya que se me es mas facil de comprender y es el init system que mas e ocupado.
```
wget (el link puede cambiar) https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/20230611T170207Z/stage3-amd64-systemd-20230611T170207Z.tar.xz (link del tarball no desktop para pruebas)

(link desktop) https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/20230611T170207Z/stage3-amd64-desktop-systemd-20230611T170207Z.tar.xz

```
### Procederemos a descomprimir el tarball
```
tar xpvf stage3-*(pulsar tab) --xattrs-include='*.*' --numeric-owner
(Por cualquier error probar >tar --numeric-owner --xattrs -xvjpf stage3-*.tar.bz2 -C /mnt/gentoo )
```

## Venimos a la parte mas complicada y es la configuracion del compilador

```
nano -w /mnt/gentoo/etc/portage/make.conf
```
### Configuraciones del compilador a aplicar en cualquier lenguaje
```
COMMON_CFLAGS="-march=native -O2 -pipe"
# Use los mismos valores en ambas variables
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${CFLAGS}"
```

Toca leer la edicion de make.conf, ya que esta la podemos adaptar a las diferentes necesidades nuestro pc, tambien las especificaciones de la pc son diferentes.

## Seleccionar servidores replicas
```
mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf
mkdir --parents /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
(Hacemos un cat de >/mnt/gentoo/etc/portage/repos.conf/gentoo.conf)
```

# Montar sistemas de archivos necesarios
```
mount -o bind /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run 
```

### Esto es por que se instala desde otra distro, si se hace desde el minimal de gentoo no es necesario
```
test -L /dev/shm && rm /dev/shm && mkdir /dev/shm 
mount --types tmpfs --options nosuid,nodev,noexec shm /dev/shm 
chmod 1777 /dev/shm /run/shm
```

# Entrar en el nuevo entorno
```
chroot /mnt/gentoo /bin/env -i TERM=$TERM /bin/bash 
env-update
source /etc/profile
export PS1="(chroot) $PS1" 
```

## Actualizar instantaneas de portage
```
emerge-webrsync
```

## Actualizar repositorio ebuild
```
emerge-sync
```

### Muy importante leer las noticias de portage, estos nos informan de errores y cambios que han tenido algunos paquetes
```
eselect news list
eselect news read
```

## Elegir perfil adecuado
```
(Por lo general esto esta dado por el tarball, pero nunca esta demas verificar, y ver si nos interesa algun cambio)
eselect profile list
eselect profile set #
```

# Actualizar el sistema
```
emerge --ask --verbose --update --deep --newuse @world
```

## Variables use
```
(Puedes verificar las variables de entorno con)
emerge --info | grep ^USE
(O podemos agregar las nuestras)
nano -w /etc/portage/make.conf
USE="-gtk -gnome qt5 kde dvd alsa cdr"(ejemplo)
```

## Actualizar nuestras CPU_FLAGS
```
emerge --ask app-portage/cpuid2cpuflags
cpuid2cpuflags
echo "*/* $(cpuid2cpuflags)" > /etc/portage/package.use/00cpu-flags
```

### Modificar las licencias que queremos aceptar
```
nano -w /etc/portage/make.conf
ACCEPT_LICENSE="*" (acepta toda las licencias)
```

### Zona horaria
```
ln -sf ../usr/share/zoneinfo/America/Managua /etc/localtime
```

### Configurar localizaciones
```
nano -w /etc/locale.gen
en_US ISO-8859-1
en_US.UTF-8 UTF-8
es_ES ISO-8859-1
es_ES.UTF-8 UTF-8 (Agregar esta linea para el formato spanish)
locale-gen
eselect locale list
eselect locale set #
```

### Recargamos variables de entorno
```
env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
```

###  Instalaremos el firmware
```
emerge --ask sys-kernel/linux-firmware
````

## Como nos queremos ahorrar tiempo en compilar kernel
```
emerge --ask sys-kernel/gentoo-kernel-bin
```

# Crear archivos fstab

```
nano -w /etc/fstab
(Con el esquema que estoy creando deberia de quedar algo asi)

shm         /dev/shm        tmpfs   nodev,nosuid,noexec                                 0 0
/dev/nvme0n1p3  /               btrfs   rw,noatime,compress=zstd:1,autodefrag,subvol=root   0 0
/dev/nvme0n1p3   /home           btrfs   rw,noatime,compress=zstd:1,autodefrag,subvol=home   0 0
/dev/nvme0n1p3   /srv            btrfs   rw,noatime,compress=zstd:1,autodefrag,subvol=srv    0 0
/dev/nvme0n1p3   /var            btrfs   rw,noatime,compress=zstd:1,autodefrag,subvol=var    0 0
/dev/nvme0n1p2   /boot           btrfs   rw,noatime                                          1 2
/dev/nvme0n1p1   /boot/efi       btrfs   noauto,noatime   

```

# Pasando a systemd
```
systemd-firstboot --prompt --setup-machine-id
```

## Configurar red
```
emerge --ask --noreplace net-misc/netifrc
emerge --ask net-misc/dhcpcd
```

## Red
```
systemctl enable --now dhcpcd
```

## Damos un password root
```
passwd (una robusta)
```

## Finalizamos config de systemd
```
systemctl preset-all --preset-mode=enable-only
```
## Indexar archivos
```
emerge --ask sys-apps/mlocate
```

## Sincronizacion temporal
```
emerge --ask net-misc/chrony
systemctl enable chronyd
```

## Herramientas  del sistema de archivos
```
emerge --ask sys-fs/btrfs-progs
emerge --ask sys-block/io-scheduler-udev-rules
```

## Herramientas de red inalambrica
```
emerge --ask net-wireless/iw net-wireless/wpa_supplicant
emerge --ask net-misc/networkmanager
systemctl start NetworkManager
systemctl enable NetworkManager
```

# Seleccionar gestor de arranque
```
echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf
emerge --ask sys-boot/grub
```

## Instalacion
```
grub-install --target=x86_64-efi --efi-directory=/boot
```

## Configuracion
```
grub-mkconfig -o /boot/grub/grub.cfg
(se supnone que no debe mostrar ningun error siguiendo estos pasos)
```

# Reiniciar y arrancar el nuevo sistema
```
exit
cd
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
reboot
```
Ya de aqui toca logearse con el nombre que se dio al host y la password que se vinculo
