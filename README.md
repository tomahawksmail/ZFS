ZFS — Справочник команд #
Данный справочник является переводом данной статьи. Авторы перевода: Евгений Ратников и Сгибнев Михаил. Огромное им спасибо за проделанную работу!

Данная информация представлена в интернете на множестве ресурсов. Оригинальная статья оформлена в виде таблицы, я же оформлю ее в привычном для моего блога формате — в формате пошагового обучения.

В любом случае не забывайте про страницы справки по командам работы с ZFS.

man zpool
man zfs
Так как включить в пул (zpool) можно любые сущности файловой системы, автор приводит примеры построения пулов и работы с ними с использованием простых файлов. Итак, создадим несколько файлов, с которыми будем работать подобно дискам.

cd /
mkfile 100m disk1 disk2 disk3 disk5
mkfile 50m disk4
Мы создали 5 «виртуальных дисков». Четыре имею размер по 100 Мб, а один — 50 Мб. Это пригодится для демонстрации работы с устройствами (разделами) разной ёмкости.

Работа с пулом ZFS #
Теперь создадим простой пул без избыточности, затем проверим его размер и использование.

zpool create myzfs /disk1 /disk2
zpool list

NAME          SIZE    USED   AVAIL    CAP  HEALTH     ALTROOT
myzfs         191M     94K    191M     0%  ONLINE     -
Созданы пул автоматически монтируется в каталог /myzfs. Посмотрим более детальную информацию о нашем хранилище.

zpool status -v

  pool: myzfs
 state: ONLINE
 scrub: none requested
config:
        NAME        STATE     READ WRITE CKSUM
        myzfs       ONLINE       0     0     0
          /disk1    ONLINE       0     0     0
          /disk2    ONLINE       0     0     0

errors: No known data errors
Из вывода видно, что в пул включены два диска. Пул без избыточности (не mirror и не RAIDZ).

Теперь попробуем удалить только что созданный пул. Должны же мы это уметь.

zpool destroy myzfs
zpool list

no pools available
Попробуем снова создать пул типа MIRROR (зеркало), но на этот раз попытаемся включить в него диски разного размера. Zpool не даст нам этого сделать. Чтобы безоговорочно создать такой пул, используйте опцию -f, но в этом случае помните — размер зеркала будет равен объему наименьшего диска.

zpool create myzfs mirror /disk1 /disk4

invalid vdev specification
use '-f' to override the following errors:
mirror contains devices of different sizes
Создать зеркалируемое (MIRROR) хранилище можно на двух и более устройствах. Сколько устройств в пуле типа MIRROR — столько у нас есть одинаковых копий данных.

zpool create myzfs mirror /disk1 /disk2 /disk3
zpool list
NAME          SIZE    USED   AVAIL    CAP  HEALTH     ALTROOT
myzfs        95.5M    112K   95.4M     0%  ONLINE     -

zpool status -v
  pool: myzfs
 state: ONLINE
 scrub: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       ONLINE       0     0     0
          mirror    ONLINE       0     0     0
            /disk1  ONLINE       0     0     0
            /disk2  ONLINE       0     0     0
            /disk3  ONLINE       0     0     0

errors: No known data errors
Вместо зеркалирования можно использовать массивы RAID. Для этого необходимо создавать пул типа raidz вместо mirror. Подробнее в хендбуке.

Давайте теперь исключим один из дисков из пула. Так как этот диск относится к зеркалу (MIRROR), то при его исключении никаких проблем не возникает.

zpool detach myzfs /disk3
zpool status -v

  pool: myzfs
 state: ONLINE
 scrub: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       ONLINE       0     0     0
          mirror    ONLINE       0     0     0
            /disk1  ONLINE       0     0     0
            /disk2  ONLINE       0     0     0

errors: No known data errors
Теперь давайте добавим к пулу новый диск. Если пул не был зеркальным, то он им станет после добавления нового диска.

zpool attach myzfs /disk1 /disk3
zpool status -v

  pool: myzfs
 state: ONLINE
 scrub: resilver completed with 0 errors on Tue Sep 11 13:31:49 2007
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       ONLINE       0     0     0
          mirror    ONLINE       0     0     0
            /disk1  ONLINE       0     0     0
            /disk2  ONLINE       0     0     0
            /disk3  ONLINE       0     0     0

errors: No known data errors
А что будет, если попытаемся удалить, а не исключить устройство из пула? Zpool сообщит нам о том, что устройство не может быть удалено. Для начала его нужно отключить.

