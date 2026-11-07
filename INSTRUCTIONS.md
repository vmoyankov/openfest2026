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
