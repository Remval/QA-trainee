<h1 align="center">🐘 PostgreSQL Bare-Metal QA & Security Audit</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Database-PostgreSQL-336791?style=flat-square&logo=postgresql&logoColor=white" alt="PostgreSQL">
  <img src="https://img.shields.io/badge/OS-Ubuntu_24.04-E95420?style=flat-square&logo=ubuntu&logoColor=white" alt="Ubuntu">
  <img src="https://img.shields.io/badge/Security-Schema_Isolation-00A550?style=flat-square&logo=security&logoColor=white" alt="Security">
  <img src="https://img.shields.io/badge/Status-100%25_Passed-brightgreen?style=flat-square" alt="Passed">
</p>

## 📌 О проекте

Этот проект представляет собой глубокий QA и Security аудит СУБД PostgreSQL на bare-metal сервере Ubuntu. В отличие от стандартного тестирования БД, фокус сделан на специфических механизмах защиты PostgreSQL: изоляции на уровне схем (Schemas), продвинутой валидации данных через регулярные выражения (Regex Constraints) на уровне ядра БД и сценариях послеаварийного восстановления (Disaster Recovery).

**Тестовый стенд:** `Ubuntu 24.04 LTS` | `PostgreSQL 16+` | `Терминал Linux (CLI)`

---

## 📊 Сводка результатов тестирования

| Фаза | Тип тестирования | Описание | Статус |
|:---|:---|:---|:---:|
| **1** | `Deployment & Smoke` | Установка СУБД и проверка работы системного демона | ✅ PASS |
| **2** | `Security / RBAC` | Настройка Peer/TCP аутентификации, изоляция схем данных | ✅ PASS |
| **3** | `Data Integrity` | Валидация входных данных через Regex CHECK Constraint | ✅ PASS |
| **4** | `Disaster Recovery` | Создание дампа (`pg_dump`), симуляция потери БД и восстановление | ✅ PASS |

---

## 🛠 Подробные Тест-кейсы и Отчеты

### Фаза 1: Установка и Smoke Test
> **Цель:** Развертывание PostgreSQL и дополнительных утилит `postgresql-contrib`, проверка стабильности работы службы.
* **Результат:** Сервис успешно запущен, менеджер процессов функционирует штатно (`active (exited)`).
<details>
  <summary>📸 Скриншот: Успешный старт сервиса</summary>
  
  ![PostgreSQL Status](images/pg-status.png) </details>

### Фаза 2: Тестирование безопасности (Schema Isolation & RBAC)
> **Цель:** Проверка невозможности горизонтальной эскалации привилегий и чтения данных из чужих изолированных схем.

Создана БД `qa_pg_store` со скрытой административной схемой `admin_secrets` и таблицей ключей. Создан ограниченный пользователь `qa_pg_user` с доступом только на подключение к БД.
* **Тест-кейс (Негативный):** Подключение ограниченным пользователем по TCP/IP (обход локального Peer auth) и попытка `SELECT` запроса к таблице `admin_secrets.keys`.
* **Ожидаемый / Фактический результат:** Ожидаемый отказ в доступе. Получена ошибка `ERROR: permission denied for schema admin_secrets`. Изоляция схем работает корректно.
<details>
  <summary>📸 Скриншот: Permission Denied (Schema Test)</summary>
  
  ![Access Denied](images/pg-access-denied.png) </details>

### Фаза 3: Продвинутая целостность данных (Advanced Data Integrity)
> **Цель:** Использование регулярных выражений (Regex) на стороне БД для защиты от некорректных пользовательских данных в обход клиентской валидации.

Создана таблица `clients` с ограничением `CHECK (phone ~ '^\+380[0-9]{9}$')` (принимаются только украинские номера в международном формате).
1. **Позитивный тест:** Вставка валидного номера `+380501234567` -> Успешно (`INSERT 0 1`).
2. **Негативный тест:** Попытка вставки номера в неверном формате `050-123-45-67`.
* **Результат:** БД заблокировала транзакцию, выдав ошибку `ERROR: new row for relation "clients" violates check constraint`. Данные надежно защищены на уровне БД.
<details>
  <summary>📸 Скриншоты: Срабатывание Regex Constraint</summary>
  
  ![Regex Test](images/pg-regex-test.png) </details>

