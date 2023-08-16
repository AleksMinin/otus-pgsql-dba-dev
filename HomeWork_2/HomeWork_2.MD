# **Домашнее задание №2** 
## Установка и настройка PostgteSQL в контейнере Docker

**Цель:**
* Установить PostgreSQL в Docker контейнере
* Настроить контейнер для внешнего подключения

**Описание/Пошаговая инструкция выполнения домашнего задания/Выполнение:**

1. Создать ВМ с Ubuntu 20.04/22.04 или развернуть докер любым удобным способом
   * Создал ВМ Ubuntu 20.04 на YC
1. Поставить на нем Docker Engine
   * Набором команд произвел скачивание и установку Docker.
	```bash
	curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER && newgrp docker

	....
	
	 Client: Docker Engine - Community
	 Version:           24.0.5
	 API version:       1.43
	 Go version:        go1.20.6
	 Git commit:        ced0996
	 Built:             Fri Jul 21 20:35:18 2023
	 OS/Arch:           linux/amd64
	 Context:           default

	Server: Docker Engine - Community
	 Engine:
	  Version:          24.0.5
	  API version:      1.43 (minimum version 1.12)
	  Go version:       go1.20.6
	  Git commit:       a61e2b4
	  Built:            Fri Jul 21 20:35:18 2023
	  OS/Arch:          linux/amd64
	  Experimental:     false
	 containerd:
	  Version:          1.6.22
	  GitCommit:        8165feabfdfe38c65b599c4993d227328c231fca
	 runc:
	  Version:          1.1.8
	  GitCommit:        v1.1.8-0-g82f18fe
	 docker-init:
	  Version:          0.19.0
	  GitCommit:        de40ad0
	```  
	
	* Запустил Docker-сеть 
	```bash
	sudo docker network create pg-net
	```
1. Cделать каталог /var/lib/postgres и развернуть контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql
	```bash
	sudo mkdir /var/lib/postgres
	
	sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15	
	```

1. Развернуть контейнер с клиентом postgres
1. Подключиться из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
	```bash
	sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
	
	create database test;
	\c test
	test=# insert into ttest(nval) values (11),(12),(13);
	INSERT 0 3
	test=# select * from ttest;
	 id | nval
	----+------
	  1 |   11
	  2 |   12
	  3 |   13
	(3 rows)
	```
1. Подключиться к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/места установки докера
   * Запустил wsl на домашнем компьютере, установил postgres-client-15.
   
		Попробовал подключиться  извне к контейнеру на ВМ на YC
	```bash
	aminin@DESKTOP-BH4I2S2:/var/lib$ psql -p 5432 -U postgres -h 158.160.75.140 -d test -W
	Password:
	psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
	Type "help" for help.

	test=# select * from ttest;
	 id | nval
	----+------
	  1 |   11
	  2 |   12
	  3 |   13
	(3 rows)

	```
   
1. Удалить контейнер с сервером
	* Проверил статус контейнера **_sudo docker ps -a_** 
	* Остановил контейнер **_sudo docker stop e6cf2e40cf18_**
	* Уддалил его **_sudo docker rm e6cf2e40cf18_**
	```bash
	aminin@epd9l1nsae17nnam6os7:~$ sudo docker ps -a
	CONTAINER ID   IMAGE         COMMAND                  CREATED       STATUS       PORTS                                       NAMES
	e6cf2e40cf18   postgres:15   "docker-entrypoint.s…"   2 hours ago   Up 2 hours   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
	aminin@epd9l1nsae17nnam6os7:~$ sudo docker stop e6cf2e40cf18
	e6cf2e40cf18
	aminin@epd9l1nsae17nnam6os7:~$ sudo docker rm e6cf2e40cf18
	e6cf2e40cf18
	aminin@epd9l1nsae17nnam6os7:~$ sudo docker ps -a
	CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
	
	```

1. Создать контейнер заново.
	* Создал контейнер той же командой, что и в первый раз **_sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15_**
	* Подключился к Postgres в контейнере и вывел список баз, база test на месте, т.к. при удалении контейнара данные остались храниться на машине - хосте, в которой разворачиваем Docker.
	```bash
	aminin@epd9l1nsae17nnam6os7:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
	0e8b56e5a21875733b6c2e117f142f1c87837bf77dc77506b2a0d20ac0dfb69b
	aminin@epd9l1nsae17nnam6os7:~$ sudo docker ps -a
	CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
	0e8b56e5a218   postgres:15   "docker-entrypoint.s…"   19 seconds ago   Up 17 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
	aminin@epd9l1nsae17nnam6os7:~$ sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
	Password for user postgres:
	psql (15.4 (Debian 15.4-1.pgdg120+1))
	Type "help" for help.

	postgres=# \l
													List of databases
	   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
	-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
	 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
	 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
			   |          |          |            |            |            |                 | postgres=CTc/postgres
	 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
			   |          |          |            |            |            |                 | postgres=CTc/postgres
	 test      | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
	(4 rows)
	```

  * Проблем с разворачиванием docker, на удивление, не было. Ставил по примерам из лекции и читал статьи.
  
