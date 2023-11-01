# Подготовительный этап
___
**Создадим новый кластер и развернем представленные таблицы товаров и продаж**
```commandline
~$ sudo -u postgres pg_createcluster 15 main
~$ sudo pg_ctlcluster 15 main start
~$ sudo -u postgres psql -c "CREATE TABLE goods
> (
>     goods_id    integer PRIMARY KEY,
>     good_name   varchar(63) NOT NULL,
>     good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
> );"
CREATE TABLE
~$ sudo -u postgres psql -c "CREATE TABLE sales
> (
>     sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
>     good_id     integer REFERENCES goods (goods_id),
>     sales_time  timestamp with time zone DEFAULT now(),
>     sales_qty   integer CHECK (sales_qty > 0)
> );"
CREATE TABLE
```
**Наполним таблицы данными из задания**
```commandline
~$ sudo -u postgres psql -c "INSERT INTO goods (goods_id, good_name, good_price)
> VALUES (1, 'Спички хозайственные', .50),
> (2, 'Автомобиль Ferrari FXX K', 185000000.01);"
INSERT 0 2
~$ sudo -u postgres psql -c "INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);"
INSERT 0 4
```
**Проверим, что отчет строится и создадим новую таблицу, для заполнения которой и будем далее писать триггер**
```commandline
~$ sudo -u postgres psql -c "SELECT G.good_name, sum(G.good_price * S.sales_qty)
> FROM goods G
> INNER JOIN sales S ON S.good_id = G.goods_id
> GROUP BY G.good_name;"
        good_name         |     sum
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

~$ sudo -u postgres psql -c "CREATE TABLE good_sum_mart
> (
> good_name   varchar(63) NOT NULL,
> sum_sale numeric(16, 2) NOT NULL
> );"
CREATE TABLE
```
___
# Выполнение задания
- [x] Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии 
(вычисляющий при каждой продаже сумму и записывающий её в витрину)
___
**Предполагаем, что раз по условию задачи триггер требуется только на таблицу продаж, то базовый контроль
наполнение доп таблицы `good_sum_mart` осуществляется вручную при добавлении/удалении/изменении товара**

**Для создания триггера сперва опишем триггерную функцию**
```commandline
~$ sudo su postgres
@pgotus:~$ psql
=# CREATE OR REPLACE FUNCTION tf_edit_good_sum_mart()
RETURNS trigger
AS
$$
DECLARE
    row_dbg record;
BEGIN
    CASE TG_OP
		WHEN 'DELETE' THEN
            UPDATE good_sum_mart SET sum_sale = 0 
				WHERE good_name = (SELECT good_name FROM goods WHERE goods_id=OLD.good_id);

            RETURN OLD;
    ELSE 
				UPDATE good_sum_mart SET sum_sale = (SELECT 
                        coalesce((SELECT sum(G.good_price * S.sales_qty)              
                        FROM goods G
                        INNER JOIN sales S ON S.good_id = G.goods_id
                        WHERE G.goods_id=NEW.good_id), 0::integer)) 
				WHERE good_name = (SELECT good_name FROM goods WHERE goods_id=NEW.good_id); 
		RETURN NEW;
	END CASE;
END
$$ LANGUAGE plpgsql;
CREATE FUNCTION
```
**Создадим сам триггер**
```commandline
=# CREATE TRIGGER trg_edit_good_sum_mart
-# AFTER INSERT OR UPDATE OR DELETE
-# ON sales
-# FOR EACH ROW
-# EXECUTE FUNCTION tf_edit_good_sum_mart();
CREATE TRIGGER
```

**Добавим необходимые базовые данные в доп таблицу и проверим как считается сумма при изменении данных в таблице продаж**

```commandline
=# INSERT INTO good_sum_mart (good_name, sum_sale) VALUES ('Автомобиль Ferrari FXX K',185000000.01), ('Спички хозайственные',65.50);
INSERT 0 2
=# SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 20), (2, 3);       
INSERT 0 2

=# SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Спички хозайственные     |        75.50
 Автомобиль Ferrari FXX K | 740000000.04
(2 rows)

=# UPDATE sales SET sales_qty = 1 WHERE good_id=2;
UPDATE 2
postgres=# SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Спички хозайственные     |        75.50
 Автомобиль Ferrari FXX K | 370000000.02
(2 rows)

=# DELETE FROM sales WHERE good_id=2;
DELETE 1
=# SELECT * FROM good_sum_mart;
        good_name         | sum_sale
--------------------------+----------
 Спички хозайственные     |   155.50
 Автомобиль Ferrari FXX K |     0.00
(2 rows)
```
___

# Задание со звездочкой*
- [x] Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?

___
**Схема предоставления данных через сводную таблицу предпочтительнее из таких соображений:**
- Данные для формирования отчета считаются в момент запроса, что может сильно нагрузить БД
- Если дать доступ только к гибридной таблице, которую обновляет триггер, то тот, кому предоставлен такой доступ не 
сможет увидеть структуру основных таблиц (Б - безопасность)