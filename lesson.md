## Select
```sql
-- получаем кортеж
select 1;
select 1, 2, 3, 4;
-- внезапно! кортеж в кортеже, муххаха 
select (1);
select pg_typeof((1));
select pg_typeof((1, 2));
select (1, 2), (3, 4);
select row(1, 2), row(3, 4);

create type t2 as (a int);
select (1) = (row(1)::t2).a;
select (1,2) = row(1,2);
```

## Values
```sql
-- а как получить несколько строк?
select 1, 2 union all select 3, 4;
-- все используют в insert into... values, но можно без insert
values (1);
values (1, 2);
values (1, 2), (3, 4);
```

## CTE
```sql
with t1(a) as (values (1), (1), (2)) select a from t1;
create table t (a int);
insert into t select generate_series(1, 1000);
create index t_idx on t using btree (a);
explain with t1(a) as not materialized (select * from t) select * from t1 where a = 666;
explain with t1(a) as materialized (select * from t) select * from t1 where a = 666;
```
### CTE returning
```sql
create table tc(a, b) as values (1, 10), (2, 20), (3, 30);
with t as (update tc set a = 666 where b = 10 returning *) select * from t;
with t(a) as (delete from tc where b = 10 returning a) select * from t;
```

### Recursive CTE
```sql
with recursive t(a) as (select 1 union all select a + 1 from t where a < 10) 
select a from t;

with recursive tr(id, pid, value) as (
        select * from t where id = 3
        union all
        select t.* from t, tr where t.id = tr.pid
),
t(id, pid, value) as (
        values (1, null, 'khabarovsk'),
        (2, 1, 'gogolya'),
        (3, 2, '39'),
        (4, null, 'vladivostok')
)
select string_agg(value, ',' order by id) from tr;
```

## Join
```sql
with t1(a) as (values (1), (1), (2)), t2(a) as (values (1), (3)) select a from t2;
-- декартово произведение (cross)
with t1(a) as (values (1), (1), (2)), t2(a) as (values (1), (3)) select t1.a, t2.a from t1 cross join t2;
with t1(a, b) as (values (1, 'a1'), (1, 'b1')), t2(a, b) as (values (1, 'a2'), (3, 'b2')) select t1.*, t2.* from t1, t2;
-- декартово произведение с фильтрацией одинаковых ключей (join)
with t1(a, b) as (values (1, 'a1'), (1, 'b1')), t2(a, b) as (values (1, 'a2'), (1, 'b2'), (2, 'c1')) select t1.*, t2.* from t1 join t2 using(a);
-- null слева (left)
with t1(a, b) as (values (1, 'a1'), (1, 'b1'), (3, 'd1')), t2(a, b) as (values (1, 'a2'), (1, 'b2'), (2, 'c1')) select t1.*, t2.* from t1 left join t2 using(a);
-- null справа (right)
with t1(a, b) as (values (1, 'a1'), (1, 'b1'), (3, 'd1')), t2(a, b) as (values (1, 'a2'), (1, 'b2'), (2, 'c2')) select t1.*, t2.* from t1 right join t2 using(a);
-- natural (замена using)
with t1(a, b) as (values (1, 'a1'), (1, 'b1'), (3, 'd1')), t2(a, b) as (values (1, 'a1'), (1, 'b2'), (2, 'c1')) select t1.*, t2.* from t1 natural join t2;
-- хотим nested loop (lateral!)
with t1(a) as (values (1), (2)), t2(a) as (values (2), (3)) select t1.a, q.* from t1 join lateral (select * from t2 where a = t1.a) as q using(a);
```

## Array
```sql
with t(a) as (select generate_series(1,10)) select unnest(array_agg(a)) from t;
with t(a) as (select array_agg(g) from generate_series(1, 10) as g) select a from t;
with t(a) as (select array_agg(g) from generate_series(1, 10) as g) select a[10] from t;
with t(a) as (select array_agg(g) from generate_series(1, 9) as g) select a[10] from t;
with t(a) as (select array_agg(g) from generate_series(1, 1) as g)
select case
        when array_length(a, 1) = 1 then a[1]
        when array_length(a, 1) = 2 then a[2]
end from t;
with t(a) as (select array_agg(g) from generate_series(1, 3) as g)
select case
        when array_length(a, 1) = 1 then a[1]
        when array_length(a, 1) = 2 then a[2]
end from t;
```

## Conditional error
```sql
-- вызов ошибки по условию в sql запросе
create function error(e text) returns int as $$begin raise exception '%', e;end$$ language plpgsql;
with t(a) as (select array_agg(g) from generate_series(1, 3) as g)
select case
        when array_length(a, 1) = 1 then a[1]
        when array_length(a, 1) = 2 then a[2]
        else error('wat?!')
end from t;
```

## Array/unnest (свертки)

Array не может превышать 1GB
```sql
select array[1,2,3]::int[];
create type t1 as (a int, b timestamptz);
select array[(1, now()),(2, now())]::t1[];
select unnest(array[(1, now()),(2, now())]::t1[]);
select (unnest(array[(1, now()),(2, now())]::t1[])).a;

with t(a, b) as (select g, now() - interval '1 week' * random() from generate_series(1, 3) as g) select a, b from t;
with t(a, b) as (select g, now() - interval '1 week' * random() from generate_series(1, 3) as g) select array_agg((a, b)) from t;
-- для работы с колонками строка должна быть приведена к определенному типу
with t(a, b) as (select g, now() - interval '1 week' * random() from generate_series(1, 3) as g) select array_agg((a, b)::t1) from t;
with t(a, b) as (select g, now() - interval '1 week' * random() from generate_series(1, 3) as g) select (unnest(array_agg((a, b)::t1))).a from t;
```

## Jsonb (свертки)

Json объект не может превышать 1GB
```sql
select jsonb_build_object('a', 1, 'b', now());
select jsonb_build_object('a', g, 'b', now() - random() * interval '1 week') from generate_series(1, 10) as g;
select jsonb_agg(jsonb_build_object('a', g, 'b', now() - random() * interval '1 week')) as j from generate_series(1, 10) as g;
with t(j) as (
    select jsonb_agg(jsonb_build_object('a', g, 'b', now() - random() * interval '1 week')) 
    from generate_series(1, 10) as g
)
select * from jsonb_to_recordset((select j from t)) as f(a int, b timestamptz);
```

## Пагинация

```sql
create table tp(c int, t timestamptz, i int);
insert into tp select random()::int, now() - random() * interval '1 week', (random() * 10000)::int from generate_series(1, 100000);
create index tp_idx on tp using btree(c, t, i);
vacuum freeze tp;
--в индексе не учтена сортировка, муххаха
explain analyze select * from tp where c = 1 order by t desc, i offset 10 limit 10;

drop index tp_idx;
create index tp_idx on tp using btree(c, t desc, i);
-- 1 странциа результатов - 10 указателей из индекса
explain analyze select * from tp where c = 1 order by t desc, i offset 0 limit 10;
-- 2 страница - еще 20 (сумма 30 > 20) и т.д.
explain analyze select * from tp where c = 1 order by t desc, i offset 10 limit 10;
--сохраняем последнюю строку в приложение и используем индекс
explain analyze select t, i from tp where c = 1 and (t, i) < ('2019-10-29 13:34:37.859964+00'::timestamptz, 2826) order by t desc, i limit 10;
-- lateral for fun
explain analyze select tp.* from tp left join lateral (select t, i from tp where c = 1 order by t desc, i offset 9 limit 1) as tmp on true where c = 1 and (tp.t, tp.i) < (tmp.t, tmp.i) order by
 t desc, i limit 10;
```