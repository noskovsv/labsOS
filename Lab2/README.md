# Лабораторная работа №2
###  Выполнил: Носков Сергей Владимирович ББСО-01-18

### Задание 1 - Установка ОС, настройка LVM, RAID
По заданию я выбрал создаваемой виртуальной машине соответствующие характеристики в настройках:
| Параметр  | Значение  |
| ------------ | ------------ |
| Операционная система | Debian 10.4 |
|  Оперативная память | 1 GB  |
| Процессор  |  1 ядро |
| SSD  | 2 шт.  |
| Объём дисков  | 5 GB  |
| SATA контроллер  | 4 порта  |

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.1.jpg)

Далее я начал процесс установки по инструкции в пояснительной записке к этой Л/Р.

На следуещем скриншоте показано, что процесс разметки прошёл успешно.

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.2.jpg)

После того, как ОС загрузилась, я проверил её работоспособность. Всё оказалось в норме.
Команда `lsblk`  выдала красивую информацию о состоянии дисков. С помощью команды `dd if=/dev/sda1 of=/dev/sdb1` я копировал содержимое раздела /boot с диска sda (ssd1) на диск sdb (ssd2).  Выполнил установку загрузчика ОС на второй диск с помощью `grub-install /dev/sdb`. Теперь в случае отказа первого диска система сможет спокойно загрузится. 
C помощью команды `cat /proc/mdstat` я ознакомился с информацией о текущем состоянии raid, где md0 — имя RAID устройства; raid1 sda2[0] sdb2[1] — из каких дисков собран; 1046528 blocks — размер массива; [2/2] [UU] — количество юнитов, которые на данный момент используются.
- `pvs` - выводит информацию о физической памяти
- `lvs` - выводит информацию о Logical Volume
- `vgs` - выводит информацию о группе томов LVM

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.3.jpg)

**Благодаря данному зданию я научился:**
- **производить разметку дисков;**
- **создавать RAID;**
- **монтировать тома LVM;**
- **просматривать информацию о состояниях дисков и raid;**
- **устанавливать GRUB.**

------------

### Задание 2 - Эмуляция отказа одного из дисков

Команда просмотра состояния дисков `fdisk -l` после удаления диска ssd1 и перезагрузки  показала следующее:

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.4.jpg)

Просмотрев состояние raid, я убедился, что эмуляция отказа одного из дисков прошла успешно. 

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.5.jpg)

Чтобы решить проблему отказа одного диска, я создал новый диск.

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.6.jpg)

После перезагрузки система видит новый диск. Далее я скопировал таблицу разделов со старого диска на новый с помощью `sfdisk -d /dev/sda | sfdisk /dev/sdb`
и добавили новый диск в наш raid c помощью команды `mdadm --manage /dev/md0 --add /dev/sdb2`. Получили:

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.7.jpg)

После установки GRUB и копирования раздела /boot можно считать, что проблема отказа диска была полностью устранена.

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.8.jpg)

**В ходе выполнения второго задания в данной лабораторной работе я научился:**
- **проводить эмуляцию отказа одного из дисков;**
- **добавлять новый диск в систему и в raid;**
- **устранять проблему связанную с отказом одного из дисков в составе raid.**

------------

### Задание 3 - Добавление новых дисков и перенос раздела

Я проэмулировал отказ ssd2 . Получили:

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.3.1.jpg)

Затем я добавил новый диск большего объёма и проделал действия, которые были во втором задании (скопировал файловую таблицу со старого диска на новый с помощью команды `sfdisk -d /dev/sda | sfdisk /dev/sdb`;  скопировал данные /boot на новый диск, причём с помощью команды `mount -a` я перемонтировал /boot на действующий диск)

С помощью команды `mdadm --create --verbose /dev/md63 --force --level=1 --raid-devices=1 /dev/sdb2`  получилось создать новый рейд-массив с включением туда только нового ssd диска. Мы воспользуемся опцией `--level`, для того чтобы создать RAID-массив 1 уровня. А с помощью ключа `--raid-devices` укажем устройства, поверх которых будет собираться RAID-массив.

Справку об этой команде я нашёл тут: http://xgu.ru/wiki/mdadm

В итоге получили:

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.3.2.jpg)

Теперь пришло время настроить LVM. Я создал новый физический том, включив в него ранее созданный RAID массив с помощью команды `pvcreate /dev/md63`.
Вывод информации о физических томах до и после выполнения данной команды:

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.3.3.jpg)

Чтобы увеличеть размер Volume Group system, я использовал команду `vgextend system /dev/md63`. Затем я переместил данные со старого диска на новый с помощью команды `pvmove -i 10 -n /dev/system/* /dev/md0 /dev/md63` и получил: 

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.3.4.jpg)

Далее удаляем диск старого raid из Volume Group с помощью команды `vgreduce system /dev/md0`, перемонтруем /boot и добавляем новые диски ssd5, hdd1, hdd2. При помощи `fsdisk` меняем объём раздела первого и второго диска.
> Директория **/boot** cодержит важные файлы для процесса загрузки, включая ядро Linux

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.3.5.jpg)

Далее расширяем raid с помощью `pvresize /dev/md127`. 

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.3.6.jpg)

Добавим вновь появившееся место VG var,root.

Следующий этап. Создаём новый raid-массив, на HDD создаём логические тома и форматируем эти диски под ext4 с помощью `mkfs.ext4 /dev/mapper/data-var_log`.
> **Ext4** — популярная файловая системы в Linux.

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.3.7.jpg)

Теперь я перенёс /var/log (для этого мне пришлось монтировать временные хранилища логов , останавливать процессы и синхронизировать разделы).

После этого я поправил файл fstab

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.3.8.jpg)

Итоговый результат получился таким:

![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.3.9.jpg)
![](https://github.com/noskovsv/labsOS/blob/master/Screenshots/2.3.10.jpg)

**При выполнении задания №3 я научился:**
- **менять размеры raid;**
- **создавать новые raid массивы;**
- **производить разметку дисков прямо в терминале;**
- **форматировать диски в терминале.**