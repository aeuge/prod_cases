Q:
Коллеги, столкнулся с достаточно интересным поведением в работе синтаксической конструкции подзапросов. Если пишу 
select "FIELDNAME" from schema.Table where fieldname2 IN (select subquery) то результат возвращается очень медленно, несмотря на то, что select subquery отрабатывает <1c. А если select "FIELDNAME" from schema.Table where fieldname2 = (select subquery), то в аутпут приходит столбец без значений. Кто сталкивался с подобным поведением?
