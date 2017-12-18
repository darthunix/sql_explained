Фактически, unique not null и primary key в postgresql - это одно и то же. Разница заключается только в паре флагов в системных представлениях pg_constraint и pg_index. Например, превратим первичный ключ в уникальный через эти представления
```sql
create table t (a integer primary key, b integer unique);

update pg_constraint set contype = 'u' where true
  and conname = 't_pkey'
  and connamespace = (select oid from pg_namespace where nspname = 'public');
  
update pg_index set indisprimary = false 
where indexrelid = (
  select oid from pg_class 
  where true
    and relname = 't_pkey'
    and relnamespace = (
      select oid from pg_namespace 
      where nspname = 'public'
    )
);
```

*UPDATE:* Есть одно существенное различие - primary key не поддерживает выбор порядока сортировки по колонке и использует только asc. Более того, через ddl его не получится обмануть:
```sql
create table t3 as
select generate_series as id, now() - random() * interval '1 year' as stamp from generate_series(1,1000);

create unique index t3_idx on t3 (id, stamp desc);

alter table t3 add primary key using index t3_idx;
--Error!
--index "t3_idx" does not have default sorting behavior
--Cannot create a primary key or unique constraint using such an index.
```
Если попробовать создать primary key (id, stamp), а потом попытаться через метод выше подменить его на unique index not null (id, stamp desc), то все равно будет использоваться Index Only Scan Backward. Где в системных каталогах можно поменять ожидаемый плнировщиком порядок сортировки у столбца на desc я не нашел...
Поэтому мы всегда будем попадать на Index Only Scan Backward в primary key, который медленнее прямого Index Only Scan.
