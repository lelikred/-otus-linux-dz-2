## 1. Работа с репозиторием

**Делаем форк репозитория erlong15/otus-linux  
Клонируем репозиторий, копируем Vagrantfile в свою рабочую папку**  
```
mkdir /home/user/git_repo/otus-linux-dz-2
git clone git@github.com:lelikred/otus-linux.git
git clone git@github.com:lelikred/otus-linux-dz-2.git
cd /home/user/git_repo/otus-linux/
cp Vagrantfile /home/user/git_repo/otus-linux-dz-2/
```
## 2. Создаем скрипт для создания RAID

`vi create-RAID-GPT.sh`  
```
#!/bin/bash

sudo su

# create RAID5
mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}
mdadm --create --verbose /dev/md0 -l 5 -n 5 /dev/sd{b,c,d,e,f}

# create and write mdadm.conf

mkdir /etc/mdadm/
echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf

# greate GPT and 5 partitions

parted -s /dev/md0 mklabel gpt
parted /dev/md0 mkpart primary ext4 0% 20%
parted /dev/md0 mkpart primary ext4 20% 40%
parted /dev/md0 mkpart primary ext4 40% 60%
parted /dev/md0 mkpart primary ext4 60% 80%
parted /dev/md0 mkpart primary ext4 80% 100%

# for each partition create fs and mount

for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
mkdir -p /raid/part{1,2,3,4,5}
for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done
```

## 3. Редактируем Vagrantfile

**В блок MACHINES добавляем конфирурации дополнительных дисков**
```
:sata5 => {
    :dfile => './sata5.vdi',
    :size => 250, # Megabytes
    :port => 5
```
**В блок Vagrant.configure в конце добавляем строчку, для использования ранее созданного нами скрипта при загрузке**

`box.vm.provision "shell", path: "create-RAID-GPT.sh"`

## 4. Развернем машину, проверим, что партиции и RAID создались

```
mdadm -D /dev/md0
lsblk
```

**Сломаем/починим рейд**
```
mdadm /dev/md0 --fail /dev/sde
cat /proc/mdstat
```
**Удалим Диск из массива и добавим его**
```
mdadm /dev/md0 --remove /dev/sde
lsblk
mdadm /dev/md0 --add /dev/sde
lsblk
```
**Проверим вывод следующих команд, учитывая маленький объем rebuid происходит очень быстро.**

**cat /proc/mdstat**
```
Personalities : [raid6] [raid5] [raid4]
md0 : active raid5 sde[6] sdf[5] sdd[2] sdc[1] sdb[0]
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/5] [UUUUU]
```

**mdadm -D /dev/md0**
```
    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       6       8       64        3      active sync   /dev/sde
       5       8       80        4      active sync   /dev/sdf
```       
