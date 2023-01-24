# Практические навыки работы с ZFS
1) Определить алгоритм с наилучшим сжатием
2) Определить настройки пула
3) Работа со снапшотами

#### 1 Подключаемся по ssh к вертуальной машине запущеной по Vagrantfile с Bash-скриптом, который конфигурирует сервер с ZFS

Из 8 дисков создаем 4 пула с RAID1 
```
zpool create otus1 mirror /dev/sdb /dev/sdc
zpool create otus2 mirror /dev/sdd /dev/sde
zpool create otus3 mirror /dev/sdf /dev/sdg
zpool create otus4 mirror /dev/sdh /dev/sdi
 ```
и добавляем на каждый пул свой алгоритм сжатия
```
 zfs set compression=lzjb otus1
 zfs set compression=lz4 otus2
 zfs set compression=gzip-9 otus3
 zfs set compression=zle otus4
```
Скачиваем один и тот же файл во все пулы и проверяем степень сжатия. 
```
zfs list
NAME    USED  AVAIL     REFER  MOUNTPOINT
otus1  21.6M   330M     21.5M  /otus1
otus2  17.7M   334M     17.6M  /otus2
otus3  10.8M   341M     10.7M  /otus3  > наибольшая степень сжатия
otus4  39.0M   313M     38.9M  /otus4
```

#### 2 Скачиваем архив в домашний каталог командой 
```
wget -O archive.tar.gz --no-check-certificate https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg/zfs_task1.tar.gz&export=download
```
И разархивируем его. 
```
[root@zfs ~]# tar -xzvf archive.tar.gz
zpoolexport/
zpoolexport/filea
zpoolexport/fileb
```
Проверяем, возможность импорта данного каталога в пул
```
[root@zfs ~]# zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

	otus                         ONLINE
	  mirror-0                   ONLINE
	    /root/zpoolexport/filea  ONLINE
	    /root/zpoolexport/fileb  ONLINE
```
И выполняем импорт
```
[root@zfs ~]# zpool import -d zpoolexport/ otus
```
Получаем 
```
[root@zfs ~]# zpool status
  pool: otus
 state: ONLINE
  scan: none requested
config:

	NAME                         STATE     READ WRITE CKSUM
	otus                         ONLINE       0     0     0
	  mirror-0                   ONLINE       0     0     0
	    /root/zpoolexport/filea  ONLINE       0     0     0
	    /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors

  pool: otus1
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus1       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus2       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdd     ONLINE       0     0     0
	    sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus3       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdf     ONLINE       0     0     0
	    sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus4       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdh     ONLINE       0     0     0
	    sdi     ONLINE       0     0     0

errors: No known data errors
```
Выполняем запрос всех параметров файловой системы командой
```
[root@zfs ~]# zfs get all otus
```
командой grep можно уточнить конкретный параметр
```
[root@zfs ~]# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
[root@zfs ~]# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default
[root@zfs ~]# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local
[root@zfs ~]# zfs get compression otus
NAME  PROPERTY     VALUE     SOURCE
otus  compression  zle       local
[root@zfs ~]# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
```

#### 2 Скачиваем архив по ссылке
```
[root@zfs ~]# wget -O otus_task2.file --no-check-certificate https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download
[1] 3217
```
восстанавливаем ФС из снапшота
```
[root@zfs ~]# zfs receive otus/test@today < otus_task2.file
```
ищем в каталоге /otus/test файл с именем “secret_message” и смотрим содиржимое
```
[root@zfs ~]# find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message
[root@zfs ~]# cat /otus/test/task1/file_mess/secret_message
https://github.com/sindresorhus/awesome              => переходим по ссылке в репозиторий на github
```
