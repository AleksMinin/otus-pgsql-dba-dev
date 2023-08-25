# Домашнее задание №3
## Установка и настройка PostgreSQL

Цель:
создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
переносить содержимое базы данных PostgreSQL на дополнительный диск
переносить содержимое БД PostgreSQL между виртуальными машинами

Описание/Пошаговая инструкция выполнения домашнего задания:

1. Создайте виртуальную машину c Ubuntu 20.04/22.04 LTS в GCE/ЯО/Virtual Box/докере

1. Поставьте на нее PostgreSQL 15 через sudo apt

1. Проверьте что кластер запущен через sudo -u postgres pg_lsclusters

	```bash
	ubuntu@vm-ubuntu:~$ sudo -u postgres pg_lsclusters
	Ver Cluster Port Status Owner    Data directory              Log file
	15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
	```

1. Зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
	```sql
	postgres=# create table test(c1 text);
	postgres=# insert into test values('1');
	postgres=# select * from test;
	 c1
	----
	 1
	(1 row)
	```
1. Остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop
1. Создайте новый диск к ВМ размером 10GB
1. Добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk

1. Проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux

	```bash
	ubuntu@vm-ubuntu:~$ sudo parted -l |grep Error
	Error: /dev/vdb: unrecognised disk label

	ubuntu@vm-ubuntu:~$ lsblk
	NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
	vda    252:0    0    15G  0 disk
	├─vda1 252:1    0     1M  0 part
	└─vda2 252:2    0    15G  0 part /
	vdb    252:16   0    10G  0 disk

	ubuntu@vm-ubuntu:~$ sudo parted /dev/vdb mklabel gpt
	Information: You may need to update /etc/fstab.

	ubuntu@vm-ubuntu:~$ sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%
	Information: You may need to update /etc/fstab.

	ubuntu@vm-ubuntu:~$ lsblk
	NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
	vda    252:0    0    15G  0 disk
	├─vda1 252:1    0     1M  0 part
	└─vda2 252:2    0    15G  0 part /
	vdb    252:16   0    10G  0 disk
	└─vdb1 252:17   0    10G  0 part
	```

	* Создание файловой системы с меткой test
	```bash
	ubuntu@vm-ubuntu:~$ sudo mkfs.ext4 -L test /dev/vdb1
	ubuntu@vm-ubuntu:~$ lsblk --fs
	vda
	├─vda1
	└─vda2 ext4     1.0         ed465c6e-049a-41c6-8e0b-c8da348a3577    9.6G    30% /
	vdb
	└─vdb1 ext4     1.0   test  34f7092f-23eb-4952-8994-2aa77c941b91	
	
	ubuntu@vm-ubuntu:~$ sudo mkdir -p /mnt/data
	ubuntu@vm-ubuntu:~$ sudo mount -o defaults /dev/vdb1 /mnt/data
	
	```'
1. Перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
	```bash
	ubuntu@vm-ubuntu:~$ lsblk --fs
	NAME   FSTYPE   FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
	vda
	├─vda1
	└─vda2 ext4     1.0         ed465c6e-049a-41c6-8e0b-c8da348a3577    9.6G    30% /
	vdb
	└─vdb1 ext4     1.0   test  34f7092f-23eb-4952-8994-2aa77c941b91    9.4G     0% /mnt/data
	```	
1. Сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
	```
	ubuntu@vm-ubuntu:/mnt/data$ sudo chown -R postgres:postgres /mnt/data/
	```
1. Перенесите содержимое /var/lib/postgres/15 в /mnt/data - mv /var/lib/postgresql/15/mnt/data
   ```bash
   postgres@vm-ubuntu:/mnt/data$ mv /var/lib/postgresql/15/ .
   ```
1. Попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start.  Напишите получилось или нет и почему
   ```bash
	ubuntu@vm-ubuntu:~$ sudo -u postgres pg_ctlcluster 15 main start
	Error: /var/lib/postgresql/15/main is not accessible or does not exist
   ```
   Директория отсутствует в указанном месте, PG не запуcтился.
   
1. Задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который надо поменять и поменяйте его
	1. Напишите что и почему поменяли
	* В файле postgres.conf заменил значение пути в параметре data_directory = '/mnt/data/15/main' #use data in another directory
	  Собственно, в коментарии и указано: Использовать данные в другом каталоге
1. Попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
   ```bash
   ubuntu@vm-ubuntu:~$ sudo -u postgres pg_ctlcluster 15 main start
	Warning: the cluster will not be running as a systemd service. Consider using systemctl:
	  sudo systemctl start postgresql@15-main
   ubuntu@vm-ubuntu:~$
   
   ubuntu@vm-ubuntu:~$ sudo -u postgres pg_ctlcluster 15 main status
   pg_ctl: server is running (PID: 2556)
   /usr/lib/postgresql/15/bin/postgres "-D" "/mnt/data/15/main" "-c" "config_file=/etc/postgresql/15/main/postgresql.conf"
   ```
1. Напишите получилось или нет и почему
	* Кластер запустился, т.к. я указал путь к данным кластера в конфигурационном файле.
1. Зайдите через через psql и проверьте содержимое ранее созданной таблицы
   ```sql
	ubuntu@vm-ubuntu:~$ sudo -u postgres psql
	psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
	Type "help" for help.
	
	postgres=# select * from test;
	 c1
	----
	 1
	(1 row)

	postgres=#
   ```
1. __Задание со звездочкой *__: не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.

	* Выключил инстанс ubuntu. 
	* Отсоединил добавленный диск от ubuntu в web-панели YC.
	* Создал еще один инстанс ubuntu2. 
	* Поставил PG по той же схеме, что и выше. 
	* Выключил ubuntu2. Зашел в диски ubuntu2 в web-панели YC и присоединил отсоединенный от vm ubuntu диск.
	* Включил ubuntu2. Проверил, появился ли диск, создал директорию /mnt/data и примонтировал его в /mnt/data
	
	```bash
	ubuntu@vm-ubuntu2:~$ lsblk
	NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
	vda    252:0    0    10G  0 disk
	├─vda1 252:1    0     1M  0 part
	└─vda2 252:2    0    10G  0 part /
	vdb    252:16   0    10G  0 disk
	└─vdb1 252:17   0    10G  0 part
	
	ubuntu@vm-ubuntu2:~$ sudo mkdir -p /mnt/data
	
	ubuntu@vm-ubuntu2:~$ sudo mount -o defaults /dev/vdb1 /mnt/data
	
	ubuntu@vm-ubuntu2:~$ df -h -x tmpfs
	Filesystem      Size  Used Avail Use% Mounted on
	/dev/vda2       9.8G  4.5G  4.9G  48% /
	/dev/vdb1       9.8G   39M  9.4G   1% /mnt/data
	```
    * Назначил пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
	* Проверил статус кластера и произвел остановку
	```bash
	ubuntu@vm-ubuntu2:~$ sudo -u postgres pg_ctlcluster 15 main status
	pg_ctl: server is running (PID: 37151)
	/usr/lib/postgresql/15/bin/postgres "-D" "/var/lib/postgresql/15/main" "-c" "config_file=/etc/postgresql/15/main/postgresql.conf"
	
	ubuntu@vm-ubuntu2:~$ sudo -u postgres pg_ctlcluster 15 main stop
	Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
		sudo systemctl stop postgresql@15-main
	```
	* Командой sudo rm -fr /var/lib/postgresql/15 удалил все данные в каталоге 15 и сам каталоге
	* Изменил значение пути в параметре data_directory = '/mnt/data/15/main'/ в файле etc/postgresql/15/main/postgres.conf

	```bash
	ubuntu@vm-ubuntu2:/etc/postgresql/15/main# sudo -u postgres pg_ctlcluster 15 main start
	Warning: the cluster will not be running as a systemd service. Consider using systemctl:
	  sudo systemctl start postgresql@15-main
	  
	ubuntu@vm-ubuntu2:/etc/postgresql/15/main# sudo -u postgres pg_ctlcluster 15 main status
	pg_ctl: server is running (PID: 37788)
	/usr/lib/postgresql/15/bin/postgres "-D" "/mnt/data/15/main" "-c" "config_file=/etc/postgresql/15/main/postgresql.conf"
	
	ubuntu@vm-ubuntu2:/etc/postgresql/15/main# sudo -u postgres psql
	psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
	Type "help" for help.

	postgres=# select * from test;
	 c1
	----
	 1
	(1 row)

	postgres=#
	```
	* Все получилось, перенос данных прошел без проблем.



























