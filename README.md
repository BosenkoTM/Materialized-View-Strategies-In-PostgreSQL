# Стратегии материализованного представления в PostgreSQL

Запросы, возвращающие совокупные, сводные и вычисленные данные, часто используются при разработке приложений. 
	Иногда эти запросы выполняются недостаточно быстро. Кэширование результатов запроса с помощью Memcached или Redis — распространенный подход к решению этих проблем с производительностью. 
	Однако они приносят свои собственные проблемы. Прежде чем обратиться к внешнему инструменту, стоит изучить, какие методы предлагает PostgreSQL для кэширования результатов запросов.

## Пример БД
Рассмотрим подходы, используя пример домена(бд) упрощенной системы учета. 

	- Счета могут иметь много транзакций. 
  
	- Транзакции могут быть записаны заранее и вступят в силу только после их завершения. 
  
	- Дебет, вступивший в силу 9 марта, можно ввести в марте.

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
