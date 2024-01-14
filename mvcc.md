
    Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB
Установить на него PostgreSQL 15 с дефолтными настройками
Создать БД для тестов: выполнить pgbench -i postgres
Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres

*Это 8 соединений, сообщать статистику каждые 6 сек. работы, работать в течение 60 сек. 
    
    pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
    starting vacuum...end.
    progress: 6.0 s, 623.7 tps, lat 12.785 ms stddev 6.345, 0 failed
    progress: 12.0 s, 635.2 tps, lat 12.602 ms stddev 6.888, 0 failed
    progress: 18.0 s, 639.3 tps, lat 12.510 ms stddev 6.137, 0 failed
    progress: 24.0 s, 628.5 tps, lat 12.721 ms stddev 6.813, 0 failed
    progress: 30.0 s, 601.5 tps, lat 13.306 ms stddev 6.873, 0 failed
    progress: 36.0 s, 589.2 tps, lat 13.580 ms stddev 7.495, 0 failed
    progress: 42.0 s, 592.0 tps, lat 13.503 ms stddev 7.002, 0 failed
    progress: 48.0 s, 590.5 tps, lat 13.540 ms stddev 7.718, 0 failed
    progress: 54.0 s, 579.3 tps, lat 13.824 ms stddev 8.478, 0 failed
    progress: 60.0 s, 581.3 tps, lat 13.756 ms stddev 8.004, 0 failed
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 8
    number of threads: 1
    maximum number of tries: 1
    duration: 60 s
    number of transactions actually processed: 36371
    number of failed transactions: 0 (0.000%)
    latency average = 13.197 ms
    latency stddev = 7.204 ms
    initial connection time = 9.727 ms
    tps = 606.076856 (without initial connection time)

    
Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
Протестировать заново
    
    starting vacuum...end.
    progress: 6.0 s, 564.2 tps, lat 14.128 ms stddev 7.514, 0 failed
    progress: 12.0 s, 597.6 tps, lat 13.394 ms stddev 7.339, 0 failed
    progress: 18.0 s, 618.2 tps, lat 12.938 ms stddev 6.073, 0 failed
    progress: 24.0 s, 604.7 tps, lat 13.233 ms stddev 6.676, 0 failed
    progress: 30.0 s, 620.8 tps, lat 12.877 ms stddev 6.607, 0 failed
    progress: 36.0 s, 562.3 tps, lat 14.234 ms stddev 8.064, 0 failed
    progress: 42.0 s, 581.3 tps, lat 13.755 ms stddev 7.827, 0 failed
    progress: 48.0 s, 587.8 tps, lat 13.612 ms stddev 7.755, 0 failed
    progress: 54.0 s, 606.7 tps, lat 13.188 ms stddev 6.729, 0 failed
    progress: 60.0 s, 588.3 tps, lat 13.588 ms stddev 7.135, 0 failed
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 8
    number of threads: 1
    maximum number of tries: 1
    duration: 60 s
    number of transactions actually processed: 35600
    number of failed transactions: 0 (0.000%)
    latency average = 13.482 ms
    latency stddev = 7.194 ms
    initial connection time = 11.430 ms
    tps = 593.263635 (without initial connection time)

Что изменилось и почему?
    
*Изменений в процентном соотношении очень мало и непонятно с чем они связаны.     
    
Создать таблицу с текстовым полем (добавлен номер строки)

    
       lesson8=# select current_database();
	 current_database 
	------------------
 	lesson8
	(1 row)
	
    lesson8=# \dt
        List of relations
     Schema | Name | Type  |  Owner   
    --------+------+-------+----------
     public | les  | table | postgres
    (1 row)


