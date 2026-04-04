# Как уменьшить max_connections в Patroni и нафига нам recovery.signal

> **Кейс:** production-кластер PostgreSQL 12 + Patroni 3.3.5 + etcd.  
> При попытке уменьшить `max_connections` с 10000 до 3120 через `patronictl restart -r leader` возникал HTTP 503 и автоматическая смена лидера.  
> **Причина:** «забытый» файл `recovery.signal` на всех нодах из-за дефолтного `keep_existing_recovery_conf: true`.  
> **Решение:** ручное удаление файла на лидере, а в тяжёлых случаях — принудительное обновление `pg_controldata` через ручной старт PostgreSQL.

В статье — диагностика, алгоритм восстановления и профилактика.

---
## 1. Введение: как Patroni меняет требующие перезапуска параметры
---
В кластере под управлением Patroni (с DCS etcd) параметры PostgreSQL хранятся в `/config/<cluster_name>` в etcd. 
Изменение параметра выполняется через `patronictl edit-config`. 
Для параметров, которые нельзя применить через `pg_reload_conf()` (например, `max_connections`), Patroni выставляет флаг `pending restart`.

Обычный процесс:
```sql
patronictl edit-config    # меняем max_connections с 10000 на 3120
patronictl list           # видим '*' в колонке Pending restart
patronictl restart <cluster> -r leader
patronictl restart <cluster> -r replica
```
Но в нашем случае команда patronictl restart -r leader завершалась с 503 Service Unavailable и провоцировала autofailover — лидер переключался на другую ноду, кластер терял доступ на несколько секунд.

Вывод patronictl restart:
```
Failed: restart for member leader failed, status code=503
В логах Patroni (/var/log/paroni/patroni.log) появляется строка:

WARNING: pg_controldata will be used because recovery.signal exists
```
В логах PostgreSQL — ничего критичного, обычный старт в режиме recovery.

После failover новый лидер также не применяет новое значение max_connections, и pending restart никуда не исчезает.

---
## 2. Симптомы
---

Вывод patronictl restart:
```
Failed: restart for member leader failed, status code=503
```
В логах Patroni (/var/log/patroni/patroni.log) появляется строка:
WARNING: pg_controldata will be used because recovery.signal exists

В логах PostgreSQL — ничего критичного, обычный старт в режиме recovery.

После failover новый лидер также не применяет новое значение max_connections, и pending restart никуда не исчезает.

---
### 3. Диагностика: находим корень зла
---
3.1. Обнаружение recovery.signal
На всех 4 нодах кластера был найден файл:

bash
/var/lib/postgresql/12/main/recovery.signal
3.2. Что такое recovery.signal?
В PostgreSQL 12+ наличие этого файла при старте переводит инстанс в режим восстановления (PITR, репликация, восстановление из резервной копии). Файл создаётся автоматически при:

выполнении pgbackrest restore (например, при отставании реплики, когда мастер уже не хранит нужные WAL);

запуске PITR через recovery_target;

некоторых операциях pg_rewind.

3.3. Почему файл не удаляется?
В Patroni 3.3.5 для PostgreSQL >=12 действует настройка по умолчанию:

yaml
bootstrap:
  dcs:
    postgresql:
      parameters:
        keep_existing_recovery_conf: true
Поскольку мы её не переопределяли, Patroni после успешного восстановления не удаляет recovery.signal. В результате на каждой ноде, где когда-либо происходил restore (а кластер пожилой), файл остался.

3.4. Как это мешает уменьшить max_connections?
Когда Patroni выполняет restart -r leader:

Останавливает PostgreSQL.

Запускает его снова.

При наличии recovery.signal PostgreSQL стартует в режиме recovery.

В режиме recovery не применяются изменения из postgresql.auto.conf (где Patroni хранит новые параметры).

Patroni проверяет значение max_connections через pg_controldata (старое, 10000) и видит расхождение с новым (3120), но не может его перезаписать, потому что recovery-режим блокирует обновление контрольных данных.

Выход: ошибка 503 и принудительный failover.

Подтверждение — строка pg_controldata will be used в логе.

---
#### 4. Решение (проверенный алгоритм)
---

Мы воспроизвели проблему на тестовом кластере и выработали алгоритм. Важно: все действия выполняются от root или через sudo, если не указано иное.

