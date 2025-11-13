Кейс по системной аналитике - блокировка платежей юридическим лицами
---
## API
[Спецификация OpenAPI/Swagger](https://github.com/yn96k/client-block-service/blob/main/client-block-service-v1.openapi.yaml)
### 1. Блокировка клиента
POST /clients/{client_id}/block <br>
Параметры requestbody:
- block_type_name - Название причины блокировки
- comment - Комментарий модератора

Алгоритм сервиса при вызове запроса:
- Обновляет поле is_blocked в таблице client:
```sql
UPDATE client
SET is_blocked = TRUE
WHERE client_id = {client_id}
```
- Добавляет строку в client_block:
```sql
INSERT INTO client_block(client_id, block_type_id, comment)
VALUES({client_id, {block_type_id}, {comment})
```
### 2. Разблокировка клиента
POST /clients/{client_id}/unblock

Алгоритм сервиса при вызове запроса:
- Обновляет поле is_blocked в таблице client:
```sql
UPDATE client
SET is_blocked = FALSE
WHERE client_id = {client_id}
```
- Удаляет строку в client_block:
```sql
DELETE FROM client_block
WHERE client_id = {client_id}
```

### 3. Просмотр статуса блокировки
GET /clients/{client_id}/block_status

Ответ сервера:
- client_id - UUID клиента
- is_blocked - статус блокировки
- block_type - причина блокировки (опицональное поле) 
- blocked_at - дата и время блокировки (опицональное поле)
- comment - комментарий модератора (опицональное поле)

Алгоритм сервиса при вызове запроса:
- Запрашивает выборку из client_block:

```sql
SELECT client_id, block_type_id, blocked_at, comment
WHERE client_id = {client_id}
```

- Возвращает client_id и is_blocked = False если получает пустую выборку

## База данных
### 1. Таблица client
   Хранит информацию о клиенте, его основные признаки. Предлагается в рамках реализаций функций блокировки добавить атрибут статуса блокировки is_blocked:
   
|Имя атрибута|Тип данных|Описание|
|:----|:----|:----|
|client_id|VARCHAR(36) PK|UUID клиента|
|client_name|VARCHAR(30) NOT NULL|Название юридического лица|
|ogrn|DECIMAL(13) NOT NULL|ОГРН организации|
|block_type|VARCHAR(30) NOT NULL|Название причины блокировки|
|blocked_at|TIMESTAMP NOT NULL|Дата и время блокировки|

Так как запросы приходят по client_id, целесообразно индексировать это поле:

|Индекс|Тип|Столбцы|Порядок|
|:----|:----|:----|:----|
|idx_client_client_id|UNIQUE|client_id|ASC|

### 2. Таблица block_type
Таблица-словарь типов (причин) блокировки. Отдельная таблица вместо enum позволит гибко изменять список причин.
   
|Имя атрибута|Тип данных|Описание|
|:----|:----|:----|
|block_type_id|INT PK AUTO_INCREMENT|ID причины блокировки|
|block_type_name|VARCHAR(30) NOT NULL|Название причины блокировки|


### 3. Таблица client_block
Хранит список всех заблокированных на данный момент клиентов. Если клиент разблокирован, строка с его UUID удаляется из таблицы.
   
|Имя атрибута|Тип данных|Описание|
|:----|:----|:----|
|client_block_id|INT PK AUTO_INCREMENT|ID блокировки|
|client_id|CHAR(36) FK client(client_id) UNIQUE NOT NULL|UUID клиента|
|block_type_id|INT FK block_type(block_type_id) NOT NULL|ID причины блокировки|
|blocked_at|TIMESTAMP NOT NULL|Дата и время блокировки|
|comment|VARCHAR(50)|Комментарий модератора. Поле позволяет хранить специфичные детали о конкретной блокировке|

Здесь по аналогичным причинам нужно индексировать поле client_id:

|Индекс|Тип|Столбцы|Порядок|
|:----|:----|:----|:----|
|idx_client_block_client_id|UNIQUE|client_id|ASC|

### ER-диаграмма таблиц:
<img width="4287" height="1651" alt="image" src="https://github.com/user-attachments/assets/355b8fde-79b7-4146-bd7f-92b54705839b" />


