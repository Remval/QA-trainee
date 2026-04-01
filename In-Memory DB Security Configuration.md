<h1 align="center">⚡ In-Memory DB Security Audit: Valkey (Redis Fork)</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Database-Valkey_7.2-DD0031?style=flat-square&logo=redis&logoColor=white" alt="Valkey">
  <img src="https://img.shields.io/badge/Security-Hardening_%26_Auth-005C84?style=flat-square" alt="Hardening">
  <img src="https://img.shields.io/badge/OS-Ubuntu_24.04-E95420?style=flat-square&logo=ubuntu&logoColor=white" alt="Ubuntu">
  <img src="https://img.shields.io/badge/Status-Secured-brightgreen?style=flat-square" alt="Secured">
</p>

## 📌 О проекте

Этот проект демонстрирует процесс развертывания, аудита и защиты высокопроизводительной in-memory базы данных **Valkey** (open-source форк Redis от Linux Foundation). Основной фокус сделан на устранении классической уязвимости **Security Misconfiguration** — отсутствии аутентификации по умолчанию, что является критическим риском для хранилищ кэша и пользовательских сессий.

---

## 📊 Сводка результатов аудита

| Фаза | Вектор тестирования | Результат / Описание | Статус |
|:---|:---|:---|:---:|
| **1** | `Baseline & Smoke Test` | Установка Valkey из репозиториев Ubuntu. Успешный запуск службы, потребление памяти ~3MB. | ✅ PASS |
| **2** | `Penetration Test (Exploit)` | Подтверждена уязвимость конфигурации: успешная запись и чтение данных (`set` / `get`) без аутентификации. | ⚠️ VULNERABLE |
| **3** | `Hardening` | Настройка безопасности: редактирование `valkey.conf`, активация директивы `requirepass`. | 🔒 APPLIED |
| **4** | `Regression Test (Negative)` | Попытка анонимного доступа (Ping) после рестарта. Запрос отклонен системой безопасности. | ✅ BLOCKED |
| **5** | `Positive Test` | Успешная авторизация по паролю (`auth`) и валидация доступа к сохраненным данным. | ✅ SECURED |

---

## 🛠 Подробные тест-кейсы и отчеты

### Фаза 1 и 2: Выявление уязвимости (Unauthenticated Access)
> **Цель:** Проверить возможность несанкционированного доступа к данным "из коробки".
* Служба успешно запущена. Отправлен тестовый запрос `ping`, получен ответ `PONG`.
* **Эксплуатация:** Имитируя действия злоумышленника, через утилиту `valkey-cli` был локально записан ключ `secret_data`. Система позволила сохранить и прочитать данные без запроса учетных данных.
<details>
  <summary>📸 Скриншот: Успешная запись данных без пароля</summary>
  
  ![Valkey Exploit](images/valkey-exploit.png)
</details>

### Фаза 3 и 4: Security Patch & Защита от анонимов
> **Цель:** Внедрить обязательную аутентификацию.
* Отредактирован главный конфигурационный файл `/etc/valkey/valkey.conf`. Активирован и задан параметр `requirepass`. Служба перезапущена.
* **Negative Test:** Повторная попытка анонимно отправить команду `ping` была успешно заблокирована. Система выдала ожидаемую ошибку: `(error) NOAUTH Authentication required.`
<details>
  <summary>📸 Скриншот: Блокировка анонимного доступа (NOAUTH)</summary>
  
  ![Valkey NOAUTH](images/valkey-noauth.png)
</details>

### Фаза 5: Валидация прав доступа (Positive Testing)
> **Цель:** Проверить корректность работы аутентификации для администратора.
* Выполнен вход в консоль с передачей пароля `auth Qwerty123Sec!`.
* **Результат:** База данных подтвердила личность (`OK`) и успешно вернула содержимое защищенной переменной `secret_data`.
<details>
  <summary>📸 Скриншот: Успешная авторизация и чтение данных</summary>
  
  ![Valkey Auth Success](images/valkey-auth-success.png)
</details>

---

## 🚀 Инструкция по воспроизведению стенда

### 1. Установка и проверка
```bash
sudo apt update
sudo apt install valkey -y
systemctl status valkey
valkey-cli ping
```
### 2. Проверка уязвимости (До защиты)
```bash
valkey-cli set secret_data "This database is completely open!"
valkey-cli get secret_data
```
### 3. Применение защиты (Hardening)
Отредактировать файл /etc/valkey/valkey.conf, раскомментировать и изменить строку:
```bash
requirepass YourSecurePassword!
```
Перезапустить службу:
```bash
sudo systemctl restart valkey
```
### 4. Проверка защиты (NOAUTH)
```bash
valkey-cli ping
# Ожидаемый результат: (error) NOAUTH Authentication required.
```
### 5. Правильная авторизация
```bash
valkey-cli
> auth YourSecurePassword!
> get secret_data
> exit
```
