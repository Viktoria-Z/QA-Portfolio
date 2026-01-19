# WTP API — SQL‑проверки (документ для портфолио)
 
Этот документ показывает, как я подтверждаю результаты API‑запросов не только по ответу сервера, но и по фактическим данным в PostgreSQL.
 
Формат: **что сделать в Postman → какой SQL выполнить → что должно получиться**.
 
## Окружение
 
- БД: PostgreSQL (контейнер в составе стенда)
- Инструмент для SQL: pgAdmin 4 или psql
- Таблицы:
  - `users(id, name, email, password, role_id)`
  - `tokens(id, user_id, token, expire_at)`

**Скриншоты:**
- [docker-running.png](../screenshots/docker-running.png)
- [collection-structure.png](../screenshots/collection-structure.png)
 
## Тестовые данные, которые использую в примерах
 
- Для сценариев с пользователем (create/update/delete) использую email:
  - `qa.portfolio.wtp.user@example.com`
  Если выполняешь сценарии повторно и email уже занят — просто возьми другой (например, добавь цифру).
- Для сценария логина использую пользователя из сидов БД:
  - `john@example.com` / `password123`

---

## 1) POST `/users` → пользователь создался
 
**Postman:** `POST /users` (создать пользователя).

**Скриншот:** [сreate-user-request.png](../screenshots/сreate-user-request.png)
 
**SQL:** проверить, что пользователь появился в `users`.

**Скриншот (SQL):** [sql_user_created.png](../screenshots/sql_user_created.png)
 
```sql
SELECT id, name, email, role_id
FROM users
WHERE email = 'qa.portfolio.wtp.user@example.com';
```
 
**Ожидаемо:** 1 строка.
 
**SQL (нужно для следующих шагов):** узнать `id` этого пользователя.

**Скриншот (SQL):** [sql_user_id.png](../screenshots/sql_user_id.png)
 
```sql
SELECT id
FROM users
WHERE email = 'qa.portfolio.wtp.user@example.com';
```

---

## 2) POST `/users` (дубль email) → новый пользователь не создаётся
 
**Postman:** повторить `POST /users` с тем же `email` (ожидается ошибка).
 
**SQL:** убедиться, что дубль не появился.

**Скриншот (SQL):** [sql_duplicate_email_count.png](../screenshots/sql_duplicate_email_count.png)
 
```sql
SELECT email, COUNT(*)
FROM users
WHERE email = 'qa.portfolio.wtp.user@example.com'
GROUP BY email;
```
 
**Ожидаемо:** `COUNT(*) = 1`.

---

## 3) GET `/users` → сверка с БД + проверка бага `role_id`

**Postman:** `GET /users`.

**Скриншот:** [get-users-response.png](../screenshots/get-users-response.png)

**SQL (количество в БД):**

**Скриншот (SQL):** [sql_users_count.png](../screenshots/sql_users_count.png)

```sql
SELECT COUNT(*) AS users_count
FROM users;
```

**Ожидаемо:** количество элементов в ответе API = `users_count`.

**SQL (роль в БД — число):**

**Скриншот (SQL):** [sql_role_id_values.png](../screenshots/sql_role_id_values.png)
 
```sql
SELECT id, role_id
FROM users
ORDER BY id;
```
 
**SQL (тип поля role_id в БД):**

**Скриншот (SQL):** [sql_role_id_type.png](../screenshots/sql_role_id_type.png)
 
```sql
SELECT data_type
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name = 'users'
  AND column_name = 'role_id';
```

**Ожидаемо:** в БД `role_id` — число (`INTEGER`).
Если в API приходит `role_id: [4]` — это баг.

---
 
## 4) Массовое создание пользователей (Postman Runner) → записи создались
 
**Postman:** я запускала коллекцию на несколько итераций (Runner), чтобы создать несколько пользователей.