zpool remove myzfs /disk3

cannot remove /disk3: only inactive hot spares can be removed

zpool detach myzfs /disk3
Теперь давайте попробуем добавить диск горячей замены (hot spare) к нашему пулу.

zpool add myzfs spare /disk3
zpool status -v

  pool: myzfs
 state: ONLINE
 scrub: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       ONLINE       0     0     0
          mirror    ONLINE       0     0     0
            /disk1  ONLINE       0     0     0
            /disk2  ONLINE       0     0     0
        spares
          /disk3    AVAIL   

errors: No known data errors
А теперь удалим его из пула.

zpool remove myzfs /disk3
zpool status -v

  pool: myzfs
 state: ONLINE
 scrub: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       ONLINE       0     0     0
          mirror    ONLINE       0     0     0
            /disk1  ONLINE       0     0     0
            /disk2  ONLINE       0     0     0

errors: No known data errors
Теперь попробуем отключить один из дисков. Пока диск отключен, на него не будет производиться запись и с него не будет производиться чтение. Если использовать параметр -t, то при перезагрузке сервера диск вернется в состояние онлайн автоматически.

zpool offline myzfs /disk1
zpool status -v

  pool: myzfs
 state: DEGRADED
status: One or more devices has been taken offline by the administrator.
        Sufficient replicas exist for the pool to continue functioning 
        in a degraded state.
action: Online the device using 'zpool online' or replace the device 
        with 'zpool replace'.
 scrub: resilver completed with 0 errors on Tue Sep 11 13:39:25 2007
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       DEGRADED     0     0     0
          mirror    DEGRADED     0     0     0
            /disk1  OFFLINE      0     0     0
            /disk2  ONLINE       0     0     0

errors: No known data errors
Обратите внимание на состояние пула: DEGRADED

Теперь включим этот диск.

zpool online myzfs /disk1
zpool status -v

  pool: myzfs
 state: ONLINE
 scrub: resilver completed with 0 errors on Tue Sep 11 13:47:14 2007
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       ONLINE       0     0     0
          mirror    ONLINE       0     0     0
            /disk1  ONLINE       0     0     0
            /disk2  ONLINE       0     0     0

errors: No known data errors
Состояние пула снова ONLINE.

В данный момент в нашем пуле два диска: disc1 и disc2. Также в системе имеется диск disc3, но он не подключен к пулу. Предположим, что disc1 вышел из строя и его нужно заменить на disc3.

zpool replace myzfs /disk1 /disk3
zpool status -v

  pool: myzfs
 state: ONLINE
 scrub: resilver completed with 0 errors on Tue Sep 11 13:25:48 2007
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       ONLINE       0     0     0
          mirror    ONLINE       0     0     0
            /disk3  ONLINE       0     0     0
            /disk2  ONLINE       0     0     0

errors: No known data errors
Периодически для исправления ошибок необходимо выполнять процедуру чистки (scrubbing) для пулов типа MIRROR или RAID-Z. Данная процедура находит ошибки в контрольных суммах и исправляет их. Также восстанавливаются сбойные блоки.

Данная операция слишком ресурсоемка! Следует выполнять ее только во время наименьшей нагрузки на пул.

zpool scrub myzfs
Если необходимо перенести пул в другую систему, то его необходимо сначала экспортировать.

zpool export myzfs
pool list

no pools available
А затем импортировать в новой системе.

Если ключ -d не указать, то команда ищет /dev/dsk. Так как в данном примере мы используем файлы, необходимо указать директорию с файлами используемыми хранилищем.

zpool import -d / myzfs
zpool list

NAME          SIZE    USED   AVAIL    CAP  HEALTH     ALTROOT
myzfs        95.5M    114K   95.4M     0%  ONLINE     -
Обновление версии пула. Показать версию используемого пула. Флаг -v показывает возможности, поддерживаемые данным пулом. Используйте флаг -a, чтобы обновить все доступные пулы до новейшей из них версии. Обновленные пулы больше не будут доступны из систем, на которых работают более старые версии.

zpool upgrade
This system is currently running ZFS pool version 8.

All pools are formatted using this version.


zpool upgrade -v

This system is currently running ZFS pool version 8.

The following versions are supported:

VER  DESCRIPTION
---  --------------------------------------------------------
 1   Initial ZFS version
 2   Ditto blocks (replicated metadata)
 3   Hot spares and double parity RAID-Z
 4   zpool history
 5   Compression using the gzip algorithm
 6   pool properties
 7   Separate intent log devices
 8   Delegated administration
