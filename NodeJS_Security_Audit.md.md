<h1 align="center">🚀 Enterprise Node.js: High Availability & OWASP Security Audit</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Environment-Node.js-339933?style=flat-square&logo=node.js&logoColor=white" alt="Node.js">
  <img src="https://img.shields.io/badge/Process_Manager-PM2-2B037A?style=flat-square&logo=pm2&logoColor=white" alt="PM2">
  <img src="https://img.shields.io/badge/Security-OWASP_Top_10-00A550?style=flat-square" alt="OWASP">
  <img src="https://img.shields.io/badge/Status-Patched_%26_Secured-brightgreen?style=flat-square" alt="Secured">
</p>

## 📌 Про проект

Цей проект являє собою комплексний аудит відмовостійкості та безпеки API на базі Node.js. В рамках тестування було налагоджено високу доступність (High Availability) за допомогою диспетчера процесів PM2, а також проведено емуляцію реальної кібератаки (Path Traversal / Local File Inclusion) з подальшим регресійним тестуванням випущеного патча безпеки.

**Тестовый стенд:** `Ubuntu 24.04 LTS` | `Node.js v18+` | `PM2` | `UFW`

---

### 📊 Зведення результатів аудиту (Security & Stability)

| Фаза | Тип тестування | Опис | Статус |
|:---:|:---|:---|:---:|
| **1** | `High Availability` | Crash-тест: примусове завершення процесу (SIGKILL) та перевірка автовідновлення через PM2 | ✅ **PASS** |
| **2** | `Security Audit` | Аналіз коду на наявність уразливостей некоректної обробки введення користувача | ⚠️ **VULNERABLE** |
| **3** | `Penetration Test` | Експлуатація вразливості Path Traversal (LFI) для читання системного файлу `/etc/passwd` | 🚩 **EXPLOITED** |
| **4** | `Regression Test` | Впровадження Security Patch (`path.basename`) та повторна перевірка вектора атаки | ✅ **SECURED** |
---

## 🛠 Детальні тест-кейси та звіти

### Фаза 1: Інфраструктурний Crash-тест (High Availability)
> **Мета:** Перевірка можливості застосування до самовідновлення після критичного збою.
* **Сценарій:** Додаток запущено через PM2. За допомогою команди `kill -9 <PID> процес сервера був примусово знищений на рівні ядра ОС.
* **Результат:** Диспетчер PM2 миттєво зафіксував падіння, збільшив лічильник `restarts` і автоматично підняв новий процес (новий PID). Час простою (Downtime) становив менше секунди.
<details> 
<summary>📸 Скріншот: Успішне відновлення процесу в PM2</summary>
  
  ![PM2 Crash Test](images/pm2-crash.png) </details>

### Фаза 2 та 3: Експлуатація вразливості (Path Traversal / LFI)
> **Мета:** Аудит механізму маршрутизації файлів API на стійкість до атак типу Local File Inclusion (OWASP).
* **Сценарій:** У коді була присутня вразливість сліпої довіри введення користувача (`fs.readFileSync(filename)`). Був відправлений шкідливий HTTP GET запит з корисним навантаженням: `?file=/etc/passwd`.
* **Результат:** Сервер обробив запит без санітизації шляхів і успішно повернув вміст критичного системного файлу `/etc/passwd` зі списком локальних користувачів. Атаку визнано успішною.
<details> 
<summary>📸 Скріншот: Успішний злом (читання /etc/passwd)</summary>
  
  ![LFI Exploit](images/lfi-exploit.png) </details>

### Фаза 4: Security Patch & Regression Test
> **Мета:** Усунення вразливості та верифікація захисту.
* **Дії:** Вихідний код API модифікований. Впроваджено модуль `path`, змінна очищається через `path.basename(filename)`, що відсікає будь-які спроби виходу з робочої директорії через `../` або абсолютні шляхи `/`. Сервер перезапущен через "pm2 restart".
* **Регресійний тест:** Повторне відправлення пейлоаду `?file=/etc/passwd`.
* **Результат:** Атака успішно заблокована. Сервер коректно обробив виняток і повернув статус "File not found or Access Denied"**.
<details> 
<summary>📸 Скріншот: Успішне блокування атаки після патчу</summary>
  
  ![LFI Patched](images/lfi-patched.png) </details>

---

📦 Інструкція: Установка Node.js та npm на Ubuntu 24.04

Крок 1: Оновлення індексів пакетів

Перед встановленням нового софту в Linux завжди правилом хорошого тону є оновлення списку доступних пакетів. Це гарантує, що система завантажить найсвіжіші стабільні версії.
Введи у терміналі:
``` bash
sudo apt update
````
Крок 2: Встановлення Node.js та пакетного менеджера (npm)

У Ubuntu платформа Node.js та її пакетний менеджер (npm — Node Package Manager, який потрібен для встановлення модулів типу PM2) ставляться однією командою. Прапор -y автоматично погоджується з усіма питаннями системи про обсяг скачуваних даних.
Введи:
``` bash
sudo apt install nodejs npm -y
````
Крок 3: Перевірка установки (Smoke Test)

Як випробувачі, ми не віримо системі на слово. Нам потрібно переконатися, що бінарні файли успішно додані до системи та відгукуються на команди.
Перевіряємо версію Node.js:
``` bash
node -v
````
(Очікуваний висновок: щось подібне до v18.19.1 або вище).

Перевіряємо версію пакетного менеджера:
``` bash
npm -v
````
(Очікуваний висновок: щось на зразок 9.2.0 або вище).

---

## 🚀 Інструкція з відтворення стенду

### 1. Встановлення та запуск PM2
``` bash
sudo npm install -g pm2
pm2 start app.js --name "qa-secure-api"
````
### 2. Тестування автовідновлення
``` bash
pm2 status
kill -9 <PID_процесу>
pm2 status # Перевіряємо, що лічильник restarts збільшився
````
### 3. Вразливий код (Для Фази 3)
```Java
// ... ініціалізація
const filename = parsedUrl.query.file;
// ⚠️ Вразливість: прямий доступ до ФС без перевірок
const data = fs.readFileSync(filename, 'utf8');
// ...
````
### 4. Проведення патчингу (Фаза 4)
Замінити вразливий блок на безпечний варіант за допомогою path:
```Java
const path = require('path');
// ...
const filename = parsedUrl.query.file;
// ✅ Патч: очищення колії від директорій
const safeFilename = path.basename(filename);
const data = fs.readFileSync(safeFilename, 'utf8');
// ...
````
Після оновлення коду перезапустити сервіс:
``` bash
pm2 restart qa-secure-api
````
