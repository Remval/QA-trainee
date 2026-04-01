<h1 align="center">🗄️ MySQL Bare-Metal QA & Security Audit</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Database-MySQL-4479A1?style=flat-square&logo=mysql&logoColor=white" alt="MySQL">
  <img src="https://img.shields.io/badge/OS-Ubuntu_24.04-E95420?style=flat-square&logo=ubuntu&logoColor=white" alt="Ubuntu">
  <img src="https://img.shields.io/badge/Security-RBAC_Testing-00A550?style=flat-square&logo=security&logoColor=white" alt="Security">
  <img src="https://img.shields.io/badge/Status-100%25_Passed-brightgreen?style=flat-square" alt="Passed">
</p>

## 📌 О проекте

Этот проект посвящен комплексному тестированию базы данных MySQL, развернутой на "голом железе" (bare-metal сервер Ubuntu). В фокусе тестирования — не просто CRUD-операции, а глубокая проверка механизмов безопасности, изоляции прав доступа, бизнес-ограничений (Constraints) на уровне СУБД и проверка сценария Disaster Recovery.

**Тестовый стенд:** `Ubuntu 24.04 LTS` | `MySQL 8.x` | `Терминал Linux (CLI)`

---

## 📊 Сводка результатов тестирования

| Фаза | Тип тестирования | Описание | Статус |
|:---|:---|:---|:---:|
| **1** | `Smoke Test` | Установка и запуск сервиса MySQL | ✅ PASS |
| **2** | `Security / RBAC` | Изоляция прав пользователя, защита от эскалации привилегий | ✅ PASS |
| **3** | `Data Integrity` | Проверка ограничений (NOT NULL, UNIQUE, CHECK) | ✅ PASS |
| **4** | `Disaster Recovery` | Имитация потери БД и полное восстановление из дампа | ✅ PASS |

---

## 🛠 Подробные Тест-кейсы и Отчеты

### Фаза 1: Установка и Smoke Test
> **Цель:** Убедиться, что сервис базы данных успешно установлен, работает стабильно и управляется через `systemd`.
* **Результат:** Сервис запущен (`active (running)`), процесс изолирован штатно.
<details>
  <summary>📸 Скриншот: Успешный старт сервиса</summary>
  
  ![MySQL Status](images/mysql-status.png)
</details>

### Фаза 2: Тестирование безопасности и прав доступа (Security & RBAC)
> **Цель:** Провести негативное тестирование контроля доступа. Обычный пользователь не должен иметь доступа к системной БД `mysql`, где хранятся хеши паролей.

#### ⚠️ Траблшутинг (Bypassing systemd auth constraints)
В процессе настройки среды столкнулся со строгими ограничениями Ubuntu 24.04: локальный рут не имел пароля и блокировался `auth_socket`. 
* **Решение:** Проведен принудительный обход защиты (Hacker Mode) путем добавления параметра `skip-grant-tables` напрямую в `/etc/mysql/mysql.conf.d/mysqld.cnf`. После успешного сброса пароля `root` и создания тестового юзера с ограниченными правами, уязвимость в конфиге была устранена, а система возвращена в безопасный режим.

* **Тест-кейс (Негативный):** Попытка доступа к `mysql` от лица ограниченного `qa_user`.
* **Ожидаемый / Фактический результат:** Ожидаемый отказ в доступе. Получена ошибка `ERROR 1044 (42000): Access denied for user`. Система безопасности работает корректно.
<details>
  <summary>📸 Скриншот: Access Denied (RBAC Test)</summary>
  
  ![Access Denied](images/access-denied.png)
</details>

### Фаза 3: Тестирование целостности данных (Data Integrity Testing)
> **Цель:** Проверка того, что БД самостоятельно блокирует некорректные данные, защищая бизнес-логику даже при отсутствии валидации на бэкенде.

