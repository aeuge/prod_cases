### Q:
А сталкивались с тем, что patronictl не все ноды отдает, но в логах патрони есть все ноды ? )

### A1:
Проверь конфиг какие ноды в нем указаны, проверь логи патрони, етсд, postgres.

### Q:
Конфиги никто не трогал.
У меня вообще утром два кластера себя так повели. С одним выкрутился путём обновления патрони, а вот другой чего то не хочет в апи корректно отображаться.
```bash
[06.08.2025 10:41]
2025-08-06 10:30:56 MSK [1361165-1]  ВАЖНО:  не удалось начать трансляцию WAL: ОШИБКА:  слот репликации "pg_dev_node03" не существует
2025-08-06 10:31:01 MSK [1361184-1]  ВАЖНО:  не удалось начать трансляцию WAL: ОШИБКА:  слот репликации "pg_dev_node03" не существует
node1
| 26 | 10964296955256 | no recovery target specified | 2025-02-21T15:24:41.860000+00:00 |               |
| 27 | 10988774490112 | no recovery target specified | 2025-03-06T09:35:36.392000+00:00 |               |
| 28 | 11176777346992 | no recovery target specified | 2025-08-06T06:54:36.930679+00:00 | pg-dev-node01 |
node2
| 26 | 10964296955256 | no recovery target specified | 2025-02-21T15:24:41.860000+00:00 |               |
| 27 | 10988774490112 | no recovery target specified | 2025-03-06T09:35:36.392000+00:00 |               |
| 28 | 11176777346992 | no recovery target specified | 2025-08-06T06:54:36.930679+00:00 | pg-dev-node01 |
node3
| 26 | 10964296955256 | no recovery target specified | 2025-02-21T15:24:41.860000+00:00 |               |
| 27 | 10988774490112 | no recovery target specified | 2025-03-06T09:35:36.392000+00:00 |               |
| 28 | 11176777346992 | no recovery target specified | 2025-08-06T06:54:36.930679+00:00 | pg-dev-node01 |
Все началось с ERROR: Request to server http://10.14.0.173:2379 failed: MaxRetryError(.
Aug  6 06:54:22 pg-dev-node02 patroni[775]: 2025-08-06 06:54:22,960 INFO: Leader key is not deleted and Postgresql is not stopped due paused state
Aug  6 06:54:23 pg-dev-node02 systemd[1]: patroni.service: Succeeded.
Aug  6 06:54:23 pg-dev-node02 systemd[1]: patroni.service: Found left-over process 1426895 (postgres) in control group while starting unit. Ignoring.
Aug  6 06:54:39 pg-dev-node02 patroni[178889]: 2025-08-06 09:54:39 MSK [178889-1]  ВАЖНО:  ранее выделенный блок разделяемой памяти (ключ 5432001, ID 4) по-прежнему используется
Aug  6 06:54:39 pg-dev-node02 patroni[178889]: 2025-08-06 09:54:39 MSK [178889-2]  ПОДСКАЗКА:  Завершите все старые серверные процессы, работающие с каталогом данных "/var/lib/postgresql/10/pg-dev-cluster".


Началось всё с 
ERROR: Request to server http://10.14.0.173:2379 failed: MaxRetryError(

Марсель Габдрахманов, [06.08.2025 11:54]
Aug  6 06:54:22 pg-dev-node02 patroni[775]: 2025-08-06 06:54:22,960 INFO: Leader key is not deleted and Postgresql is not stopped due paused state
Aug  6 06:54:23 pg-dev-node02 systemd[1]: patroni.service: Succeeded.
Aug  6 06:54:23 pg-dev-node02 systemd[1]: patroni.service: Found left-over process 1426895 (postgres) in control group while starting unit. Ignoring.
Aug  6 06:54:39 pg-dev-node02 patroni[178889]: 2025-08-06 09:54:39 MSK [178889-1]  ВАЖНО:  ранее выделенный блок разделяемой памяти (ключ 5432001, ID 4) по-прежнему используется
Aug  6 06:54:39 pg-dev-node02 patroni[178889]: 2025-08-06 09:54:39 MSK [178889-2]  ПОДСКАЗКА:  Завершите все старые серверные процессы, работающие с каталогом данных "/var/lib/postgresql/10/pg-dev-cluster".
```
Решено - ошибка была в рассинхроне времени на ETCD.
