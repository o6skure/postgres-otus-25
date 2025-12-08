# Работа с уровнями изоляции транзакций в PostgreSQL

## Описание задания

Цель — на практике посмотреть, как ведут себя уровни изоляции **READ COMMITTED** (по умолчанию) и **REPEATABLE READ** в PostgreSQL, используя две параллельные сессии и одну таблицу `persons`.

---

## Окружение

- **OS:** macOS
- **Docker:** Docker Desktop
- **PostgreSQL:** `postgres:17` (запуск через `docker-compose`)
- **Пользователь БД:** `user`
- **База данных:** `otus`
- **Подключение клиента:** через `docker exec` внутрь контейнера `db` (две разные сессии — два разных терминала)

---

## Docker Compose

Используем такой `docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:17
    container_name: db
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_PASSWORD=password
      - POSTGRES_USER=user
      - POSTGRES_DB=otus
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d otus || exit 1"]
      interval: 3s
      timeout: 3s
      retries: 20
```

### Запуск СУБД

В каталоге с `docker-compose.yml`:

```bash
docker compose up -d
```

Проверяем, что контейнер поднялся и здоров:

```bash
docker ps
```

Ожидаем увидеть, например:

```
CONTAINER ID IMAGE COMMAND STATUS PORTS NAMES
48416620d89d postgres:17 "docker-entrypoint.s…" Up ... (healthy) 0.0.0.0:5432->5432/tcp db
```

---

## Настройка двух сессий

Дальше всё делаем в двух разных терминалах.

### Сессия 1

```bash
docker exec -it db psql -U user -d otus
```

### Сессия 2

Во втором терминале:

```bash
docker exec -it db psql -U user -d otus
```

Теперь у нас две параллельные сессии `psql`, работающие с одной и той же БД.

---

## Отключение автофиксации транзакций

В обеих сессиях:

```
\set AUTOCOMMIT Off
```

Результат: `AUTOCOMMIT = 'off'`.

Теперь каждый `BEGIN` / `COMMIT` делаем вручную.

---

## Подготовка данных (сессия 1)

В сессии 1:

```sql
CREATE TABLE persons(
    id SERIAL,
    first_name TEXT,
    second_name TEXT
);

INSERT INTO persons(first_name, second_name) VALUES ('ivan', 'ivanov');
INSERT INTO persons(first_name, second_name) VALUES ('petr', 'petrov');

COMMIT;
```

Проверим содержимое таблицы (в любой сессии):

```sql
SELECT * FROM persons;
```

Ожидаемый результат:

| id | first_name | second_name |
|----|------------|-------------|
| 1  | ivan       | ivanov      |
| 2  | petr       | petrov      |

*(2 rows)*

---

## Часть 1. Уровень изоляции по умолчанию — READ COMMITTED

### Проверка уровня изоляции (сессия 1)

```sql
SHOW TRANSACTION ISOLATION LEVEL;
```

Результат: `read committed`.

### Начало транзакций (обе сессии)

В сессии 1:

```sql
BEGIN TRANSACTION;
```

В сессии 2:

```sql
BEGIN TRANSACTION;
```

Возможное предупреждение в сессии 1:  
*WARNING: there is already a transaction in progress*  
*BEGIN*  
(остаток от предыдущей команды — не критично, продолжаем работу.)

### Вставка новой записи в сессии 1 (без коммита)

Сессия 1:

```sql
INSERT INTO persons(first_name, second_name)
VALUES ('sergey', 'sergeev');
```

Транзакция в сессии 1 пока не закоммичена.

### Проверка из сессии 2

Сессия 2:

```sql
SELECT * FROM persons;
```

Ожидаемый результат:

| id | first_name | second_name |
|----|------------|-------------|
| 1  | ivan       | ivanov      |
| 2  | petr       | petrov      |

*(2 rows)*

**Почему не видно sergey?**
- Уровень READ COMMITTED не показывает неподтверждённые изменения (нет доступа к dirty reads).
- Вставка sergey в сессии 1 ещё не прошла COMMIT, поэтому для других транзакций её как бы «не существует».

### Коммит в сессии 1

Сессия 1:

```sql
COMMIT;
```

Теперь запись sergey стала подтверждённой.

### Повторный SELECT в сессии 2 (всё ещё в той же транзакции)

Сессия 2:

```sql
SELECT * FROM persons;
```

Ожидаемый результат:

| id | first_name | second_name |
|----|------------|-------------|
| 1  | ivan       | ivanov      |
| 2  | petr       | petrov      |
| 3  | sergey     | sergeev     |

