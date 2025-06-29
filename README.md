## Репликация и масштабирование. Часть 2

## Домашнее задание к занятию «Репликация и масштабирование. Часть 2»
### Андреев Александр Вадимович

#### Задание 1
##### Опишите основные преимущества использования масштабирования методами:
- активный master-сервер и пассивный репликационный slave-сервер;
- master-сервер и несколько slave-серверов;
##### Дайте ответ в свободной форме.

1) активный master-сервер и пассивный репликационный slave-сервер :

- Высокая доступность (HA): если master выходит из строя, можно вручную или автоматически переключиться на slave
- Резервная копия: slave содержит актуальную копию данных для восстановления
- Без нагрузки на slave: он не обслуживает клиентов, а только реплицирует данные

2) master-сервер и несколько slave-серверов :

- Горизонтальное масштабирование: увеличение числа slave позволяет балансировать нагрузку на чтение
- Разделение функций: например, один slave — для отчетов, другой — для аналитики, третий — для резервной копии
- Повышение отказоустойчивости: если один slave выйдет из строя, другие продолжают работать
- Резервные копии без простоя: можно делать бэкапы с одного из slave

#### Задание 2
##### Разработайте план для выполнения горизонтального и вертикального шаринга базы данных. База данных состоит из трёх таблиц:
 - пользователи,
 - книги,
 - магазины (столбцы произвольно).
##### Опишите принципы построения системы и их разграничение или разбивку между базами данных.
##### Пришлите блоксхему, где и что будет располагаться Опишите, в каких режимах будут работать сервера

** Вертикальное разделение **
- Принцип: Разделить функционал и нагрузку между серверами, основываясь на типах операций или группировке таблиц

DBserv1 (Master) -> Пользователи (Управление аккаунтами, регистрацией, авторизацией)
DBserv2 (Master) -> Книги (Высокая частота обновлений, поиск, фильтрация)
DBserv3 (Master) -> Магазины (Геоданные, информация о наличии товара)
DBslv1	(Slave) -> Пользователи (Только чтение для отчетов, профилей)
DBslv2	(Slave) -> Книги (Чтение для поиска, рекомендаций)
DBslv3	(Slave) -> Магазины (Для аналитики наличия товаров)

```
+-------------------+
|      API / App    |
+---------+---------+
          |
   +------+-------+----------------------+
   |              |                     |
+--v--------+  +--v---------+        +--v---------+
| Роутер    |  |  Роутер    |        | Роутер     |
| (ProxySQL)|  | (ProxySQL) |        | (ProxySQL) |
+-----------+  +------------+        +------------+
        |             |                   |
+-------v--+   +------v-------+   +-------v-----+
| DBserv1  |   |  DBserv2     |   |    DBserv3  |
| (Master) |   |  (Master)    |   |    (Master) |
+----------+   +--------------+   +-------------+
        |             |                   |
+-------v--+   +------v-------+   +-------v--+
|  DBslv1  |   |  DBslv2      |   |   DBslv3 |
| (Slave)  |   |  (Slave)     |   | (Slave)  |
+----------+   +--------------+   +----------+
        |             |                   |
   +----v----+   +----v-----+  +----------v--+
   | Reports  |  | Search   |  | Stock       |
   | Analysis |  | Catalog  |  | Lookup      |
   +----------+  +----------+  +-------------+
```

** Горизонтальное разделение **
- Принцип: Распределить запросы на чтение/запись по разным серверам, чтобы снизить нагрузку на один экземпляр базы данных.

DBserv1 (Master) -> (Обрабатывает все операции записи (INSERT,UPDATE,DELETE))
DBslv1	(Slave) -> (Принимает часть запросов на чтение (SELECT))
DBslv2	(Slave) -> (Дополнительный слейв для чтения или аналитики)

- Все изменения данных (INSERT INTO пользователи..., UPDATE книги SET ...) идут на DB_Master
- Чтение (SELECT * FROM магазины WHERE ...) — распределяется между DB_Slave_1 и DB_Slave_2
- Для аналитики или отчетности можно использовать отдельный slave
- Можно использовать роутер / прокси (например, ProxySQL), чтобы автоматически направлять запросы

```
+-------------------+
|      API / App    |
+---------+---------+
          |
   +------v---------+---------------------------+
   |                |                           |
   |    Роутер      |                           |
   |  (ProxySQL)    |                           |
   +-------+--------+                           |
           |                                    |
+----------v----------+                 +-------v-------------+
|   DBserv1           |                 |   DBslv1            |
|  (Master)           | <-------------+ |  (Slave)            |
| Обработка записи:   |   Репликация    | Обработка чтения:   |
| INSERT, UPDATE,     |                 | SELECT              |
| DELETE              |                 |                     |
+---------------------+                 +---------------------+
                                           |
                                           v
                                 +--------------------------+
                                 |    DBslv2                |
                                 |   (Slave)                |
                                 |  Для аналитики / отчетов |
                                 +--------------------------+
```