For more information on a particular version, including supported 
releases, see:

http://www.opensolaris.org/os/community/zfs/version/N

Where 'N' is the version number.
Если нужно получить статистику по операциям ввода/вывода пулов, выполняем команду.

zpool iostat 5
               capacity     operations    bandwidth 
pool         used  avail   read  write   read  write
----------  -----  -----  -----  -----  -----  -----
myzfs        112K  95.4M      0      4     26  11.4K
myzfs        112K  95.4M      0      0      0      0
myzfs        112K  95.4M      0      0      0      0
Работа с файловой и другими системами ZFS #
Файловая система #
Создадим файловую систему в нашем пуле. Затем проверим автомонтирование новой файловой системы.

zfs create myzfs/colin
df -h

Filesystem   kbytes    used   avail capacity  Mounted on
...
myzfs/colin  64M        18K     63M       1%  /myzfs/colin
Получить список файловых систем ZFS можно следующей командой.

zfs list

NAME          USED  AVAIL  REFER  MOUNTPOINT
myzfs         139K  63.4M    19K  /myzfs
myzfs/colin    18K  63.4M    18K  /myzfs/colin
В данный момент в нашем пуле имеется одно зеркало, в которое входят два диска: disc2 и disc3.

Давайте попробуем расширить наш пул. Попытаемся добавить к нему disc1

zpool add myzfs /disk1

invalid vdev specification
use '-f' to override the following errors:
mismatched replication level: pool uses mirror and new vdev is file
Попытка добавления не удалась, т.к. она неоднозначно и при добавлении диска к существующему зеркалу необходимо указать дополнительно один из существующих в этом зеркале дисков, либо добавить минимум два диска для формирования нового зеркала, которое будет входить в данный пул.

Добавим к пулу новое зеркало, состоящее из дисков: disc1 и disc5

zpool add myzfs mirror /disk1 /disk5
zpool status -v

  pool: myzfs
 state: ONLINE
 scrub: none requested
config:

        NAME         STATE     READ WRITE CKSUM
        myzfs        ONLINE       0     0     0
          mirror     ONLINE       0     0     0
            /disk3   ONLINE       0     0     0
            /disk2   ONLINE       0     0     0
          mirror     ONLINE       0     0     0
            /disk1   ONLINE       0     0     0
            /disk5   ONLINE       0     0     0

errors: No known data errors
Добавим теперь к пулу еще одну файловую систему и посмотрим, как это отразится на размере файловых систем, входящих в пул.

zfs create myzfs/colin2
zfs list

NAME           USED  AVAIL  REFER  MOUNTPOINT
myzfs          172K   159M    21K  /myzfs
myzfs/colin     18K   159M    18K  /myzfs/colin
myzfs/colin2    18K   159M    18K  /myzfs/colin2
Обе файловые системы, входящие в пул, по объему равны всему пулу. В этом заключается одно из преимуществ системы ZFS — по умолчанию нет никакого ограничения на файловые системы.

Чтобы явно управлять объемом файловых систем, можно прибегнуть к резервированию — выделению гарантированного объема для файловой системы, либо квотированию — ограничению файловой системы по максимальному объему.

Давайте зарезервируем для файловой системы /myzfs/colin место в пуле, равное 20 Мб. Остальные файловые системы, заполняя пул, в любом случае оставят для этой файловой системы 20 Мб места.

zfs set reservation=20m myzfs/colin
zfs list -o reservation

RESERV
  none
   20M
  none
Теперь для файловой системы /myzfs/colin2 установим квоту в 20 Мб. Это означает, что данная файловая система не сможет занять в пуле более 20 Мб, даже если пул будет полностью свободным.

zfs set quota=20m myzfs/colin2
zfs list -o quota myzfs/colin myzfs/colin2

QUOTA
 none
  20M
Также для файловой системы /myzfs/colin2 включим сжатие. Сжатие достаточно эффективно работает на уровне ZFS практически без потерь производительности (конечно же, при условии, что производительности сервера достаточно). Вместо compression=on можно использовать compression=gzip.

zfs set compression=on myzfs/colin2
zfs list -o compression

COMPRESS
     off
     off
      on
Чтобы сделать файловую систему доступной по протоколу NFS, достаточно выполнить одну команду. Причем после перезагрузки сервера доступ к файловой системе утерян не будет. Никаких дополнительных настроек операционной системы производить не нужно.

