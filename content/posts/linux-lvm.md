+++
author = "Yury Ozhegov"
title = "Работа с LVM в Linux: быстрый гайд"
date = "2025-08-19"
description = "Сегодня разберём, как управлять LVM в Linux."
tags = [
    "lvm",
    "lsblk",
	"mkfs",
]
categories = [
    "linux",
    "howto",
]
+++

Сегодня разберём, как управлять LVM в Linux.
<!--more-->

## 1️. Установка инструментов

Для начала давай проверим, что в системе присутствуют утилиты, и при необходимости установим их:
```shell
# apt install -y lvm2 parted
```

## 2️. Подготовка дисков

Добавим новые диски и проверим, что они видны в системе:
```shell
# lsblk
```

Пример вывода команды:
```shell
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sdb      8:16   0   10G  0 disk
sdc      8:32   0   10G  0 disk
Создадим разделы под LVM:
```shell
# parted /dev/sdb

(parted) mklabel gpt
(parted) mkpart primary 0% 100%
(parted) set 1 lvm on
(parted) q


## 3️. Создание группы томов

```shell
# vgcreate data-vg /dev/sdb1
# vgextend data-vg /dev/sdc1
```

Проверим:
```shell
# vgs

VG      #PV #LV #SN Attr   VSize  VFree
data-vg   2   0   0 wz--n- 19.99g 19.99g
```

## 4️. Создание логических томов

```shell
# lvcreate --size 8g --type linear -n data-lv1 data-vg
# lvcreate --size 8g --type linear -n data-lv2 data-vg
```

Проверим:
```shell
# lvs
```

LV       VG      Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
data-lv1 data-vg -wi-a----- 8.00g
data-lv2 data-vg -wi-a----- 8.00g
```

Кстати, символические ссылки на устройства появились по пути /dev/mapper:  data--vg-data--lv1 и data--vg-data--lv2

## 5. Создание файловой системы и монтирование дисков

```shell
# mkfs -t ext4 /dev/mapper/data--vg-data--lv1
# mkdir /mnt/data
# mount /dev/mapper/data--vg-data--lv1 /mnt/data/
# touch /mnt/data/hello
```

## 6. Управление томами

Допустим, что внезапно второй том стал не нужен, удалим его:
```shell
# lvremove data-vg/data-lv2
```

Выполним расширение первого тома с автоматическим увеличением файловой системы:
```shell
# lvresize -r -L +10G data-vg/data-lv1
```

Проверим результат:
```shell
# df -Th

Filesystem                     Type      Size  Used Avail Use% Mounted on
/dev/mapper/data--vg-data--lv1 ext4       18G  2.1M   17G   1% /mnt/data
```

## 7️. Просмотр физического распределения данных
```shell
dmsetup table
```

Пример вывода команды выше:
```shell
data--vg-data--lv1: 0 20963328 linear 8:17 2048
data--vg-data--lv1: 20963328 16785408 linear 8:33 2048
```

Здесь 8:17 и 8:33 — major:minor номера физических устройств, которые можно вычислить по выводу lsblk, а числа 0 20963328 — начальный сектор и количество секторов. 
Например, 20963328 * 512 байт это ~10 Гб. 
Видно, что 10 Гб размещены на sdb, а 8 Гб — на sdc.

## Итог

LVM — мощный инструмент для гибкого управления томами. 
С его помощью можно:
* динамически расширять тома,
* объединять диски в группы,
* безопасно перераспределять пространство.

Начни с тестовых дисков, попробуй создать группу томов и несколько логических томов,  Уже через пару команд ты почувствуешь всю силу LVM! 🚀














