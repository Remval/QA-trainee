<h1 align="center">🗄️ MySQL Bare-Metal QA & Security Audit</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Database-MySQL-4479A1?style=flat-square&logo=mysql&logoColor=white" alt="MySQL">
  <img src="https://img.shields.io/badge/OS-Ubuntu_24.04-E95420?style=flat-square&logo=ubuntu&logoColor=white" alt="Ubuntu">
  <img src="https://img.shields.io/badge/Security-RBAC_Testing-00A550?style=flat-square&logo=security&logoColor=white" alt="Security">
  <img src="https://img.shields.io/badge/Status-100%25_Passed-brightgreen?style=flat-square" alt="Passed">
</p>

## 📌 Про проект

Цей проект присвячений комплексному тестуванню бази даних MySQL, розгорнутій на "голому залізі" (bare-metal сервер Ubuntu). У фокусі тестування – не просто CRUD-операції, а глибока перевірка механізмів безпеки, ізоляції прав доступу, бізнес-обмежень (Constraints) на рівні СУБД та перевірка сценарію Disaster Recovery.

**Тестовий стенд:** `Ubuntu 24.04 LTS` | `MySQL 8.x` | `Термінал Linux (CLI)`

---

## 📊 Зведення результатів тестування

| **1** | `Smoke Test` | Встановлення та запуск сервісу MySQL ✅ PASS |
| **2** | `Security/RBAC` | Ізоляція прав користувача, захист від ескалації привілеїв ✅ PASS |
| **3** | `Data Integrity` | Перевірка обмежень (NOT NULL, UNIQUE, CHECK) ✅ PASS |
| **4** | `Disaster Recovery` | Імітація втрати БД та повне відновлення з дампи | ✅ PASS |

---

## 🛠 Детальні Тест-кейси та Звіти

### Фаза 1: Встановлення та Smoke Test
> **Мета:** Переконатися, що сервіс бази даних успішно встановлений, працює стабільно та керується через `systemd`.
* **Результат:** Сервіс запущений (`active (running)`), процес ізольований штатно.
<details> 
<summary>📸 Скріншот: Успішний старт сервісу</summary> 

![MySQL Status](images/mysql-status.png)
</details>

### Фаза 2: Тестування безпеки та прав доступу (Security & RBAC)
> **Мета:** Провести негативне тестування контролю доступу. Звичайний користувач не повинен мати доступу до системної БД `mysql`, де зберігаються хеші паролів.

#### ⚠️ Траблшутинг (Bypassing systemd auth constraints)
У процесі налаштування середовища зіткнувся зі строгими обмеженнями Ubuntu 24.04: локальний рут не мав пароля і блокувався `auth_socket`.
* **Рішення:** Проведено примусовий обхід захисту (Hacker Mode) шляхом додавання параметра `skip-grant-tables` безпосередньо в `/etc/mysql/mysql.conf.d/mysqld.cnf`. Після успішного скидання пароля `root` і створення тестового користувача з обмеженими правами, вразливість у конфізі була усунена, а система повернена у безпечний режим.

* **Тест-кейс (Негативний):** Спроба доступу до `mysql` від імені обмеженого `qa_user`.
* **Очікуваний / Фактичний результат:** Очікувана відмова у доступі. Отримана помилка `ERROR 1044 (42000): Access denied for user`. Система безпеки працює коректно.
<details> 
<summary>📸 Скріншот: Access Denied (RBAC Test)</summary> 

![Access Denied](images/access-denied.png)
</details>

### Фаза 3: Тестування цілісності даних (Data Integrity Testing)
> **Мета:** Перевірка того, що БД самостійно блокує некоректні дані, захищаючи бізнес-логіку навіть за відсутності валідації на бекенді.

Створено таблицю `customers` зі строгими обмеженнями: `email NOT NULL UNIQUE` та вік `age CHECK (>= 18)`. Проведено негативні тести (спроби вставки невалідних даних):
1. **Тест NOT NULL:** Спроба створити користувача без email -> `ERROR 1364: Field 'email' не має default value`.
2. **Тест UNIQUE:** Спроба вставити дублікат email -> `ERROR 1062: Duplicate entry`.
3. **Тест CHECK:** Спроба додати 15-річного користувача -> `ERROR 3819: Check constraint is violated`.
* **Результат:** БД успішно відбила всі 3 спроби порушення цілісності даних.
<details> 
<summary>📸 Скріншоти: Спрацювання Constraints (Помилки вставки)</summary> 

