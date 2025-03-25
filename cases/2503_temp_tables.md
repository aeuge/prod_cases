###Q^
как храняться и работают  temp tables

###Стенд:
Postgres 17-

###A:
Оказывается, temp tables сначала сохраняются на диск и работа идет через temp_buffers.
Почему? ХЗ, но есть идея, что для снижения конкурентности в shared buffers
https://www.postgresql.org/docs/current/runtime-config-resource.html#GUC-TEMP-BUFFERS

###Что делать?
Протестировать снижение производительности с другими вариантами