### Фаза 4: Disaster Recovery (Бэкап и Восстановление)
> **Цель:** Проверка регламента резервного копирования и восстановления после катастрофического сбоя (DROP DATABASE).
* **Шаги:**
  1. Создание логического дампа БД с помощью утилиты `pg_dump`.
  2. Имитация катастрофы: переключение в БД `postgres` и выполнение `DROP DATABASE qa_pg_store;`. Проверка через `\l`.
  3. Восстановление: создание пустой БД и заливка дампа через CLI.
* **Результат:** База данных полностью восстановлена, проверочный `SELECT` подтвердил наличие ранее внесенных данных. Потерь нет.
<details>
  <summary>📸 Скриншоты: DROP DATABASE и Успешное восстановление</summary>
  
  ![Drop DB](images/pg-drop-db.png) ![Recovered DB](images/pg-recovered-db.png) </details>

---

## 🚀 Инструкция по воспроизведению (Step-by-Step Guide)

Для развертывания стенда и повторения тестов на вашем сервере, выполните следующие шаги:

### Шаг 1: Установка сервера
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
sudo systemctl status postgresql

### Шаг 2: Создание схем и настройка доступов
Заходим под системным пользователем postgres:

```bash
sudo -u postgres psql
```
Создаем базу, изолированную схему и ограниченного пользователя:

```bash
CREATE DATABASE qa_pg_store;
```
```bash
\c qa_pg_store
```
```bash
CREATE SCHEMA admin_secrets;
CREATE TABLE admin_secrets.keys (key_value VARCHAR(100));
CREATE USER qa_pg_user WITH PASSWORD 'TestPass123!';
GRANT CONNECT ON DATABASE qa_pg_store TO qa_pg_user;
```
```bash
\q
```
### Шаг 3: Проведение Security Test (Schema Isolation)
Подключаемся по TCP/IP от лица ограниченного пользователя:

```bash
psql -h 127.0.0.1 -U qa_pg_user -d qa_pg_store
```
# Пароль: TestPass123!

Пытаемся прочитать секретную схему:
```bash
SELECT * FROM admin_secrets.keys;
```
-- Ожидаем: ERROR: permission denied for schema admin_secrets

```bash
\q
```
### Шаг 4: Тестирование Data Integrity (Regex)
Заходим под админом и создаем таблицу с Regex-проверкой:
```bash
sudo -u postgres psql -d qa_pg_store
```
```bash
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(15) CHECK (phone ~ '^\+380[0-9]{9}$')
);
GRANT ALL ON clients TO qa_pg_user;
GRANT USAGE ON SEQUENCE clients_id_seq TO qa_pg_user;
```
```bash
\q
```
Заходим под QA-пользователем и проводим тесты:
```bash
psql -h 127.0.0.1 -U qa_pg_user -d qa_pg_store
```
-- Happy Path
```bash
INSERT INTO clients (name, phone) VALUES ('Ivan', '+380501234567');
```
-- Negative Test (Срабатывание Regex CHECK)
```bash
INSERT INTO clients (name, phone) VALUES ('Hacker', '050-123-45-67');
```
```bash
\q
```
### Шаг 5: Имитация Disaster Recovery
Создаем резервную копию:
```bash
sudo -u postgres pg_dump qa_pg_store > qa_pg_store_backup.sql
```
Удаляем базу (имитация потери):
```bash
sudo -u postgres psql -d postgres
```
```bash
DROP DATABASE qa_pg_store;
```
```bash
\l
```
```bash
\q
```
Восстанавливаем данные из дампа:
```bash
sudo -u postgres psql -c "CREATE DATABASE qa_pg_store;"
sudo -u postgres psql qa_pg_store < qa_pg_store_backup.sql
```
Проверяем восстановленные данные:
```bash
sudo -u postgres psql -d qa_pg_store -c "SELECT * FROM clients;"
```
