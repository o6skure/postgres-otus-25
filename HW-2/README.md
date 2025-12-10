# PostgreSQL 15 в Docker с сохранением данных при пересоздании контейнера

## Цель

Цель — на практике убедиться, что данные PostgreSQL **сохраняются** при удалении и пересоздании контейнера, если каталог с данными вынесен на хост и смонтирован в контейнер.

---

## Окружение

- **OS:** macOS
- **Docker:** Docker Desktop
- **PostgreSQL:** образ `postgres:15`
- **Пользователь БД:** `user`
- **База данных:** `otus`
- **Каталог с данными в репозитории:** `./var/lib/postgres` (относительно `docker-compose.yml`)

---

### Установка Docker на macOS

Docker Desktop для macOS устанавливается по официальной инструкции:  
[https://docs.docker.com/desktop/setup/install/mac-install/](https://docs.docker.com/desktop/setup/install/mac-install/)

После установки проверьте, что Docker работает:

```bash
docker ps
```

---

### Docker Compose

Файл `docker-compose.yml` (находится в корне `HW-2/`):

```yaml
services:
  pg-server:
    image: postgres:15
    container_name: pg-server
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_PASSWORD=password
      - POSTGRES_USER=user
      - POSTGRES_DB=otus
    volumes:
      - ./var/lib/postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d otus || exit 1"]
      interval: 3s
      timeout: 3s
      retries: 20
  pg-client:
    image: postgres:15
    container_name: pg-client
    depends_on:
      - pg-server
    stdin_open: true
    tty: true
    command: ["sleep", "infinity"]
```

- `pg-server` — контейнер с сервером PostgreSQL 15.
- `pg-client` — контейнер-клиент, в который будем заходить и запускать `psql`.
- `./var/lib/postgres:/var/lib/postgresql/data` — монтируем локальный каталог репозитория в стандартный каталог данных Postgres внутри контейнера.

---

### 1. Запуск контейнеров

Из каталога `HW-2/`:

```bash
docker compose up -d
```

Проверяем:

```bash
docker ps
```

Должно быть примерно так:

```
CONTAINER ID   IMAGE         COMMAND                  STATUS                   PORTS                    NAMES
...            postgres:15   "docker-entrypoint.s…"   Up ... (healthy)         0.0.0.0:5432->5432/tcp   pg-server
...            postgres:15   "sleep infinity"         Up ...                                            pg-client
```

---

### 2. Подключение из контейнера-клиента и создание таблицы

Заходим в клиент:

```bash
docker exec -it pg-client bash
```

Внутри контейнера запускаем `psql`, подключаясь к `pg-server` по имени сервиса:

```bash
psql -h pg-server -U user -d otus
```

Пароль: `password`.

Создаём таблицу и вставляем пару строк:

```sql
create table demo_data (
    id serial primary key,
    name text
);

insert into demo_data (name) values
  ('first row'),
  ('second row');

select * from demo_data;
```

Ожидаемый результат:

```
 id |     name
----+-------------
  1 | first row
  2 | second row
(2 rows)
```

Выходим из `psql`:

```
\q
```

И, при необходимости, из контейнера:

```
exit
```

---

### 3. Подключение к серверу с ноутбука/компьютера

Так как Docker Desktop работает на этой же macOS-машине, подключаемся к `localhost`:

```bash
psql -h localhost -p 5432 -U user -d otus
```

Проверяем:

```sql
select * from demo_data;
```

Ожидаемый результат — те же две строки.

---

### 4. Удаление контейнера с сервером

Удаляем только контейнер `pg-server`:

```bash
docker rm -f pg-server
```

Проверяем:

```bash
docker ps
```

Контейнер `pg-client` ещё работает, а `pg-server` удалён.  
Данные при этом остаются в каталоге `HW-2/var/lib/postgres` в репозитории.

---

### 5. Создание контейнера с сервером заново

Пересоздаём только сервис `pg-server`:

```bash
docker compose up -d pg-server
```

Проверяем:

```bash
docker ps
```

`pg-server` снова должен быть в статусе `Up (healthy)`.

---

### 6. Повторная проверка данных из контейнера-клиента

Снова заходим в `pg-client`:

```bash
docker exec -it pg-client bash
```

```bash
psql -h pg-server -U user -d otus
```

Проверяем:

```sql
select * from demo_data;
```

Ожидаемый результат:

```
 id |     name
----+-------------
  1 | first row
  2 | second row
(2 rows)
```

Данные сохранились, хотя контейнер с сервером мы удаляли и создавали заново — благодаря тому, что каталог с данными (`./var/lib/postgres`) вынесен из контейнера и хранится в репозитории (на хосте).

---

## Итог

- Используем два сервиса: `pg-server` (сервер БД) и `pg-client` (клиент).
- Данные хранятся в `HW-2/var/lib/postgres` и монтируются в контейнер `pg-server`.
- После удаления и пересоздания `pg-server` данные в таблице `demo_data` остаются на месте.
