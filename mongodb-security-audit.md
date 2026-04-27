<h1 align="center">🍃 MongoDB: NoSQL Security Audit & RBAC Hardening</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Database-MongoDB_7.0-47A248?style=flat-square&logo=mongodb&logoColor=white" alt="MongoDB">
  <img src="https://img.shields.io/badge/Security-RBAC_%26_Hardening-005C84?style=flat-square" alt="RBAC">
  <img src="https://img.shields.io/badge/OS-Ubuntu_24.04-E95420?style=flat-square&logo=ubuntu&logoColor=white" alt="Ubuntu">
  <img src="https://img.shields.io/badge/Status-Secured-brightgreen?style=flat-square" alt="Secured">
</p>

## 📌 Про проект

Цей інфраструктурний проект демонструє процес встановлення NoSQL бази даних MongoDB з офіційних репозиторіїв розробника на чистий сервер Ubuntu, проведення аудиту безпеки конфігурації за умовчанням та подальше впровадження жорсткого контролю доступу (Hardening).

Проект сфокусований на усуненні критичної вразливості Security Misconfiguration (OWASP), властивої MongoDB з коробки - відсутності автентифікації. У рамках харденінгу реалізовано рольову модель доступу (RBAC).

---

## 📊 Зведення результатів аудиту

| Фаза | Векторні тестування | Результат / Опис | Статус |
|:---|:---|:---|:---:|
| **1** | `Baseline Configuration` | Встановлення СУБД та аналіз налаштувань за умовчанням. Виявлено відкритий доступ до всіх баз без пароля. | ⚠️ VULNERABLE |
| **2** | `Negative Test (Hacking)` | Спроба анонімного доступу (`Unauthorized`) після редагування конфігураційного файлу `mongod.conf`. | ✅ BLOCKED |
| **3** | `Positive Test (RBAC)` | Авторизація через створеного суперкористувача та успішне виконання системних команд. | ✅ SECURED |

---

## 🛠 Детальні тест-кейси та звіти

### Фаза 1: Розгортання та Виявлення вразливості
> **Мета:** Встановити MongoDB v7.0 та проаналізувати базову безпеку.
* Сервер MongoDB успішно встановлений та запущений (`systemctl status mongod`).
* **Уразливість:** При вході в консоль `mongosh` система видала системне попередження: *Access control is not enabled for the database*. Будь-який анонімний користувач мав необмежені права на читання та запис (`readWriteAnyDatabase`).
<details> 
<summary>📸 Скріншоти: Успішний запуск та Відсутність контролю доступу</summary> 

![MongoDB Status](images/mongo-status.png) ![MongoDB No Auth](images/mongo-no-auth.png) </details>

### Фаза 2: Hardening & Конфігурація RBAC
> **Мета:** Налаштування рольової моделі доступу та примусове включення авторизації.
* У системній базі `admin` створено профіль адміністратора з повними правами.
* Внесені зміни до `/etc/mongod.conf`: активовано параметр `security.authorization: enabled`.
* **Negative Test:** Після рестарту сервісу спроба анонімно виконати команду `show dbs` призвела до очікуваної помилки **MongoServerError[Unauthorized]**. Захист успішно відбив нелегітимний запит.
<details> 
<summary>📸 Скріншоти: Налаштування Конфігу та Блокування Аноніма</summary> 

![YAML Config](images/mongo-config.png) ![Unauthorized Error](images/mongo-unauthorized.png) </details>

### Фаза 3: Валідація прав доступу (Positive Testing)
> **Мета:** Підтвердити коректну роботу авторизації для довірених користувачів.
* Здійснено вхід з явною передачею облікових даних: `mongosh -u "admin" -p "***" --authenticationDatabase "admin"`.
* **Результат:** Сервер успішно аутентифікував сесію, дозволивши виконання системних команд (читання списку БД `show dbs`). Інфраструктура визнана безпечною для розгортання програм.
<details> 
<summary>📸 Скріншот: Успішна автентифікація</summary> 

![Successful Login](images/mongo-login-success.png) </details>

---

## 🚀 Інструкція з відтворення (Step-by-Step Guide)

### 1. Встановлення MongoDB з офіційного репозиторію
``` bash
sudo apt install gnupg curl -y
curl -fsSL [https://www.mongodb.org/static/pgp/server-7.0.asc](https://www.mongodb.org/static/pgp/server-7.0.asc) | sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] [https://repo.mongodb.org/apt/ubuntu](https://repo.mongodb.org/apt/ubuntu) jammy | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt update
sudo apt install -y mongodb-org
sudo systemctl enable --now mongod
````
### 2. Створення адміністратора (всередині mongosh)
```Java
use admin
db.createUser({ 
user: "username", 
pwd: "YourSecurePassword!", 
roles: [ 
{ role: "userAdminAnyDatabase", db: "admin"}, 
{ role: "readWriteAnyDatabase", db: "admin"}, 
{ role: "dbAdminAnyDatabase", db: "admin"} 
]
})
exit
````
### 3. Включення обов'язкової авторизації
Відредагувати файл /etc/mongod.conf:
``YAML
security: 
authorization: enabled
````
Перезапустити сервіс:
``` bash
sudo systemctl restart mongod
````
### 4. Тестування доступу
Спроба зайти без пароля (передбачається помилка виконання команд):
``` bash
mongosh
````
Успішний вхід за паролем:
``` bash
mongosh -u "username" -p "YourSecurePassword!" --authenticationDatabase "admin"
````
