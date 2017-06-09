Фактически, unique not null и primary key в postgresql - это одно и то же. Разница заключается только в паре флагов в системных представлениях pg_constraint и pg_index. Например, превратим первичный ключ в уникальный через эти представления
```
create table t (a integer primary key, b integer unique);

update pg_constraint set contype = 'u' where conname = 't_pkey';
  
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
