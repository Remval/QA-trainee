<h1 align="center">🐘 PostgreSQL Bare-Metal QA & Security Audit</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Database-PostgreSQL-336791?style=flat-square&logo=postgresql&logoColor=white" alt="PostgreSQL">
  <img src="https://img.shields.io/badge/OS-Ubuntu_24.04-E95420?style=flat-square&logo=ubuntu&logoColor=white" alt="Ubuntu">
  <img src="https://img.shields.io/badge/Security-Schema_Isolation-00A550?style=flat-square&logo=security&logoColor=white" alt="Security">
  <img src="https://img.shields.io/badge/Status-100%25_Passed-brightgreen?style=flat-square" alt="Passed">
</p>

## 📌 Про проект

Цей проект є глибоким QA і Security аудит СУБД PostgreSQL на bare-metal сервері Ubuntu. На відміну від стандартного тестування БД, фокус зроблено на специфічних механізмах захисту PostgreSQL: ізоляції на рівні схем (Schemas), просунутої валідації даних через регулярні вирази (Regex Constraints) на рівні ядра БД та сценаріях післяаварійного відновлення (Disaster Recovery).

**Тестовий стенд:** `Ubuntu 24.04 LTS` | `PostgreSQL 16+` | `Термінал Linux (CLI)`

---

## 📊 Зведення результатів тестування

| Фаза | Тип тестування Опис | Статус |
|:---|:---|:---|:---:|
| **1** | `Deployment & Smoke` | Встановлення СУБД та перевірка роботи системного демона ✅ PASS |
| **2** | `Security/RBAC` | Налаштування Peer/TCP аутентифікації, ізоляція схем даних ✅ PASS |
| **3** | `Data Integrity` | Валідація вхідних даних через Regex CHECK Constraint ✅ PASS |
| **4** | `Disaster Recovery` | Створення дампа (`pg_dump`), симуляція втрати БД та відновлення ✅ PASS |

---

## 🛠 Детальні Тест-кейси та Звіти

### Фаза 1: Встановлення та Smoke Test
> **Мета:** Розгортання PostgreSQL та додаткових утиліт `postgresql-contrib`, перевірка стабільності роботи служби.
* **Результат:** Сервіс успішно запущений, менеджер процесів функціонує штатно (`active (exited)`).
<details> 
<summary>📸 Скріншот: Успішний старт сервісу</summary> 

![PostgreSQL Status](images/pg-status.png) </details>

### Фаза 2: Тестування безпеки (Schema Isolation & RBAC)
> **Мета:** Перевірка неможливості горизонтальної ескалації привілеїв та читання даних із чужих ізольованих схем.

Створено БД `qa_pg_store` з прихованою адміністративною схемою `admin_secrets` та таблицею ключів. Створено обмежений користувач qa_pg_user з доступом тільки на підключення до БД.
* **Тест-кейс (Негативний):** Підключення обмеженим користувачем по TCP/IP (обхід локального Peer auth) та спроба `SELECT` запиту до таблиці `admin_secrets.keys`.
* **Очікуваний / Фактичний результат:** Очікувана відмова у доступі. Отримана помилка `ERROR: permission denied for schema admin_secrets`. Ізоляція схем працює коректно.
<details> 
<summary>📸 Скріншот: Permission Denied (Schema Test)</summary> 

![Access Denied](images/pg-access-denied.png) </details>

### Фаза 3: Просунута цілісність даних (Advanced Data Integrity)
> **Мета:** Використання регулярних виразів (Regex) на стороні БД для захисту від некоректних даних користувача в обхід клієнтської валідації.

Створено таблицю `clients` з обмеженням `CHECK (phone ~ '^\+380[0-9]{9}$')` (приймаються лише українські номери у міжнародному форматі).
1. **Позитивний тест:** Вставка валідного номера `+380501234567` -> Успішно (`INSERT 0 1`).
2. **Негативний тест:** Спроба вставки номера в неправильному форматі `050-123-45-67`.
* **Результат:** БД заблокувала транзакцію, видавши помилку `ERROR: New row for relation "clients" violates check constraint`. Дані надійно захищені лише на рівні БД.
<details> 
<summary>📸 Скріншоти: Спрацювання Regex Constraint</summary> 

