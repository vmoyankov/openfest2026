# Какво ще построим

Един скрипт на bash който да изпълнява основни функции в cloud за виртуални
машини или Virtual Private Servers (VPS).

С поетапното създаване на този скрипт ще видим кои са основните компоненти в
един cloud и как те си взаимодействат. Ще видим основните инструменти в Linux
които изграждан всички cloud платформи - от малките частни облаци съставени от
2 или 3 сървъра до hyperscalers.

Ето как ще изглежда скрипта в края на тази работилница:

```
	Usage: cloud <command> [args]

	Commands:
	  create   <vm_name>          Create a VM based on the CONFIG, images, and flavor
	  start    <vm_name> [host]   Start the VM on the given host, or on the
	                              host with the most free memory
	  stop     <vm_name>          Power off the VM on whichever host it is running on
	  shutdown <vm_name>          Ask the guest OS to shut down the VM
	  migrate  <vm_name> <host>   Live migrate the VM to the given host
	  console  <vm_name>          Connect to the VM's serial console (Ctrl+] exits)
	  list                        List running VMs on all hosts
```


# Инфраструктура

Всеки участник разполага със среда съставена от 4 машини:

- 1 x controller
- 2 x KVM hosts (хипервайзори), където се изпълняват виртуалните машини
- 1 x NFS server - осигурява споделен сторидж да хипервайзорите, където се
  съхраняват дисковите имиджи на виртуалните машини.


# Достъп

Само ``controller`` разполага с публичен IP адрес. Достъпа до всички останали машини от средата е само през тази машина.

Влезте с:

```
ssh root@<controller IP address>
```

Използвайте IP адреса и паролата която всеки участник ще получи.

След като вече сте в ``controller``, може да достъпите и останалите 3 машини от
вашата среда с:

```
ssh kvm1
ssh kvm2
ssh nfs
```

Достъпа по тях е по име. Във всички среди имената са еднакви, но адресите за
всяка среда са различни. Винаги използвайте само имената. Ssh автентикацията е
без парола от ``controller``.


# Файлова структура

##  На controller

```
  /root/cloud
          ├── bin
          │   └── cloud
          ├── config
          │   └── kvm-hosts
          ├── flavors
          │   └── simple
          │       └── domain.xml
          └── vms
              ├── vm1
              │   └── CONFIG
              └── vm2
                  ├── CONFIG
                  └──  SEED_VALUES
  /storage/nfs  (mounted from the nfs server)
```

## На KVM хостовете

```
  /storage/nfs  (mounted from the nfs server)
            ├── images
            │   ├── alma10.qcow2
            │   ├── alpine.qcow2
            │   └── debian-13.qcow2
            └── vms
                ├── vm1
                │   ├── disk_0.img
                ├── vm2
                │   ├── disk_0.img
                │   ├── disk_1.img
                │   └── seed.img
                └── vm3
                    ├── disk_0.img
                    └── seed.img
```


[!NOTE]
``/storage/nfs`` е монтирана NFS файлове система която е обща за KVM хостовете и контролера.


-----------------------------------------------------------

# Подготовка на хостовете

##  Controller

### Монтирайте NFS

```
apt update
apt install -y nfs-common
mkdir -p /storage/nfs
echo "nfs:/srv/nfs  /storage/nfs  nfs  vers=4.2,hard,noatime,nosuid,nodev,noexec,_netdev  0  0 " >> /etc/fstab
systemctl daemon-reload
mount -a
```

Проверете дали всичко е ОК с:

```
df -h
```

### Създаване на директориите

```
mkdir /storage/nfs/images
mkdir /storage/nfs/vms

mkdir -p cloud/{bin,config,flavors,vms}

echo -e "kvm1\nkvm2" > /root/cloud/config/kvm-hosts
```

## KVM Хостове

На всеки от KVM хостовете - kvm1, kvm2 изпълнете:

```
apt update
apt install -y nfs-common
mkdir -p /storage/nfs
echo "nfs:/srv/nfs  /storage/nfs  nfs  vers=4.2,hard,noatime,nosuid,nodev,noexec,_netdev  0  0 " >> /etc/fstab
systemctl daemon-reload
mount -a
```

Проверете дали всичко е ОК с:

```
df -h
```

-----------------------------------------------------------

# Първа минимална версия

## Инсталиране на необходимите пакети

### Controller

```
apt install -y libvirt-clients mtools tree
```

### KVM Хостове

```
apt install libvirt-daemon libvirt-clients
```

## Конфигурационни файлове

Разгледайте конфигурационните файлове:

- ``cloud/config/kvm-hosts`` - списък с KVM хостовете
- ``cloud/flavors/simple/domain.xml`` - Дефиниция на тип (flavor) виртуална машина

## cloud скрипт

Разгледайте първата версия на ``bin/cloud`` скрипта.

## Изтеглете AlpineLinux дисков image

Този имидж ще го използваме за базов за първите виртуални машини които създаваме. Изтеглете Alpine за x86_64, вид cloudinit. Базовите имиджи се съхраняват на споделения сторидж, така че може да изпълните операцията от контролера или който и да е KVM хост:

```
wget -O /storage/nfs/images/alpine.qcow2 'https://dl-cdn.alpinelinux.org/alpine/v3.24/releases/cloud/alpine-3.24.4-x86_64-cloudinit-r0.qcow2'
```

## Създаване на виртуална машина

Редактирайте параметрите на виртуаланта машина - ``cloud/vms/vm1/CONFIG``

Създайте виртулата машина

```
# bin/cloud create vm1
Formatting '/storage/nfs/vms/vm1/disk_0.img', fmt=qcow2 cluster_size=65536 extended_l2=off compression_type=zlib size=2147483648 backing_file=/storage/nfs/images/alpine.qcow2 backing_fmt=qcow2 lazy_refcounts=off refcount_bits=16
```

