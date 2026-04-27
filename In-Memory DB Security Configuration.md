<h1 align="center">⚡ In-Memory DB Security Audit: Valkey (Redis Fork)</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Database-Valkey_7.2-DD0031?style=flat-square&logo=redis&logoColor=white" alt="Valkey">
  <img src="https://img.shields.io/badge/Security-Hardening_%26_Auth-005C84?style=flat-square" alt="Hardening">
  <img src="https://img.shields.io/badge/OS-Ubuntu_24.04-E95420?style=flat-square&logo=ubuntu&logoColor=white" alt="Ubuntu">
  <img src="https://img.shields.io/badge/Status-Secured-brightgreen?style=flat-square" alt="Secured">
</p>

## 📌 Про проект

Цей проект демонструє процес розгортання, аудиту та захисту високопродуктивної in-memory бази даних **Valkey** (open-source форк Redis від Linux Foundation). Основний фокус зроблений на усуненні класичної вразливості **Security Misconfiguration** — відсутності автентифікації за умовчанням, що є критичним ризиком для сховищ кешу та сесій.

---

## 📊 Зведення результатів аудиту

| Фаза | Векторні тестування | Результат / Опис | Статус |
|:---|:---|:---|:---:|
| **1** | `Baseline & Smoke Test` | Встановлення Valkey із репозиторіїв Ubuntu. Вдалий запуск служби, споживання пам'яті ~3MB. | ✅ PASS |
| **2** | `Penetration Test (Exploit)` | Підтверджено вразливість конфігурації: успішний запис та читання даних (`set` / `get`) без аутентифікації. | ⚠️ VULNERABLE |
| **3** | `Hardening` | Налаштування безпеки: редагування `valkey.conf`, активація директиви `requirepass`. | 🔒 APPLIED |
| **4** | `Regression Test (Negative)` | Спроба анонімного доступу після рестарту. Запит відхилено системою безпеки. | ✅ BLOCKED |
| **5** | `Positive Test` | Успішна авторизація за паролем (`auth`) та валідація доступу до збережених даних. | ✅ SECURED |

---

## 🛠 Детальні тест-кейси та звіти

### Фаза 1 і 2: Виявлення вразливості (Unauthenticated Access)
> **Мета:** Перевірити можливість несанкціонованого доступу до даних "з коробки".
* Служба успішно запущена. Відправлено тестовий запит `ping`, отримано відповідь `PONG`.
* **Експлуатація:** Імітуючи дії зловмисника, через утиліту `valkey-cli` був локально записаний ключ `secret_data`. Система дозволила зберегти та прочитати дані без запиту облікових даних.
<details>
  <summary>📸 Скріншот: Успішний запис даних без пароля</summary>
  
  ![Valkey Exploit](images/valkey-exploit.png)
</details>

### Фаза 3 і 4: Security Patch & Захист від анонімів
> **Мета:** Впровадити обов'язкову аутентифікацію.
* Відредаговано головний конфігураційний файл `/etc/valkey/valkey.conf`. Активовано та задано параметр `requirepass`. Службу перезапущено.
* **Negative Test:** Повторна спроба анонімно відправити команду `ping` була успішно заблокована. Система видала очікувану помилку: `(error) NOAUTH Authentication required.`
<details>
  <summary>📸 Скріншот: Блокування анонімного доступу (NOAUTH)</summary>
  
  ![Valkey NOAUTH](images/valkey-noauth.png)
</details>

### Фаза 5: Валідація прав доступу (Positive Testing)
> **Мета:** Перевірити коректність роботи аутентифікації для адміністратора.
* Виконано вхід у консоль з передачею пароля `auth Qwerty123Sec!`.
* **Результат:** База даних підтвердила особистість (`OK`) і успішно повернула вміст захищеної змінної `secret_data`.
<details>
  <summary>📸 Скріншот: Успішна авторизація та читання даних</summary>
  
  ![Valkey Auth Success](images/valkey-auth-success.png)
</details>

---

## 🚀 Інструкція з відтворення стенду

### 1. Встановлення та перевірка
``` bash
sudo apt update
sudo apt install valkey -y
systemctl status valkey
valkey-cli ping
````
### 2. Перевірка вразливості (До захисту)
``` bash
valkey-cli set secret_data "Тим є загальний Open!"
valkey-cli get secret_data
````
### 3. Застосування захисту (Hardening)
Відредагувати файл /etc/valkey/valkey.conf, розкоментувати та змінити рядок:
``` bash
requirepass YourSecurePassword!
````
Перезапустити службу:
``` bash
sudo systemctl restart valkey
````
### 4. Перевірка захисту (NOAUTH)
``` bash
valkey-cli ping
# Очікуваний результат: (error) NOAUTH Authentication required.
````
### 5. Правильна авторизація
``` bash
valkey-cli
> auth YourSecurePassword!
> get secret_data
> exit
````
