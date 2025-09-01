### Q:
Коллега утверждает, что есть табличка с номерами телефонов(100 млн bigint) и индексом по ним. Тупо медленно битри работает и они разбили диапазон на 10 частичных неперекрывающихся индексов по префиксу 790-799 и теперь всё просто летает)

### Q:
```bash
drop table if exists phone;
CREATE TABLE phone
(
    id serial PRIMARY KEY,
    contact bigint
);

\timing
truncate phone;
INSERT INTO phone (contact)
SELECT ('79' || floor(random()*99000000))::bigint
FROM generate_series(0, 100_000_000) AS t(i);

-- 4 минуты
-- 4,2 GB

select * from phone limit 10;
create index idx_phone on phone(contact);


cat > ~/workload_phone.sql << EOL
\set r random(79000000000, 79990000000) 
SELECT contact FROM phone WHERE id = :r;
EOL

/usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -n -f ~/workload_phone.sql

-- 70к +
drop index idx_phone;
create index idx_phone0 on phone(contact) where contact >= 7_900_000_000 and contact < 7_910_000_000;
create index idx_phone1 on phone(contact) where contact >= 7_910_000_000 and contact < 7_920_000_000;
create index idx_phone2 on phone(contact) where contact >= 7_920_000_000 and contact < 7_930_000_000;
create index idx_phone3 on phone(contact) where contact >= 7_930_000_000 and contact < 7_940_000_000;
create index idx_phone4 on phone(contact) where contact >= 7_940_000_000 and contact < 7_950_000_000;
create index idx_phone5 on phone(contact) where contact >= 7_950_000_000 and contact < 7_960_000_000;
create index idx_phone6 on phone(contact) where contact >= 7_960_000_000 and contact < 7_970_000_000;
create index idx_phone7 on phone(contact) where contact >= 7_970_000_000 and contact < 7_980_000_000;
create index idx_phone8 on phone(contact) where contact >= 7_980_000_000 and contact < 7_990_000_000;
create index idx_phone9 on phone(contact) where contact >= 7_990_000_000 and contact < 7_999_000_000;
-- создаются дольше в 2 раза

\di+

drop index idx_phone;

-- 62к rps

drop table phone;
```

Проверил кейс - ожидаемо больше тратится время на выбор конкретного индекса, чем по нему проверка

плюс время создания индекса в 2 рпзп выше

### A:
А ты самое главное не сказал - какие запросы там твой перец гоняет.
Может он диапазоны выбирает или в БД за подстановкой в процессе набора лезет...
В целом, можно вообще без индексов разогнать табличку, если сделать ее секционированной и не использовать планировщик для partition pruning а лезть в секцию напрямую.
У тебя тогда скорость будет уххх какая/
Хотелось бы больше детализации и как сказал уважаемый, было бы неплохо попугаев посмотреть. Навскидку причины, по которым произошло улучшение: 
- большое количество точечных выборок , ядра не успевают обработать большой индекс
- слабое железо ( индекс не влезает в память)
- большое количество запросов к другим данным БД, что приводит к вымыванию страниц индекса из кэша
Шпиндели/сетевые полки с медленными дисками или загруженными fibrechannels
- дешёвый виртуальный хостинг (колокация БД с другими нагруженными виртуалками)