**Скриншот:** [run-results.png](../screenshots/run-results.png)

 Так как `email` генерировался рандомно, я подтверждала массовое создание через сравнение количества строк в таблице `users` **до** и **после** прогона Runner.

 **SQL (до Runner):**

 **Скриншот (SQL):** [sql_users_count_before_runner.png](../screenshots/sql_users_count_before_runner.png)

 ```sql
 SELECT COUNT(*) AS users_count_before
 FROM users;
 ```

 **SQL (после Runner):**

 **Скриншот (SQL):** [sql_users_count_after_runner.png](../screenshots/sql_users_count_after_runner.png)

 ```sql
 SELECT COUNT(*) AS users_count_after
 FROM users;
 ```

 **Ожидаемо:** разница `users_count_after - users_count_before` совпадает с количеством успешных итераций в Runner.

 **SQL (дополнительно, наглядно):** я также фиксировала максимальный `id` до/после, чтобы видеть диапазон новых записей.

 **Скриншот (SQL):** [sql_users_max_id_before_after_runner.png](../screenshots/sql_users_max_id_before_after_runner.png)

 ```sql
 SELECT MAX(id) AS max_id
 FROM users;
 ```

---

## 5) DELETE `/users/{id}` → запись удалена
 
**Postman:** после удаления пользователя через `DELETE /users/{id}` я проверяла состояние БД.
 
**SQL:** пользователя больше нет в таблице `users`.

**Скриншот (SQL):** [sql_user_deleted_cnt.png](../screenshots/sql_user_deleted_cnt.png)
 
```sql
SELECT COUNT(*)
FROM users
WHERE email = 'qa.portfolio.wtp.user@example.com';
```
 
**Ожидаемо:** `COUNT(*) = 0`.
 
---

## 6) GET `/login` → токен записался в `tokens`
 
**Postman:** при успешной авторизации через `GET /login` (email/пароль) я подтверждала, что токен записался в БД.

**Скриншот:** [get-token-response.png](../screenshots/get-token-response.png)
 
**SQL:** у пользователя `john@example.com` появилась новая запись в `tokens`.

**Скриншот (SQL):** [sql_tokens_last3.png](../screenshots/sql_tokens_last3.png)
 
```sql
SELECT id, user_id, token, expire_at
FROM tokens
WHERE user_id = (
  SELECT id
  FROM users
  WHERE email = 'john@example.com'
)
ORDER BY id DESC
LIMIT 3;
```
 
**Ожидаемо:** сверху списка появляется новая запись, и `expire_at` больше текущего времени.
 
**SQL (дополнительно):** проверка “TTL” токена (время до истечения).

**Скриншот (SQL):** [sql_token_ttl.png](../screenshots/sql_token_ttl.png)
 
```sql
SELECT
  expire_at,
  NOW() AS now,
  expire_at - NOW() AS ttl
FROM tokens
WHERE user_id = (
  SELECT id
  FROM users
  WHERE email = 'john@example.com'
)
ORDER BY id DESC
LIMIT 1;
```

---
 
## Опционально (SQL‑наблюдения по безопасности)
 
### A1) Пароли в открытом виде
 
Я проверяла, как именно пароли хранятся в таблице `users`.

**Скриншот (SQL):** [sql_passwords_plaintext.png](../screenshots/sql_passwords_plaintext.png)
 
```sql
SELECT id, email, password
FROM users
ORDER BY id;
```

### A2) Просроченные токены
 
Я проверяла, есть ли в таблице `tokens` записи с истёкшим сроком действия.

**Скриншот (SQL):** [sql_expired_tokens.png](../screenshots/sql_expired_tokens.png)
 
```sql
SELECT id, user_id, expire_at
FROM tokens
WHERE expire_at < NOW()
ORDER BY expire_at ASC;
```
---

## Итог

В этом документе я показываю, как подтверждала результаты API‑проверок через PostgreSQL:

- после `POST /users` проверяла, что запись появилась в таблице `users`;
- для `GET /users` сравнивала данные API с БД и зафиксировала дефект: `role_id` приходит как массив (`[4]`), хотя в БД это числовое поле;
- после `DELETE /users/{id}` подтверждала удаление записи через `COUNT(*) = 0`;
- после `GET /login` проверяла, что токен создаётся в `tokens` и `expire_at` больше текущего времени;
- дополнительно фиксировала SQL‑наблюдения по security (пароли в открытом виде, просроченные токены — если присутствуют).

Скриншоты (Postman/SQL) приложены в папке [screenshots](wtp-api-testing/screenshots).
