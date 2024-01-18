ДЗ WAL

Описание/Пошаговая инструкция выполнения домашнего задания:


    postgres=# show checkpoint_timeout;
     checkpoint_timeout 
    --------------------
     5min
    (1 row)

    postgres=# select pending_restart from pg_settings where name = 'checkpoint_timeout';
     pending_restart 
    -----------------
     f

Настройте выполнение контрольной точки раз в 30 секунд.

    alter system set checkpoint_timeout = '30s'; 
    SELECT pg_reload_conf();
     pg_reload_conf 
    ----------------
     t

    postgres=# show checkpoint_timeout;
     checkpoint_timeout 
    --------------------
     30s
    (1 row)

    prince@l3:~$ sudo -u postgres ls -lh  /var/lib/postgresql/15/main//global/pg_control
    -rw------- 1 postgres postgres 8,0K янв 16 12:58 /var/lib/postgresql/15/main//global/pg_control

    prince@l3:~$ sudo -u postgres ls -lh  /var/lib/postgresql/15/main/pg_wal
    total 49M
    -rw------- 1 postgres postgres  16M янв 16 12:58 000000010000000000000004
    -rw------- 1 postgres postgres  16M янв 11 20:13 000000010000000000000005
    -rw------- 1 postgres postgres  16M янв 11 20:13 000000010000000000000006
    drwx------ 2 postgres postgres 4,0K янв 11 19:44 archive_status

10 минут c помощью утилиты pgbench подавайте нагрузку.

Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.

    postgres=# select * from pg_ls_waldir() limit 10;
               name           |   size   |      modification      
    --------------------------+----------+------------------------
     000000010000000000000019 | 16777216 | 2024-01-16 15:25:11+00
     000000010000000000000018 | 16777216 | 2024-01-16 15:24:40+00
     000000010000000000000017 | 16777216 | 2024-01-16 15:26:42+00
    (3 rows)

### Мы имеем примерно 20 контрольных точек и 11 файлов по 16 МБ, отработанных и большей частью перезаписанных, а потом удаленнных (до операции номер файла сегмента макс. 6, а после мин. 17). 

 16*11:20 = 8.8 Мб на 1 контрольную точку.

Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?

            relname        | blocks | % of rel | % hot 
    -----------------------+--------+----------+-------
     pgbench_accounts      |   1738 |      100 |   100
     pgbench_history       |    634 |      100 |   100
     pgbench_accounts_pkey |    276 |      100 |   100
     pg_attribute          |     34 |       56 |    51
     pg_proc               |     25 |       23 |     7
     pg_class              |     17 |       94 |    78
     pg_toast_2619         |     15 |      100 |    87
     pg_operator           |     13 |       72 |    56
     pg_proc_oid_index     |     10 |       91 |    73
     pgbench_tellers       |      9 |      100 |   100

https://habr.com/ru/companies/postgrespro/articles/460423/

    postgres=# SELECT * FROM pg_stat_bgwriter \gx
    -[ RECORD 1 ]---------+------------------------------
    checkpoints_timed     | 151
    checkpoints_req       | 1
    checkpoint_write_time | 566577
    checkpoint_sync_time  | 3335
    buffers_checkpoint    | 33989
    buffers_clean         | 0
    maxwritten_clean      | 0
    buffers_backend       | 724
    buffers_backend_fsync | 0
    buffers_alloc         | 3097
    stats_reset           | 2024-01-16 12:58:23.443711+00

### Соотношение checkpoints_timed и  checkpoints_req говорит о штатной работе 

Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.

Sync

    tps = 303.283266  

Async

    tps = 2484.626915

###Синхронный режим встречает как минимум  три вида задержек. 1. Транзакция считается успешной после сообщения о записи на диск. 2. wal_writer_delay после каждой записи wal. 3. Время на саму запись. Т.е. длительная сама по себе операция записи замедляется приостановкой на передачу ответа и дополнительным таймаутом. При коротких многочисленных транзакциях это создает критичную задержку. При synchronous_commit = off клиент получает подтверждение что транзакция успешна пока данные еще не записаны. Запись происходит по мере накопления данных для записи и новые транзакции произхводятся в памяти когда предыдущие еще пишутся на диск. К тому же, на каждой транзакции экономится 200 мс за счет исключения wal_writer_delay.


Создайте новый кластер с включенной контрольной суммой страниц. 

    NB: In Debian/Ubuntu packaging, pg_checksums is with the other binaries in:     /usr/lib/postgresql/<major version>/bin/ И почему-то не запускается обычным образом. 

Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. 

    sudo /usr/lib/postgresql/15/bin/pg_checksums -e -D /var/lib/postgresql/15/main/

    less9=# select pg_relation_filepath('les');
     pg_relation_filepath 
    ----------------------
     base/24715/24721
    (1 row)

Добавляем с помощью nano пару байт, проверяем 

    sudo /usr/lib/postgresql/15/bin/pg_checksums -c -D /var/lib/postgresql/15/main/
    pg_checksums: ошибка: ошибка контрольных сумм в файле "/var/lib/postgresql/15/main//base/    24715/24721", блоке 0: вычислена контрольная сумма 4F53, но блок содержит 1104

    less9=# select * from les \gx
    ПРЕДУПРЕЖДЕНИЕ:  ошибка проверки страницы: получена контрольная сумма 20307, а ожидалась - 4356
    ОШИБКА:  неверная страница в блоке 0 отношения base/24715/24721

Что и почему произошло? как проигнорировать ошибку и продолжить работу?

### Зафиксировано отличие контрольнгой суммы измененного файла относительно ранее вычисленой. Отключаем проверку:

    less9=# alter system set ignore_checksum_failure = 'on';
    ALTER SYSTEM
    less9=# select pg_reload_conf();
     pg_reload_conf 
    ----------------
     t
    (1 row)

###Получаем данные в том виде, в котором они сохранились:

    less9=# select * from les where tupnum = '1';
    ПРЕДУПРЕЖДЕНИЕ:  ошибка проверки страницы: получена контрольная сумма 20307, а ожидалась - 4356
     tupnum | descr 
    --------+-------
          1 | --las
    (1 row)

### Экспорт таблицы и импорт прошли успешно с поправкой на испорченную часть данных. NB: папка, из которой экспортируется файл, должна быть readeble  для  group и(или?) others

    sudo -u postgres psql less9 < /tmp/les.sql
    





