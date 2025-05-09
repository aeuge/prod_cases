### Q:
Коллеги, столкнулся с достаточно интересным поведением в работе синтаксической конструкции подзапросов. 
Если пишу select "FIELDNAME" from schema.Table where fieldname2 IN (select subquery), то результат возвращается очень медленно, несмотря на то, что select subquery отрабатывает <1c. А если select "FIELDNAME" from schema.Table where fieldname2 = (select subquery), то в аутпут приходит столбец без значений. Кто сталкивался с подобным поведением?

### Стенд:  
PG 16, запрос через fdw идет в MySQL, срабатывает foreigh scan.

### Уточнение задачи:  
(ANY) Попробовал select "FIELDNAME" from schema.Table where fieldname2 = ANY (select subquery) - 29 секунд, несмотря на то, что подзапрос отрабатывает за 214 мс.
( = ) Если не использовать fdw(локально), отрабатывает очень быстро <1с - select "FIELDNAME" from internalschema.Table where fieldname2 = (select subquery).
( = ) А если использовать fdw - select "FIELDNAME" from fwdexternalschema.Table where fieldname2 = (select subquery) то результат не возвращается.
Т.е. с ANY запрос работает, но долго, с = - не возвращает результата вовсе.

### Как решать задачу: 
EXPLAIN + EXPLAIN ANALYZE наше всё. 
Использовать операторы в соответствии с задачей - получить массив или 1 элемент.
Разница при использовании ANY, IN и = в том, что:
- при ANY (subquery), запрос вернет строки, если fieldname2 равно хотя бы одному значению из подзапроса, т.е., если есть хотя бы одно совпадение;
- при = (subquery) подзапрос возвращает ровно одно значение. Если подзапрос вернёт более одного значения или не вернёт ничего, возникнет ошибка или запрос не вернёт результатов.
- при IN (subquery), если значение fieldname2 совпадает с любым значением, возвращаемым подзапросом, это условие всё равно будет работать.
Именно поэтому = (subquery) не возвращало данных, а ANY и IN возвращали.

https://www.percona.com/blog/sql-optimizations-in-postgresql-in-vs-exists-vs-any-all-vs-join/

### Варианты решений:  
- необходимо поисследовать кейсы использования промежуточных результатов, возвращаемых из внешней таблицы + использование индексов

### A:  
https://github.com/tds-fdw/tds_fdw/issues/168
Local subquery being corrupted - query producing incorrect results ·
Известный баг в FDW для MSSQL

Выбрать правильный оператор между ANY, IN и =, обратить внимание на EXISTS и поведение индексов во внешней таблице.

