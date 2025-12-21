### Q:
Постгрес 16.4
Добавляю частичный индекс на проект, заметил что хэш индекс с условием = не вызывается, с <> вызывается успешно.
Кто-то может обьяснить, почему такое поведение?

### Вот создаем данные:
CREATE TABLE t (
    id BIGSERIAL PRIMARY KEY,
    status TEXT NOT NULL
);

INSERT INTO t(status)
SELECT 'DONE'
FROM generate_series(1, 1000000);

INSERT INTO t(status)
SELECT 'NEW'
FROM generate_series(1, 100);

### Вот этот индекс вызывается:
CREATE INDEX idx_t_status_hash_new ON t USING hash (status) WHERE status <> 'DONE';
EXPLAIN ANALYZE SELECT * FROM t WHERE status = 'NEW' LIMIT 10;
                                                              QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.00..757.06 rows=10 width=12) (actual time=0.014..0.017 rows=10 loops=1)
   ->  Index Scan using idx_t_status_hash_new on t  (cost=0.00..12642.92 rows=167 width=12) (actual time=0.013..0.015 rows=10 loops=1)
         Index Cond: (status = 'NEW'::text)
 Planning Time: 0.192 ms
 Execution Time: 0.028 ms
(5 rows)

### А этот нет:
CREATE INDEX idx_t_status_hash_new ON t USING hash (status) WHERE status = 'NEW';
EXPLAIN ANALYZE SELECT * FROM t WHERE status = 'NEW' LIMIT 10;
                                              QUERY PLAN
-----------------------------------------------------------------------------------------------------------
 Limit  (cost=0.00..1072.43 rows=10 width=12) (actual time=81.296..81.299 rows=10 loops=1)
   Buffers: shared hit=96 read=5311
   ->  Seq Scan on t  (cost=0.00..17909.50 rows=167 width=12) (actual time=81.292..81.294 rows=10 loops=1)
         Filter: (status = 'NEW'::text)
         Rows Removed by Filter: 1000000
         Buffers: shared hit=96 read=5311
 Planning Time: 0.222 ms
 Execution Time: 81.330 ms
(8 rows)


### A1:
[Тестовый стенд](https://www.db-fiddle.com/f/stoC7HmjGyPAxbFcKMV3u1/3)

### A:
Валерий: Хэш не может быть покрывающим и имеет коллизии. Плюсом он показывает не на строки а на страницы (это в доке так пишут). 
В итоге если у нас поиск по прямому индексу и данных мало то планировщик выбирает секскан. А если у нас поиск по инверсному индексу то планировщик видит что страниц можно просканировать меньше и выбирает поиск страниц, которые не попадают в индекс

Решено - используем <> с частичным хеш индексов
