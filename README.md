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

После нескольких запусков формирования кэша, этот запрос занимает примерно `30-40` мс. 

Получается для 1500000 записей(стандартная БД) запрос будет длиться порядка `1500-2000` мс.

	
Изучим несколько решений по оптимизации транзакционных действий. 

Чтобы сохранить их пространство имен, создадим отдельные схемы для каждого подхода.

```sql
create schema matview;
create schema eager;
create schema lazy;
```

## Материализованные представления PostgreSQL

Простейший способ повысить производительность — использовать `материализованное представление`. 	

**Материализованное представление** — это снимок запроса, сохраненный в таблице.

```sql
create materialized view matview.account_balances as
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
Поскольку `материализованное представление` на самом деле представляет собой таблицу, можем создавать индексы.

```sql
create index on matview.account_balances (name);
create index on matview.account_balances (balance);
```
Чтобы получить баланс из каждой строки, выбираем из материализованного представления необходимые данные.

```sql
select * from matview.account_balances where balance < 0;
```
Время выполнения запроса для `30000` записей около `3-7` мс, что в **10** раз быстрее обычного запроса.

К сожалению, у этих материализованных представлений есть два существенных недостатка. 

**Во-первых**, они обновляются только по требованию. 

**Во-вторых**, необходимо обновить все материализованное представление; нет способа обновить только одну устаревшую строку.
```sql
--обновить все строки
refresh materialized view matview.account_balances;
```
В случае, когда допустимы устаревшие данные, этот метрод является отличным решением. Но если данные всегда должны быть актуальными, требуется другой метод оптимизации.

## Жадный метод.

 Следующий подход состоит в том, чтобы материализовать запрос в виде таблицы, которая с обновляется всякий раз, когда происходит изменение, которое делает строку неактуальной. Метод использует `триггер`. 

**Триггер** — это фрагмент кода, который запускается, когда происходит какое-либо событие, такое как вставка или обновление. 

1.  Создать таблицу для хранения материализованных строк.
```sql
create table eager.account_balances(
	name varchar primary key references accounts
		on update cascade
		on delete cascade,
	balance numeric(9,2) not null default 0
);

create index on eager.account_balances (balance);
```

2. Рассмотреть методы, на основании которых данные account_balances станут не актуальны или изменятся.

- Создана новая учетная запись.
- Учетная запись обновлена или удалена.
- Транзакция вставлена, обновлена или удалена.

### Создана новая учетная запись.

При вставке учетной записи нам необходимо создать запись `account_balances` с нулевым балансом для новой учетной записи.

```sql
create function eager.account_insert() returns trigger
	security definer
	language plpgsql
as $$
	begin
		insert into eager.account_balances(name) values(new.name);
		return new;
	end;
$$;

create trigger account_insert after insert on accounts
	for each row execute procedure eager.account_insert();
```


### Учетная запись обновлена или удалена.

Обновление и удаление учетной записи будут обрабатываться автоматически, поскольку внешний ключ учетной записи объявляется как при каскадном обновлении при каскадном удалении.

###Транзакция вставлена, обновлена или удалена.

Вставка, обновление и удаление транзакций имеют одну общую черту: они делают недействительным баланс счета. Поэтому первым шагом является определение функции обновления баланса учетной записи.
```sql
create function eager.refresh_account_balance(_name varchar)
	returns void
	security definer
	language sql
as $$
	update eager.account_balances
	set balance=
		(
			select sum(amount)
			from transactions
			where account_balances.name=transactions.name
			and post_time <= current_timestamp
		)
	where name=_name;
$$;
```
Далее создать **триггер-функцию**, которая вызывает `refresh_account_balance` всякий раз, когда вставляется транзакция.


















