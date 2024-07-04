---
title: Триггер для last update timestamp в PostgreSQL
slug: psql-last-update
date: 2024-07-04
tags:
  - postgresql
  - note
description: Заметка о том, как настроить управление полем last_update timestamp в PostgreSQL
keywords:
  - postgresql
  - trigger
  - psql
---

Если вам нужно настроить автоматическое сохранение timestamp-а последнего обновления записи в таблице, эта заметка для вас.
Расскажу о том, как реализовать такую логику в PostgreSQL при помощи триггеров.

## Создаём таблицу

Для примера создадим таблицу:

```sql
CREATE TABLE test (
	id serial PRIMARY KEY,
	value varchar(10) UNIQUE,
	last_update timestamp DEFAULT CURRENT_TIMESTAMP
);
```

- `id` — Id записи
- `value` — полезная нагрузка записи
- `last_update` — поле, в котором хранится timestamp последнего обновления записи

`CURRENT_TIMESTAMP` — функция, возвращающая текущее время по UTC на момент её вызова.

При добавлении новых записей в таблицу им будет автоматически проставляться timestamp, в который они были добавлены.

## Создаём триггерную функцию

Прежде, чем создать сам триггер, необходимо написать функцию, которая будет им вызываться. В нашем случае эта функция должна обновлять поле `last_update` по заданным правилам.

Возможно несколько вариантов реализации такой функции в зависимости от желаемого поведения.

### Без возможности ручного обновления

```sql
CREATE OR REPLACE FUNCTION set_last_update()
RETURNS trigger AS $$
BEGIN
	NEW.last_update := CURRENT_TIMESTAMP;
	RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

Если связать такую функцию с триггером, вызывающимся при любом обновлении записи в таблице `test`, значение поля `last_update` нельзя будет установить вручную UPDATE-запросом.

Перед записью в таблицу триггерная функция затрет значение из UPDATE-запроса значением `CURRENT_TIMESTAMP`.

**Результат:**
- Если запрос содержал изменение поля `last_update`, в это поле будет записан `CURRENT_TIMESTAMP`
- Если запрос не содержал изменение поля `last_update`, в это поле будет записан `CURRENT_TIMESTAMP`

### С возможностью ручного обновления

```sql
CREATE OR REPLACE FUNCTION set_last_update()  
RETURNS trigger AS $$  
BEGIN
	IF OLD.last_update = NEW.last_update THEN
		NEW.last_update := CURRENT_TIMESTAMP;
	END IF; 
	RETURN NEW;
END;
$$ LANGUAGE plpgsql; 
```

В таком случае если значение поля `last_update` изменяется в UPDATE-запросе, триггерная функция не будет затирать указанное значение.

**Результат:**
- Если запрос содержал изменение поля `last_update`, указанное в запросе значение будет записано в БД
- Если запрос не содержал изменение поля `last_update`, в это поле будет записан `CURRENT_TIMESTAMP`


### С запретом ручного обновления

```sql
CREATE OR REPLACE FUNCTION set_last_update()  
RETURNS trigger AS $$  
BEGIN
	IF OLD.last_update != NEW.last_update THEN
        RAISE EXCEPTION 'last_update field cannot be updated by SQL request';
	END IF; 
	NEW.last_update := CURRENT_TIMESTAMP;
	RETURN NEW;
END;
$$ LANGUAGE plpgsql; 
```

В таком случае при попытке обновить поле вручную будет вызываться исключение. 

**Результат:**
- Если запрос содержал изменение поля `last_update`, он не будет выполнен
- Если запрос не содержал изменение поля `last_update`, в это поле будет записан `CURRENT_TIMESTAMP`

## Создаём триггер

```sql
CREATE TRIGGER test_last_update_trigger
BEFORE UPDATE
ON test
FOR EACH ROW
EXECUTE PROCEDURE set_last_update();
```

**Как это работает:**

При выполнении UPDATE-запроса, перед модификацией таблицы `test`, для каждой изменяемой записи будет вызвана триггерная функция `set_last_update`, созданная нами ранее. Значение, которое будет записано в `last_update`, определяется запросом и реализацией этой функции.

Подробнее логика работы триггеров и триггерных функций описана в [официальной документации](https://www.postgresql.org/docs/current/plpgsql-trigger.html) и [документации PostgresPro](https://postgrespro.ru/docs/postgresql/15/plpgsql-trigger)

## Итог

Таким образом можно организовать управление значением поля `last_update` на уровне СУБД PostgreSQL.

Почему это полезно:
- При реализации клиентов БД не нужно думать об обновлении поля `last_update` при выполнении UPDATE-запросов
- Отсутствует риск случайно забыть обновить значение поля или установить некорректное значение
- Незначительно уменьшается объем данных, передаваемых по сети при выполнении запросов: нет необходимости передавать значение поля `last_update` в каждом UPDATE-запросе
