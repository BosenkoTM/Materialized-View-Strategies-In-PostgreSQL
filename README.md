# Стратегии материализованного представления в PostgreSQL

Запросы, возвращающие совокупные, сводные и вычисленные данные, часто используются при разработке приложений. 
	Иногда эти запросы выполняются недостаточно быстро. Кэширование результатов запроса с помощью Memcached или Redis — распространенный подход к решению этих проблем с производительностью. 
	Однако они приносят свои собственные проблемы. Прежде чем обратиться к внешнему инструменту, стоит изучить, какие методы предлагает PostgreSQL для кэширования результатов запросов.

## Пример БД
Рассмотрим подходы, используя пример домена(бд) упрощенной системы учета. 

	- Счета могут иметь много транзакций. 
  
	- Транзакции могут быть записаны заранее и вступят в силу только после их завершения. 
  
	- Дебет, например, поступивший в марте, можно ввести в марте.

```sql
create table accounts(
  name varchar primary key
);

create table transactions(
  id serial primary key,
  name varchar not null references accounts
    on update cascade
    on delete cascade,
  amount numeric(9,2) not null,
  post_time timestamptz not null
);

create index on transactions (name);
create index on transactions (post_time);
```

## Данные и запросы

1.Создать базу данных `pg_cache_demo`:

```sql
CREATE DATABASE pg_cache_demo
WITH
OWNER = admin
ENCODING = 'UTF8'
LC_COLLATE = 'en_US.UTF-8'
LC_CTYPE = 'en_US.UTF-8'
TABLESPACE = pg_default
CONNECTION LIMIT = -1
IS_TEMPLATE = False;
```
2. Загрузить тестовые данные. Для этого примера создадим `30 000` учетных записей, в среднем по `50` транзакций в каждой.

```sql
psql -f accounts.sql pg_cache_demo
```
3. Выполним обновление строк

```sql
psql -f transactions.sql pg_cache_demo
```
или

```sql
with r as (
  select (random() * 29999)::bigint as account_offset
  from generate_series(1, 1500000)
)
insert into transactions(name, amount, post_time)
select
  (select name from accounts offset account_offset limit 1),
  ((random()-0.5)*1000)::numeric(8,2),
  current_timestamp + '90 days'::interval - (random()*1000 || ' days')::interval
from r
;
```

4. Наш запрос, для которого будем оптимизировать, — это поиск баланса счетов. 

Для начала создадим представление, которое находит балансы для всех счетов. 

Представление PostgreSQL — это сохраненный запрос. 

После создания выбор из представления точно такой же, как и выбор из исходного запроса, т. е. каждый раз выполняется повторный запрос.

```sql
create view account_balances as
select
	name,
	coalesce(
		sum(amount) filter (where post_time <= current_timestamp),
		0
	) as balance
from accounts
	left join transactions using(name)
group by name;
```

5.Теперь просто выбираем все строки с отрицательным балансом.

```sql
select * from account_balances where balance < 0;
```

После нескольких запусков формирования кэша, этот запрос занимает примерно `3000-4000` мс. 
	
Изучим несколько решений по оптимизации транзакционных действий. 

Чтобы сохранить их пространство имен, создадим отдельные схемы для каждого подхода.