Создана таблица `customers` со строгими ограничениями: `email NOT NULL UNIQUE` и возраст `age CHECK (>= 18)`. Проведены негативные тесты (попытки вставки невалидных данных):
1. **Тест NOT NULL:** Попытка создать юзера без email -> `ERROR 1364: Field 'email' doesn't have a default value`.
2. **Тест UNIQUE:** Попытка вставить дубликат email -> `ERROR 1062: Duplicate entry`.
3. **Тест CHECK:** Попытка добавить 15-летнего пользователя -> `ERROR 3819: Check constraint is violated`.
* **Результат:** БД успешно отбила все 3 попытки нарушения целостности данных.
<details>
  <summary>📸 Скриншоты: Срабатывание Constraints (Ошибки вставки)</summary>
  
  ![Constraints Test 1](images/constraint-1.png)
  ![Constraints Test 2](images/constraint-2.png)
</details>

### Фаза 4: Disaster Recovery (Бэкап и Восстановление)
> **Цель:** Проверить возможность полного восстановления данных при их физическом или логическом уничтожении (DROP DATABASE).
* **Результат:** База данных и все записи успешно восстановлены за секунды из SQL-дампа. Потерь данных нет.
<details>
  <summary>📸 Скриншоты: DROP DATABASE и Успешное восстановление</summary>
  
  ![Drop DB](images/drop-db.png)
  ![Recovered DB](images/recovered-db.png)
</details>

---

## 🚀 Инструкция по воспроизведению (Step-by-Step Guide)

Если вы хотите развернуть этот тестовый стенд на своем Ubuntu-сервере и повторить проверки, следуйте инструкции ниже.

### Шаг 1: Установка сервера
```bash
sudo apt update
sudo apt install mysql-server -y
sudo systemctl status mysql
```

### Шаг 2: Обход защиты (systemd) и создание тестовой среды
Останавливаем БД и внедряем бэкдор

```bash
sudo systemctl stop mysql
sudo sed -i '/\[mysqld\]/a skip-grant-tables' /etc/mysql/mysql.conf.d/mysqld.cnf
sudo systemctl start mysql
```
Заходим без пароля
```bash
sudo mysql -u root
```
Внутри консоли mysql> выполняем SQL-запросы:

```bash
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'RootPass123!';
CREATE DATABASE qa_store;
CREATE USER 'qa_user'@'localhost' IDENTIFIED BY 'TestPass123!';
GRANT ALL PRIVILEGES ON qa_store.* TO 'qa_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Возвращаем сервер в безопасный режим:
```bash
sudo systemctl stop mysql
sudo sed -i '/skip-grant-tables/d' /etc/mysql/mysql.conf.d/mysqld.cnf
sudo systemctl start mysql
```

### Шаг 3: Проведение Security Test (RBAC)
```bash
mysql -u qa_user -p
# Вводим пароль: TestPass123!
```

Пытаемся взломать систему (получаем Access denied):

```bash
USE mysql;
```
### Шаг 4: Тестирование Data Integrity

В консоли ограниченного пользователя создаем таблицу:

```bash
USE qa_store;

CREATE TABLE customers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    age INT CHECK (age >= 18)
);
#Happy Path (Позитивный тест)
```



```bash
INSERT INTO customers (email, age) VALUES ('test@example.com', 25);
#Negative Tests (Проверка ограничений)
```


```bash
INSERT INTO customers (age) VALUES (30); # Ошибка NOT NULL
INSERT INTO customers (email, age) VALUES ('test@example.com', 40); # Ошибка UNIQUE
INSERT INTO customers (email, age) VALUES ('young@example.com', 15); # Ошибка CHECK
EXIT;
```

### Шаг 5: Имитация Disaster Recovery

Создаем резервную копию базы данных:

```bash
mysqldump -u root -p qa_store > qa_store_backup.sql
#Пароль рута: RootPass123!
```


Имитируем катастрофу (заходим под рутом и удаляем БД):
```bash
mysql -u root -p
```

```bash
DROP DATABASE qa_store;
SHOW DATABASES;
CREATE DATABASE qa_store;
EXIT;
```

Восстанавливаем данные из дампа и проверяем успешность:

```bash
mysql -u root -p qa_store < qa_store_backup.sql
mysql -u root -p -e "USE qa_store; SELECT * FROM customers;"
```