![Regex Test](images/pg-regex-test.png) </details>

### Фаза 4: Disaster Recovery (Бекап та Відновлення)
> **Мета:** Перевірка регламенту резервного копіювання та відновлення після катастрофічного збою (DROP DATABASE).
* **Кроки:** 
1. Створення логічного дампа БД за допомогою утиліти `pg_dump`. 
2. Імітація катастрофи: перемикання в БД `postgres` та виконання `DROP DATABASE qa_pg_store;`. Перевірка через 'l'. 
3. Відновлення: створення порожній БД та заливання дампа через CLI.
* **Результат:** База даних повністю відновлена, перевірочний `SELECT` підтвердив наявність раніше внесених даних. Втрат немає.
<details> 
<summary>📸 Скріншоти: DROP DATABASE та Успішне відновлення</summary> 

![Drop DB](images/pg-drop-db.png) ![Recovered DB](images/pg-recovered-db.png) </details>

---
## 🚀 Інструкція з відтворення (Step-by-Step Guide)

Для розгортання стенду та повторення тестів на вашому сервері виконайте такі кроки:

### Крок 1: Встановлення сервера
``` bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
sudo systemctl status postgresql

### Крок 2: Створення схем та налаштування доступів
Заходимо під системним користувачем postgres:

``` bash
sudo -u postgres psql
````
Створюємо базу, ізольовану схему та обмеженого користувача:

``` bash
CREATE DATABASE qa_pg_store;
````
``` bash
\c qa_pg_store
````
``` bash
CREATE SCHEMA admin_secrets;
CREATE TABLE admin_secrets.keys (key_value VARCHAR(100));
CREATE USER qa_pg_user WITH PASSWORD 'TestPass123!';
GRANT CONNECT ON DATABASE qa_pg_store TO qa_pg_user;
````
``` bash
\q
````
### Крок 3: Проведення Security Test (Schema Isolation)
Підключаємося через TCP/IP від ​​обмеженого користувача:

``` bash
psql -h 127.0.0.1 -U qa_pg_user -d qa_pg_store
````
# Пароль: TestPass123!

Намагаємося прочитати секретну схему:
``` bash
SELECT * FROM admin_secrets.keys;
````
-- Очікуємо: ERROR: permission denied for schema admin_secrets

``` bash
\q
````
### Крок 4: Тестування Data Integrity (Regex)
Заходимо під адміном та створюємо таблицю з Regex-перевіркою:
``` bash
sudo -u postgres psql -d qa_pg_store
````
``` bash
CREATE TABLE clients ( 
id SERIAL PRIMARY KEY, 
name VARCHAR(100) NOT NULL, 
phone VARCHAR(15) CHECK (phone ~ '^\+380[0-9]{9}$')
);
GRANT ALL ON clients TO qa_pg_user;
GRANT USAGE ON SEQUENCE clients_id_seq TO qa_pg_user;
````
``` bash
\q
````
Заходимо під QA-користувачем та проводимо тести:
``` bash
psql -h 127.0.0.1 -U qa_pg_user -d qa_pg_store
````
-- Happy Path
``` bash
INSERT INTO clients (name, phone) VALUES ('Ivan', '+380501234567');
````
-- Negative Test (Спрацьовування Regex CHECK)
``` bash
INSERT INTO clients (name, phone) VALUES ('Hacker', '050-123-45-67');
````
``` bash
\q
````
### Крок 5: Імітація Disaster Recovery
Створюємо резервну копію:
``` bash
sudo -u postgres pg_dump qa_pg_store > qa_pg_store_backup.sql
````
Видаляємо базу (імітація втрати):
``` bash
sudo -u postgres psql -d postgres
````
``` bash
DROP DATABASE qa_pg_store;
````
``` bash
\l
````
``` bash
\q
````
Відновлюємо дані з дампа:
``` bash
sudo -u postgres psql -c "CREATE DATABASE qa_pg_store;"
sudo -u postgres psql qa_pg_store <qa_pg_store_backup.sql
````
Перевіряємо відновлені дані:
``` bash
sudo -u postgres psql -d qa_pg_store -c "SELECT * FROM clients;"
````