zfs set sharenfs=on myzfs/colin2
zfs get sharenfs myzfs/colin2 

NAME           PROPERTY  VALUE     SOURCE
myzfs/colin2   sharenfs  on        local
Точно так же в одну команду ресурс можно сделать доступным по протоколу SMB. Что пользователям ОС Windows наверняка пригодится.

zfs set sharesmb=on myzfs/colin2
zfs get sharesmb myzfs/colin2

NAME           PROPERTY  VALUE     SOURCE
myzfs/colin2   sharesmb  on        local
Для повышения надежности (если у вас обычный пул, без избыточности), можно использовать следующую опцию файловой системы.

zfs set copies=2 myzfs/colin2
Теперь в файловой системе будет храниться по две копии каждого блока. Это имеет смысл, если пул без избыточности (mirror / raidz).

Snapshots (снепшоты или снимки состояния) #
Создать снепшот файловой системы очень просто. Давайте создадим снепшот для файловой системы myzfs/colin и назовем его test.

zfs snapshot myzfs/colin@test
zfs list

NAME               USED  AVAIL  REFER  MOUNTPOINT
myzfs             20.2M   139M    21K  /myzfs
myzfs/colin         18K   159M    18K  /myzfs/colin
myzfs/colin@test      0      -    18K  -
myzfs/colin2        18K  20.0M    18K  /myzfs/colin2
Если появится необходимость отката к снепшоту, достаточно выполнить одну команду.

zfs rollback myzfs/colin@test
Снэпшот можно подмониторовать, как обычно. Например так.

mount -t zfs myzfs/colin@test /mnt
Даже можно клонировать файловую системы из снепшота в новую файловую систему.

zfs clone myzfs/colin@test myzfs/colin3
zfs list

NAME               USED  AVAIL  REFER  MOUNTPOINT
myzfs             20.2M   139M    21K  /myzfs
myzfs/colin         18K   159M    18K  /myzfs/colin
myzfs/colin@test      0      -    18K  -
myzfs/colin2        18K  20.0M    18K  /myzfs/colin2
myzfs/colin3          0   139M    18K  /myzfs/colin3
Теперь давайте удалим наши файловые системы /myzfs/colin и /myzfs/colin2

Сперва удалим пустую файловую систему /myzfs/colin2

zfs destroy myzfs/colin2
zfs list

NAME               USED  AVAIL  REFER  MOUNTPOINT
myzfs             20.1M   139M    22K  /myzfs
myzfs/colin         18K   159M    18K  /myzfs/colin
myzfs/colin@test      0      -    18K  -
myzfs/colin3          0   139M    18K  /myzfs/colin3
Файловая система удалилась без проблем. Теперь удалим файловую систему, для которой существует снепшот.

zfs destroy myzfs/colin 

cannot destroy 'myzfs/colin': filesystem has children
use '-r' to destroy the following datasets:
myzfs/colin@test
Удаление невозможно, т.к. у файловой системы имеется дочерний объект. Можно воспользоваться параметром -r чтобы удалить файловую систему вместе со всеми дочерними объектами рекурсивно.

Мы можем отключить снепшот от /myzfs/colin и оставить его дочерним только для /myzfs/colin3

zfs promote myzfs/colin3
zfs list

NAME                USED  AVAIL  REFER  MOUNTPOINT
myzfs              20.1M   139M    21K  /myzfs
myzfs/colin            0   159M    18K  /myzfs/colin
myzfs/colin3         18K   139M    18K  /myzfs/colin3
myzfs/colin3@test      0      -    18K  -


zfs destroy myzfs/colin 
zfs list

NAME                USED  AVAIL  REFER  MOUNTPOINT
myzfs               147K   159M    21K  /myzfs
myzfs/colin3         18K   159M    18K  /myzfs/colin3
myzfs/colin3@test      0      -    18K  -
Теперь сделанный ранее снепшот для /myzfs/colin стал дочерним объектом /myzfs/colin3. Таким образом у файловой системы /myzfs/colin больше нет дочерних объектов и ее можно без труда разобрать (удалить).

Если вдруг понадобиться переименовать ранее созданную файловую систему или снепшот, то можно воспользоваться следующими командами.

zfs rename myzfs/colin3 myzfs/bob
zfs list

NAME             USED  AVAIL  REFER  MOUNTPOINT
myzfs            153K   159M    21K  /myzfs
myzfs/bob         18K   159M    18K  /myzfs/bob
myzfs/bob@test      0      -    18K  -

