```sql
create table t1(a integer);
create table t2(a integer, b integer);

insert into t1 select generate_series(1, 5);
insert into t2 select random()*3, generate_series(10,20);

select l.a, r.b from t1 as l
left join lateral (
  select l.a, b from t2 where a = l.a
) as r using(a)
where l.a in (1,2);
```

Это простейший пример, который показывает применение lateral join. Тот же результат можно получить обычным join, но это схематичный запрос.
Lateral по факту заменяет цикл, и запрос выше можно переписать псевдокодом:
```sql
foreach value in (select a from t1 where a in (1, 2)):
  select value as a, b from t2 where a = value
```
То есть когда у нас у нас есть запрос, который делает все, что нам надо при фиксированном значении
```sql
select a, b from t2 where a = ?     (1)
```
и мы хотим не особо меняя структуру запроса размоножить его для нескольких фиксированных значений, мы применяем алгоритм:
* помещаем запрос (1) в правую часть join lateral, а вместо **?** ставим значение из левой части. Это же значение из левой части указываем в возвращаемых колонках, чтобы по нему сделать соединение с левой частью 
```sql
...
left join lateral (
  select l.a, b from t2 where a = l.a
)
...
```
* в правой части дублируем структуру возвращаемых значений запроса (1) и создаем цкл перебора
```sql
select l.a, r.b from t1 as l
...
```
* Соединяем левую и правую части по проброшенному в правую часть полю **l.a**
```sql
...
as r using(a)
...
```
* Добавляем отборы для левой части снаружи соединения (как в обычном join), если в них есть необходимость
```sql
...
where l.a in (1,2);
```
* PROFIT!!!