...и заполнить случайными или сгенерированными данным в размере 1млн строк

    sudo apt install postgresql-15-orafce
    create extension if not exists orafce;

 	lesson8=# INSERT INTO les(letters) select dbms_random.string('U',4) 
           from generate_series(1,10);
    INSERT 0 10
    lesson8=# select id, letters from les;
     id |                                               letters                                                
    ----+------------------------------------------------------------------------------------------------------
      1 | IJOO                                                                                                
      2 | AOTX                                                                                                
      3 | SHTN                                                                                                
      4 | QDUZ                                                                                                
      5 | KNAG                                                                                                
      6 | VHVT                                                                                                
      7 | GNYI                                                                                                
      8 | OPRW                                                                                                
      9 | ZFLZ                                                                                                
     10 | TEWL                                                                                                
    (10 rows)

    lesson8=# INSERT INTO les(letters) select dbms_random.string('U',4) 
           from generate_series(1,999990);
    INSERT 0 999990
    lesson8=# select id,letters from les where id > 999990;
       id    |                                               letters                                                
    ---------+------------------------------------------------------------------------------------------------------
      999991 | UBRK                                                                                                
      999992 | POTH                                                                                                
      999993 | XYGD                                                                                                
      999994 | POMO                                                                                                
      999995 | BGVH                                                                                                
      999996 | TSCJ                                                                                                
      999997 | NJLY                                                                                                
      999998 | PJBK                                                                                                
      999999 | LSVB                                                                                                
     1000000 | HOIE                                                                                                
    (10 rows)

   
Посмотреть размер файла с таблицей

    lesson8=# SELECT pg_relation_filepath('les');
     pg_relation_filepath 
    ----------------------
     base/16389/17178
    (1 row)

    lesson8=# \! ls -lh  /var/lib/postgresql/15/main/base/16389/17178
    -rw------- 1 postgres postgres 135M янв 14 17:16 /var/lib/postgresql/15/main/base/16389/17178

5 раз обновить все строчки и добавить к каждой строчке любой символ

    lesson8=# update les set letters = letters || (select dbms_random.string('U',1));
    UPDATE 1000000
    *(5 раз)*

Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
    
*Пока писал предыдущий пункт приходил автовакуум:

   lesson8=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'les';
    relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
   ---------+------------+------------+--------+-------------------------------
    les     |    1000000 |          0 |      0 | 2024-01-14 17:55:53.904036+00
   (1 row)

*Если пройти еще один раз апдейтом, нечищенная таблица выглядит так: 

   lesson8=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'les';
    relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
   ---------+------------+------------+--------+-------------------------------
    les     |    1000000 |    1000000 |     99 | 2024-01-14 17:55:53.904036+00
   (1 row)

Подождать некоторое время, проверяя, пришел ли автовакуум
    
* На обновленную выше всяких порогов  (20%+50строк), т.е. на  99% авовакуум приходит в течение 3 мин.

5 раз обновить все строчки и добавить к каждой строчке любой символ
Посмотреть размер файла с таблицей

    lesson8=# SELECT pg_size_pretty(pg_TABLE_size('les'));
     pg_size_pretty 
    ----------------
     808 MB
    (1 row)

Отключить Автовакуум на конкретной таблице

    alter table les set (autovacuum_enabled = off);

10 раз обновить все строчки и добавить к каждой строчке любой символ
Посмотреть размер файла с таблицей

Файл привысил допустимый предел заполнения диска (Эффетивно было доступно 8,5ГБ). После превышения параметра серовер упал. Сервер не запускается.

    2024-01-14 18:46:01.304 UTC [1197] СООБЩЕНИЕ:  работа системы БД была прервана во время восстановления: 2024-01-14 18:44:06 UTC
    2024-01-14 18:46:01.304 UTC [1197] ПОДСКАЗКА:  Это скорее всего означает, что некоторые данные повреждены и вам придётся восстановить БД из последней резервной копии.


Объясните полученный результат

*в результате апдейтов невакуумизированной таблицы (каждый апдейт добавляет то жде количество строк, что было, сами строки больше предыдущих, и так * 10 раз) диск забился. Диск заполняли не только файл таблицы, а еще _fsm, журналы, видимо статистика и пр (после удаления файлов таблицы в /var/lib/postgresql/15/main/ остается 3,3 Гб).