zfs rename myzfs/bob@test myzfs/bob@newtest
zfs list

NAME                USED  AVAIL  REFER  MOUNTPOINT
myzfs               146K   159M    20K  /myzfs
myzfs/bob            18K   159M    18K  /myzfs/bob
myzfs/bob@newtest      0      -    18K  -
Снова вернемся к пулам #
Получить полную информацию о пулах можно следующим образом.

zfs get all

NAME               PROPERTY       VALUE                  SOURCE
myzfs              type           filesystem             -
myzfs              creation       Tue Sep 11 14:21 2007  -
myzfs              used           146K                   -
myzfs              available      159M                   -
myzfs              referenced     20K                    -
[...]
Если пул нам более не нужен, можем его удалить. Однако, нельзя удалить пул, в котором имеются активные файловые системы.

zpool destroy myzfs

cannot destroy 'myzfs': pool is not empty
use '-f' to force destruction anyway
Чтобы принудительно удалить пул, используйте параметр -f (не выполняйте это сейчас. Пул нам еще понадобится далее)

zpool destroy -f myzfs
zpool status -v

no pools available
Отключить файловую систему от пула можно следующим образом.

zfs unmount myzfs/bob
df -h

myzfs                  159M    20K   159M     1%    /myzfs
Подключить файловую систему к пулу вот так.

zfs mount myzfs/bob
df -h

myzfs                  159M    20K   159M     1%    /myzfs
myzfs/bob              159M    18K   159M     1%    /myzfs/bob
Снепшот можно сделать и на удаленный ресурс (или другое место в локальной системе).

zfs send myzfs/bob@newtest | ssh localhost zfs receive myzfs/backup
zfs list

NAME                   USED  AVAIL  REFER  MOUNTPOINT
myzfs                  172K   159M    20K  /myzfs
myzfs/backup            18K   159M    18K  /myzfs/backup
myzfs/backup@newtest      0      -    18K  -
myzfs/bob               18K   159M    18K  /myzfs/bob
myzfs/bob@newtest         0      -    18K  -
В данном случае снепшот передан zfs receive на локальном узле (в демонстрационных целях). В реальной ситуации таким образом можно сделать снепшот на другой узел сети.

Zpool ведет собственную историю всех команд. Посмотреть историю можно следующим образом.

zpool history

History for 'myzfs':
2007-09-11.15:35:50 zpool create myzfs mirror /disk1 /disk2 /disk3
2007-09-11.15:36:00 zpool detach myzfs /disk3
2007-09-11.15:36:10 zpool attach myzfs /disk1 /disk3
2007-09-11.15:36:53 zpool detach myzfs /disk3
2007-09-11.15:36:59 zpool add myzfs spare /disk3
2007-09-11.15:37:09 zpool remove myzfs /disk3
2007-09-11.15:37:18 zpool offline myzfs /disk1
2007-09-11.15:37:27 zpool online myzfs /disk1
2007-09-11.15:37:37 zpool replace myzfs /disk1 /disk3
2007-09-11.15:37:47 zpool scrub myzfs
2007-09-11.15:37:57 zpool export myzfs
2007-09-11.15:38:05 zpool import -d / myzfs
2007-09-11.15:38:52 zfs create myzfs/colin
2007-09-11.15:39:27 zpool add myzfs mirror /disk1 /disk5
2007-09-11.15:39:38 zfs create myzfs/colin2
2007-09-11.15:39:50 zfs set reservation=20m myzfs/colin
2007-09-11.15:40:18 zfs set quota=20m myzfs/colin2
2007-09-11.15:40:35 zfs set compression=on myzfs/colin2
2007-09-11.15:40:48 zfs snapshot myzfs/colin@test
2007-09-11.15:40:59 zfs rollback myzfs/colin@test
2007-09-11.15:41:11 zfs clone myzfs/colin@test myzfs/colin3
2007-09-11.15:41:25 zfs destroy myzfs/colin2
2007-09-11.15:42:12 zfs promote myzfs/colin3
2007-09-11.15:42:26 zfs rename myzfs/colin3 myzfs/bob
2007-09-11.15:42:57 zfs destroy myzfs/colin
2007-09-11.15:43:23 zfs rename myzfs/bob@test myzfs/bob@newtest
2007-09-11.15:44:30 zfs receive myzfs/backup
Ну вот. Основные команды для работы с пулами ZFS усвоены.

Теперь можно удалить сам пул и файлы. Они нам больше не пригодятся.
