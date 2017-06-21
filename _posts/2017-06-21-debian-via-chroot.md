---
published: true
---
## Использование Debian на роутере

<p class="message">
В итоге будет получена полноценная среда Debian, в которой не будет ни одного файла выше самой chroot-среды, т.е. сущность из busybox'а с аплетом chroot и костыли для исполнения скриптов в __/opt/etc/ndm/*.d__, которые ранее были "внешними" по отношению к Debian'у больше не будут нужны вовсе.
</p>

На профильном форуме уже есть схожая [тема](https://forum.keenetic.net/topic/458-debian-stable/), которая является адаптацией [этого](https://github.com/DontBeAPadavan/chroot-debian) решения, но кинетике это можно сделать куда изящнее.

## Подготовка на ПК

На Debian-машине подготовливаем архив с полуфабрикатом:
```
#!/bin/sh

DEB_FOLDER=debian

### Are we root or mere mortal?
if [ $(id -u) -ne 0 ] ; then
    echo 'Please run it as root'
    exit 1
fi

### First stage of debotsrap
#sudo debootstrap --arch mipsel --foreign --variant=minbase --include=openssh-server stable debian ftp://ftp.debian.org/$DEB_FOLDER

### Make f\w hook dirs, see https://github.com/ndmsystems/packages/wiki/Opkg-Component for details
for folder in button fs netfilter schedule time usb user wan; do
    mkdir -p $DEB_FOLDER/etc/ndm/${folder}.d
done

### Place default evnvironment vars to /etc/ndm/env_vars.sh
cat > $DEB_FOLDER/etc/ndm/env_vars.sh << EOF

### Unset NDMS-specific env vars
#unset HOSTNAME
#unset IFS
unset LD_BIND_NOW
unset LD_LIBRARY_PATH
#unset PS1
#unset PS2
#unset PS4
#unset SHLVL
#unset timezone

### Set Debian-speceific env vars
export LANG=en_US.UTF-8
export TERM=xterm
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export SHELL=/bin/bash
export MAIL=/var/mail/root
export LOGNAME=root
export USER=root
export USERNAME=root
export HOME=/root

### Fix broken vars
[ "$PWD" = "(unreachable)/" ] && export PWD=/
EOF

### Make start script for chroot'ed daemons
cat > $DEB_FOLDER/etc/ndm/initrc.sh << EOF
#!/bin/sh

### Fix env vars
. /etc/ndm/env_vars.sh

### Is /etc/ndm/services.txt exists?
CHROOT_SERVICES_LIST=/etc/ndm/services.txt
if [ ! -e "\$CHROOT_SERVICES_LIST" ]; then
    echo ssh > \$CHROOT_SERVICES_LIST
fi

### Start/Stop services
for item in \$(cat \$CHROOT_SERVICES_LIST); do
    /etc/init.d/\$item \$1
done
EOF
chmod +x $DEB_FOLDER/etc/ndm/initrc.sh

### The second deboostrap stage should be done on router
tar -cvzf debian_clean.tgz $DEB_FOLDER
```


Проба тега `/opt/etc`, который коряво отображает prose.io