![Constraints Test 1](images/constraint-1.png) 
![Constraints Test 2](images/constraint-2.png)
</details>

### Фаза 4: Disaster Recovery (Бекап та Відновлення)
> **Мета:** Перевірити можливість повного відновлення даних при їхньому фізичному чи логічному знищенні (DROP DATABASE).
* **Результат:** База даних та всі записи успішно відновлені за секунди з SQL-дампа. Втрат даних немає.
<details> 
<summary>📸 Скріншоти: DROP DATABASE та Успішне відновлення</summary> 

![Drop DB](images/drop-db.png) 
![Recovered DB](images/recovered-db.png)
</details>

---
## 🚀 Інструкція з відтворення (Step-by-Step Guide)

Якщо ви хочете розгорнути цей тестовий стенд на своєму Ubuntu-сервері та повторити перевірки, дотримуйтесь інструкцій нижче.

### Крок 1: Встановлення сервера
``` bash
sudo apt update
sudo apt install mysql-server -y
sudo systemctl status mysql
````

### Крок 2: Обхід захисту (systemd) та створення тестового середовища
Зупиняємо БД та впроваджуємо бекдор

``` bash
sudo systemctl stop mysql
sudo sed -i '/\[mysqld\]/a skip-grant-tables' /etc/mysql/mysql.conf.d/mysqld.cnf
sudo systemctl start mysql
````
Заходимо без пароля
``` bash
sudo mysql -u root
````
Всередині консолі mysql> виконуємо SQL-запити:

``` bash
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'RootPass123!';
CREATE DATABASE qa_store;
CREATE USER 'qa_user'@'localhost' IDENTIFIED BY 'TestPass123!';
GRANT ALL PRIVILEGES ON qa_store.* TO 'qa_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
````

Повертаємо сервер у безпечний режим:
``` bash
sudo systemctl stop mysql
sudo sed -i '/skip-grant-tables/d' /etc/mysql/mysql.conf.d/mysqld.cnf
sudo systemctl start mysql
````

### Крок 3: Проведення Security Test (RBAC)
``` bash
mysql -u qa_user -p
# Вводимо пароль: TestPass123!
````

Намагаємося зламати систему (отримуємо Access denied):

``` bash
USE mysql;
````
### Крок 4: Тестування Data Integrity

У консолі обмеженого користувача створюємо таблицю:

``` bash
USE qa_store;

CREATE TABLE customers ( 
id INT AUTO_INCREMENT PRIMARY KEY, 
email VARCHAR(255) NOT NULL UNIQUE, 
age INT CHECK (age >= 18)
);
#Happy Path (Позитивний тест)
````



``` bash
INSERT INTO customers (email, age) VALUES ('test@example.com', 25);
#Negative Tests (Перевірка обмежень)
````


``` bash
INSERT INTO customers (age) VALUES (30); # Помилка NOT NULL
INSERT INTO customers (email, age) VALUES ('test@example.com', 40); # Помилка UNIQUE
INSERT INTO customers (email, age) VALUES ('young@example.com', 15); # Помилка CHECK
EXIT;
````

### Крок 5: Імітація Disaster Recovery

Створюємо резервну копію бази даних:

``` bash
mysqldump -u root -p qa_store > qa_store_backup.sql
#Пароль рута: RootPass123!
````


Імітуємо катастрофу (заходимо під рутом і видаляємо БД):
``` bash
mysql -u root -p
````

``` bash
DROP DATABASE qa_store;
SHOW DATABASES;
CREATE DATABASE qa_store;
EXIT;
````

Відновлюємо дані з дампа та перевіряємо успішність:

``` bash
mysql -u root -p qa_store < qa_store_backup.sql
mysql -u root -p -e "USE qa_store; SELECT * FROM customers;"
````