*(3 rows)*

**Почему теперь видно sergey?**
- В режиме READ COMMITTED снимок данных берётся на каждый запрос, а не на всю транзакцию.
- На момент второго select изменения из сессии 1 уже закоммичены, поэтому они попадают в новый снимок, и строка sergey видна.

### Завершение транзакции в сессии 2

Сессия 2:

```sql
COMMIT;
```

---

## Часть 2. Уровень изоляции REPEATABLE READ

Теперь повторим эксперимент, но с уровнем изоляции REPEATABLE READ.

### Начало транзакций и установка уровня (обе сессии)

Сессия 1:

```sql
BEGIN TRANSACTION;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

Сессия 2:

```sql
BEGIN TRANSACTION;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

Обе транзакции теперь работают в режиме REPEATABLE READ.

### Вставка новой записи в сессии 1 (без коммита)

Сессия 1:

```sql
INSERT INTO persons(first_name, second_name)
VALUES ('sveta', 'svetova');
```

Транзакция в сессии 1 снова не закоммичена.

### Проверка из сессии 2

Сессия 2:

```sql
SELECT * FROM persons;
```

Ожидаемый результат:

| id | first_name | second_name |
|----|------------|-------------|
| 1  | ivan       | ivanov      |
| 2  | petr       | petrov      |
| 3  | sergey     | sergeev     |

*(3 rows)*

**Почему не видно sveta?**
- Как и раньше, незакоммиченные изменения других транзакций не видны.  
  Но здесь важно, что:
- Именно этот select в режиме REPEATABLE READ зафиксировал снимок данных для всей текущей транзакции в сессии 2.

### Коммит в сессии 1

Сессия 1:

```sql
COMMIT;
```

Теперь sveta закоммичена в БД.

### Повторный SELECT в сессии 2 (та же транзакция REPEATABLE READ)

Сессия 2:

```sql
SELECT * FROM persons;
```

Ожидаемый результат не меняется:

| id | first_name | second_name |
|----|------------|-------------|
| 1  | ivan       | ivanov      |
| 2  | petr       | petrov      |
| 3  | sergey     | sergeev     |

*(3 rows)*

**Почему всё ещё не видно sveta?**
- В REPEATABLE READ PostgreSQL использует один и тот же снимок данных на всю транзакцию.
- Снимок был создан при первом select в этой транзакции сессии 2.
- В тот момент sveta ещё не была закоммичена, поэтому в снимок она не попала.
- Все последующие запросы в рамках той же транзакции продолжают работать с этим старым снимком и не видят новые коммиты других транзакций.

### Завершение транзакции в сессии 2

Сессия 2:

```sql
COMMIT;
```

### Новый SELECT в сессии 2 (уже вне старой транзакции)

Сессия 2:

```sql
SELECT * FROM persons;
```

Ожидаемый результат:

| id | first_name | second_name |
|----|------------|-------------|
| 1  | ivan       | ivanov      |
| 2  | petr       | petrov      |
| 3  | sergey     | sergeev     |
| 4  | sveta      | svetova     |

*(4 rows)*

**Почему теперь видно sveta?**
- Предыдущая транзакция REPEATABLE READ завершена.
- Новый select выполняется в новой транзакции, которая создаёт новый снимок данных.
- В этот новый снимок попадают все закоммиченные изменения, включая строку sveta.

---

## Итоговые выводы

1. **READ COMMITTED** (уровень по умолчанию в PostgreSQL):
    - Снимок данных формируется на каждый запрос.
    - Подтверждённые (COMMIT) изменения других транзакций становятся видны сразу при следующем запросе.
    - В рамках одной транзакции разные select могут возвращать разные наборы строк.

2. **REPEATABLE READ**:
    - Снимок данных создаётся один раз — при первом запросе в транзакции.
    - Все последующие запросы в этой транзакции видят один и тот же снимок, даже если другие транзакции уже закоммитили изменения.
    - Это исключает неповторяемые чтения (non-repeatable read), но делает данные «замороженными» на время транзакции.

3. **Эксперимент на таблице persons показывает**:
    - При READ COMMITTED во второй сессии после COMMIT первой сессии становится видна запись sergey в той же транзакции, просто на новом select.
    - При REPEATABLE READ запись sveta не видна во второй сессии до тех пор, пока там не завершится текущая транзакция и не начнётся новая.

Таким образом, мы на практике сравнили:
- модель «снимок на запрос» (READ COMMITTED);
- и модель «снимок на транзакцию» (REPEATABLE READ) в PostgreSQL.