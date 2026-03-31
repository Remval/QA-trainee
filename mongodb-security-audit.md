<h1 align="center">🍃 MongoDB: NoSQL Security Audit & RBAC Hardening</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Database-MongoDB_7.0-47A248?style=flat-square&logo=mongodb&logoColor=white" alt="MongoDB">
  <img src="https://img.shields.io/badge/Security-RBAC_%26_Hardening-005C84?style=flat-square" alt="RBAC">
  <img src="https://img.shields.io/badge/OS-Ubuntu_24.04-E95420?style=flat-square&logo=ubuntu&logoColor=white" alt="Ubuntu">
  <img src="https://img.shields.io/badge/Status-Secured-brightgreen?style=flat-square" alt="Secured">
</p>

## 📌 О проекте

Этот инфраструктурный проект демонстрирует процесс установки NoSQL базы данных MongoDB из официальных репозиториев разработчика на "чистый" сервер Ubuntu, проведение аудита безопасности конфигурации по умолчанию и последующее внедрение жесткого контроля доступа (Hardening). 

Проект сфокусирован на устранении критической уязвимости **Security Misconfiguration (OWASP)**, свойственной MongoDB из коробки — отсутствию аутентификации. В рамках харденинга реализована ролевая модель доступа (RBAC).

---

## 📊 Сводка результатов аудита

| Фаза | Вектор тестирования | Результат / Описание | Статус |
|:---|:---|:---|:---:|
| **1** | `Baseline Configuration` | Установка СУБД и анализ настроек по умолчанию. Выявлен открытый доступ ко всем базам без пароля. | ⚠️ VULNERABLE |
| **2** | `Negative Test (Hacking)` | Попытка анонимного доступа (`Unauthorized`) после редактирования конфигурационного файла `mongod.conf`. | ✅ BLOCKED |
| **3** | `Positive Test (RBAC)` | Авторизация через созданного суперпользователя и успешное выполнение системных команд. | ✅ SECURED |

---

## 🛠 Подробные тест-кейсы и отчеты

### Фаза 1: Развертывание и Выявление уязвимости
> **Цель:** Установить MongoDB v7.0 и проанализировать базовую безопасность.
* Сервер MongoDB успешно установлен и запущен (`systemctl status mongod`).
* **Уязвимость:** При входе в консоль `mongosh` система выдала системное предупреждение: *Access control is not enabled for the database*. Любой анонимный пользователь имел неограниченные права на чтение и запись (`readWriteAnyDatabase`).
<details>
  <summary>📸 Скриншоты: Успешный запуск и Отсутствие контроля доступа</summary>
  
  ![MongoDB Status](images/mongo-status.png) ![MongoDB No Auth](images/mongo-no-auth.png) </details>

### Фаза 2: Hardening & Конфигурация RBAC
> **Цель:** Настройка ролевой модели доступа и принудительное включение авторизации.
* В системной базе `admin` создан профиль администратора с полными правами.
* Внесены изменения в `/etc/mongod.conf`: активирован параметр `security.authorization: enabled`.
* **Negative Test:** После рестарта сервиса попытка анонимно выполнить команду `show dbs` привела к ожидаемой ошибке **MongoServerError[Unauthorized]**. Защита успешно отразила нелегитимный запрос.
<details>
  <summary>📸 Скриншоты: Настройка Конфига и Блокировка Анонима</summary>
  
  ![YAML Config](images/mongo-config.png) ![Unauthorized Error](images/mongo-unauthorized.png) </details>

### Фаза 3: Валидация прав доступа (Positive Testing)
> **Цель:** Подтвердить корректную работу авторизации для доверенных пользователей.
* Осуществлен вход с явной передачей учетных данных: `mongosh -u "admin" -p "***" --authenticationDatabase "admin"`.
* **Результат:** Сервер успешно аутентифицировал сессию, разрешив выполнение системных команд (чтение списка БД `show dbs`). Инфраструктура признана безопасной для развертывания приложений.
<details>
  <summary>📸 Скриншот: Успешная аутентификация</summary>
  
  ![Successful Login](images/mongo-login-success.png) </details>

---

## 🚀 Инструкция по воспроизведению (Step-by-Step Guide)

### 1. Установка MongoDB из официального репозитория
```bash
sudo apt install gnupg curl -y
curl -fsSL [https://www.mongodb.org/static/pgp/server-7.0.asc](https://www.mongodb.org/static/pgp/server-7.0.asc) | sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] [https://repo.mongodb.org/apt/ubuntu](https://repo.mongodb.org/apt/ubuntu) jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt update
sudo apt install -y mongodb-org
sudo systemctl enable --now mongod
```
### 2. Создание администратора (внутри mongosh)
```Java
use admin
db.createUser({
  user: "username",
  pwd: "YourSecurePassword!",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" },
    { role: "dbAdminAnyDatabase", db: "admin" }
  ]
})
exit
```
### 3. Включение обязательной авторизации
Отредактировать файл /etc/mongod.conf:
```YAML
security:
  authorization: enabled
```
Перезапустить сервис:
```bash
sudo systemctl restart mongod
```
### 4. Тестирование доступа
Попытка зайти без пароля (ожидается ошибка выполнения команд):
```bash
mongosh
```
Успешный вход по паролю:
```bash
mongosh -u "username" -p "YourSecurePassword!" --authenticationDatabase "admin"
````