---
4.1. Базовый сценарий (если restore_command уже задан)
---
Шаг 1. Удаляем recovery.signal на текущем лидере
bash
rm /var/lib/postgresql/12/main/recovery.signal
Не трогаем этот файл на репликах — он нужен им для репликации.

Шаг 2. Перезапускаем лидера через patronictl
bash
patronictl -c /etc/patroni/patroni.yml restart cluster_name -r leader
После успеха pending restart должен исчезнуть, а SHOW max_connections; на лидере показать 3120.

Шаг 3. Перезапускаем реплики
bash
patronictl -c /etc/patroni/patroni.yml restart cluster_name -r replica

4.2. Если restore_command отсутствовал в конфиге
Проверяем на всех нодах наличие restore_command в разделе postgresql файла patroni.yml. Если нет — добавляем:
```yaml
yaml
postgresql:
  create_replica_methods:
    - pgbackrest
    - basebackup
  recovery_conf:
    restore_command: /usr/bin/pgbackrest archive-get %f %p --stanza=cluster_name
  pgbackrest:
    command: /usr/bin/pgbackrest restore --stanza=cluster_name --delta
    no_params: True
    keep_data: True
```
После этого:

bash
systemctl restart patroni   # на каждой ноде
Проверяем:

sql
SHOW restore_command;
-- должно вернуть команду pgbackrest
Затем выполняем п. 4.1.

4.3. Тяжёлый сценарий: когда pg_controldata не обновляется даже после удаления recovery.signal
Если после выполнения п. 4.1 pending restart не исчез и в логах снова появилось pg_controldata will be used — значит, pg_controldata упорно хранит старое значение 10000. Это случается, если удаление recovery.signal произошло после того, как Patroni уже прочитал контрольные данные, или если PostgreSQL ни разу не стартовал без recovery.signal после изменения параметра.

---
## 5. Почему это работает: разбор механики
---

pg_controldata — бинарный файл в каталоге данных, содержащий критическую информацию о состоянии кластера, включая значение max_connections на момент последнего нормального (не recovery) запуска.

Patroni, обнаружив recovery.signal, не рискует перезаписывать контрольные данные и полагается на то, что там уже записано.

Ручной запуск pg_ctlcluster start (без recovery.signal) заставляет PostgreSQL обновить pg_controldata в соответствии с текущим postgresql.conf.

После этого Patroni, стартуя поверх уже «чистых» контрольных данных, видит новое значение и успешно применяет перезапуск.

---
## 6. Предотвращение в будущем
---

Чтобы не попадать в эту ловушку снова (особенно при уменьшении max_connections или других параметров, требующих перезапуска):

6.1. Явно отключить сохранение recovery.conf
В patroni.yml (секция bootstrap.dcs.postgresql.parameters) добавьте:

yaml
keep_existing_recovery_conf: false
Затем:

bash
patronictl edit-config   # применит изменения в etcd
patronictl restart <cluster> -r leader -r replica
После этого Patroni будет удалять recovery.signal при успешном завершении восстановления.

6.2. При использовании pgbackrest всегда задавать restore_command
Как показано в п. 4.2 — это позволяет Patroni корректно обрабатывать случаи отставания реплик без оставления «висящих» файлов.

6.3. Перед изменением параметров, требующих перезапуска
Простая ручная проверка на лидере:

bash
ssh leader "ls -la /var/lib/postgresql/12/main/recovery.signal"
Если файл есть — удалить, если не уверены — выполнить patronictl flush <cluster> <member> (но лучше проверить документацию).

---
## Заключение
---

Этот кейс — отличный пример того, как даже «безобидный» файл-призрак recovery.signal может нарушить работу, казалось бы, отлаженного механизма Patroni.

Благодарности: коллегам из команды сопровождения, которые помогли воспроизвести проблему на тестовом стенде и отточить решение.

P.S. Если вы столкнулись с похожим поведением на более новых версиях PostgreSQL (13, 14, 15, 16) — механизм с recovery.signal остался тем же. Разница только в пути к каталогу данных (например, /var/lib/postgresql/14/main).

## Полезные ссылки:

Patroni documentation: Replica bootstrap

Patroni source: Bootstrap._custom_bootstrap

PostgreSQL 12: recovery.signal

Обсуждение и вопросы: создавайте Issue в этом репозитории или пишите в Telegram-канал (DBElit).