Това създава дисков имидж базиран на дефинирания в конфигурацията имидж
(alpine.qcow2), както и дефиниция на виртуалната машина базирана на избрания
flavor (`simple`).

```
# ls -lh /storage/nfs/vms/vm1/
# qemu-img info /storage/nfs/vms/vm1/disk_0.img 
# cat /root/cloud/vms/vm1/domain.xml
```

## Стартиране на виртуалната машина

```
# bin/cloud start vm1 kvm1
Domain 'vm1' created from /root/cloud/vms/vm1/domain.xml
```

Можете да видите конзолата като изпълните

```
# virsh -c qemu+ssh://kvm1/system console vm1
Connected to domain 'vm1'
Escape character is ^] (Ctrl + ])
```

## Спиране на виртуалната машина

```
# bin/cloud stop vm1 kvm1
Domain 'vm1' destroyed
```


-----------------------------------------------------------

# Network

Ще добавим виртуални мрежи на хостовете и във виртуалните машини.

## KVM хостове


За виртуалните мрежи ще използваме Linux bridge. Всеки KVM хост трябва да има
дефиниран по един `bridge` инрефейс за всяка виртуална мрежа, към която
вируалните машини може да се свързват. 

Ще създадем 2 виртуални мрежи - ``guest`` и ``priv``. Те ще бъдат свързани
съответно към ``eth1`` и ``eth2`` и на двата KVM хоста.

Копирайте файловете от ``files/netplan/*.yaml`` на всички KVM хостове в
``/etc/netplan/`` и приложете конфигурацията с:

```
# netplan try
```

Проверете резултата с:

```
# bridge link
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master guest state forwarding priority 32 cost 100 
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master priv state forwarding priority 32 cost 100 
```

## VM дефиниция

Разгледайте дефиницията на новия flavor - ``flavors/simple-net/domain.xml`` и
промените в параметрите на ``vm1``:

## Промяна на хонфигурацията на VM

- Спрете VM ако все още работи.
- изтрийте диска: ``rm /storage/nfs/vms/vm1/disk_0.img``
- регенрирайте конфигурацията и диска с ``bin/cloud create vm1``

```
# cat vms/vm1/domain.xml
```

Стартирайте VM:

```
bin/cloud start vm1 kvm1
```

Проверка на мрежата на kvm1:

```
root@kvm1:~# bridge link
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master guest state forwarding priority 32 cost 100 
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master priv state forwarding priority 32 cost 100 
8: vnet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master guest state forwarding priority 32 cost 2 
```


-----------------------------------------------------------

# Multiple Disks and Networks

Вижте новите темплейти ``flavors/multi/dimain.xml`` и елементите ``disk.tmpl``
и ``interface.tmpl``. 

Kонфигурацията на VM вече позволява дефиниране ня няколко диска и мрежови
интерфейси, които се описват като масиви. Виж ``vms/vm2/CONFIG``. Виж функциите
``include_disks()`` и ``include_interfaces()``.

Друга нова функция е създаването на дискове. Ако се подаде имидж, той се
клонира. Ако не се подаде, се създава празен диск. Виж функцията
``create_disks()``

Друга нова функция е генериране на постоянно UUID ако няма такова.

## Създай VM

```
# bin/cloud create vm2
```
Провери резултата:

```
# cat vms/vm2/domain.xml
```

## Стартирай VM

```
# bin/cloud start vm2 kvm1
Domain 'vm2' created from /root/cloud/vms/vm2/domain.xml
```

-----------------------------------------------------------

# Автоматичен избор на Хост

Нови функции:

- Хоста може да се избира автоматично според натоварването.
- Намиране на работещите виртуални машини
- Серийна конзола от контролера
- Списък на работещите виртуални машини


-----------------------------------------------------------

# Live Migration

```
# bin/cloud list
kvm1	vm1
kvm2	vm2

# bin/cloud migrate vm1 kvm2

# bin/cloud list
kvm2	vm2
kvm2	vm1
```

-----------------------------------------------------------

# Cloud-init

Можете да контекстуализирате виртуалните машини със специфични настройки на guest OS, като:

- hostname
- създаване на user
- password
- ssh public key authentication
- IP адрес, DNS

Виж flavor ``full`` , ``cloud/ci/`` и ``vm3``. Поставете ваш ssh key в ``SEED_VALUES``

Изтеглете AlmaLinux 10 имидж:

```
# wget -O /storage/nfs/images/alma10.qcow2 https://repo.almalinux.org/almalinux/10/cloud/
x86_64/images/AlmaLinux-10-GenericCloud-latest.x86_64.qcow2
```

Тествайте ``vm3``:

```
# bin/cloud create vm3
Formatting '/storage/nfs/vms/vm3/disk_0.img', fmt=qcow2 cluster_size=65536 extended_l2=off compression_type=zlib size=16106127360 backing_file=/storage/nfs/images/alma10.qcow2 backing_fmt=qcow2 lazy_refcounts=off refcount_bits=16

# ls -l /storage/nfs/vms/vm3/
total 220
-rw-r--r-- 1 64055 kvm  196848 Nov  7 20:58 disk_0.img
-rw-r--r-- 1 64055 kvm 2097152 Nov  7 20:59 seed.img

root@controller:~/cloud# bin/cloud start vm3
Domain 'vm3' created from /root/cloud/vms/vm3/domain.xml

root@controller:~/cloud# bin/cloud list vm3
kvm1	vm3
kvm2	vm2
kvm2	vm1
root@controller:~/cloud# bin/cloud console vm3
Connected to domain 'vm3'
Escape character is ^] (Ctrl + ])

vm3 login: 
vm3 login: root
Password: 
[root@vm3 ~]# 
```


