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
psql -f transactions.sql pg_cache_demo
psql -f postgresql_view.sql pg_cache_demo
psql -f postgresql_mat_view.sql pg_cache_demo
psql -f eager_mat_view.sql pg_cache_demo
psql -f lazy_mat_view.sql pg_cache_demo
```














