<h1 align="center">🚀 Enterprise Node.js: High Availability & OWASP Security Audit</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Environment-Node.js-339933?style=flat-square&logo=node.js&logoColor=white" alt="Node.js">
  <img src="https://img.shields.io/badge/Process_Manager-PM2-2B037A?style=flat-square&logo=pm2&logoColor=white" alt="PM2">
  <img src="https://img.shields.io/badge/Security-OWASP_Top_10-00A550?style=flat-square" alt="OWASP">
  <img src="https://img.shields.io/badge/Status-Patched_%26_Secured-brightgreen?style=flat-square" alt="Secured">
</p>

## 📌 О проекте

Этот проект представляет собой комплексный аудит отказоустойчивости и безопасности API на базе Node.js. В рамках тестирования была настроена высокая доступность (High Availability) с помощью диспетчера процессов PM2, а также проведена эмуляция реальной кибератаки (Path Traversal / Local File Inclusion) с последующим регрессионным тестированием выпущенного патча безопасности.

**Тестовый стенд:** `Ubuntu 24.04 LTS` | `Node.js v18+` | `PM2` | `UFW`

---

## 📊 Сводка результатов аудита

| Фаза | Тип тестирования | Описание | Статус |
|:---|:---|:---|:---:|
| **1** | `High Availability` | Crash-тест: принудительное завершение процесса (SIGKILL) и проверка автовосстановления через PM2 | ✅ PASS |
| **2** | `Security Audit` | Анализ кода на наличие уязвимостей некорректной обработки пользовательского ввода | ⚠️ VULNERABLE |
| **3** | `Penetration Test` | Эксплуатация уязвимости Path Traversal (LFI) для чтения системного файла `/etc/passwd` | 🚩 EXPLOITED |
| **4** | `Regression Test` | Внедрение Security Patch (`path.basename`) и повторная проверка вектора атаки | ✅ SECURED |

---

## 🛠 Подробные тест-кейсы и отчеты

### Фаза 1: Инфраструктурный Crash-тест (High Availability)
> **Цель:** Проверка способности приложения к самовосстановлению после критического сбоя.
* **Сценарий:** Приложение запущено через PM2. С помощью команды `kill -9 <PID>` процесс сервера был принудительно уничтожен на уровне ядра ОС.
* **Результат:** Диспетчер PM2 мгновенно зафиксировал падение, увеличил счетчик `restarts` и автоматически поднял новый процесс (новый PID). Время простоя (Downtime) составило менее секунды.
<details>
  <summary>📸 Скриншот: Успешное восстановление процесса в PM2</summary>
  
  ![PM2 Crash Test](images/pm2-crash.png) </details>

### Фаза 2 и 3: Эксплуатация уязвимости (Path Traversal / LFI)
> **Цель:** Аудит механизма маршрутизации файлов API на устойчивость к атакам типа Local File Inclusion (OWASP).
* **Сценарий:** В коде присутствовала уязвимость слепого доверия пользовательскому вводу (`fs.readFileSync(filename)`). Был отправлен вредоносный HTTP GET запрос с полезной нагрузкой: `?file=/etc/passwd`.
* **Результат:** Сервер обработал запрос без санитизации путей и успешно вернул содержимое критического системного файла `/etc/passwd` со списком локальных пользователей. Атака признана успешной.
<details>
  <summary>📸 Скриншот: Успешный взлом (чтение /etc/passwd)</summary>
  
  ![LFI Exploit](images/lfi-exploit.png) </details>

### Фаза 4: Security Patch & Regression Test
> **Цель:** Устранение уязвимости и верификация защиты.
* **Действия:** Исходный код API модифицирован. Внедрен модуль `path`, переменная очищается через `path.basename(filename)`, что отсекает любые попытки выхода из рабочей директории через `../` или абсолютные пути `/`. Сервер перезапущен через `pm2 restart`.
* **Регрессионный тест:** Повторная отправка пейлоада `?file=/etc/passwd`.
* **Результат:** Атака успешно заблокирована. Сервер корректно обработал исключение и вернул статус **"File not found or Access Denied"**.
<details>
  <summary>📸 Скриншот: Успешная блокировка атаки после патча</summary>
  
  ![LFI Patched](images/lfi-patched.png) </details>

---

📦 Инструкция: Установка Node.js и npm на Ubuntu 24.04

Шаг 1: Обновление индексов пакетов

Перед установкой любого нового софта в Linux всегда правилом хорошего тона является обновление списка доступных пакетов. Это гарантирует, что система скачает самые свежие стабильные версии.
Введи в терминале:
```bash
sudo apt update
```
Шаг 2: Установка Node.js и пакетного менеджера (npm)

В Ubuntu платформа Node.js и её пакетный менеджер (npm — Node Package Manager, который нужен для установки модулей вроде PM2) ставятся одной командой. Флаг -y автоматически соглашается со всеми вопросами системы об объеме скачиваемых данных.
Введи:
```bash
sudo apt install nodejs npm -y
```
Шаг 3: Проверка установки (Smoke Test)

Как тестировщики, мы не верим системе на слово. Нам нужно убедиться, что бинарные файлы успешно добавлены в систему и отзываются на команды.
Проверяем версию Node.js:
```bash
node -v
```
(Ожидаемый вывод: что-то вроде v18.19.1 или выше).

Проверяем версию пакетного менеджера:
```bash
npm -v
```
(Ожидаемый вывод: что-то вроде 9.2.0 или выше).

---

## 🚀 Инструкция по воспроизведению стенда

### 1. Установка и запуск PM2
```bash
sudo npm install -g pm2
pm2 start app.js --name "qa-secure-api"
```
### 2. Тестирование автовосстановления
```bash
pm2 status
kill -9 <PID_процесса>
pm2 status # Проверяем, что счетчик restarts увеличился
```
### 3. Уязвимый код (Для Фазы 3)
```Java
// ... инициализация
const filename = parsedUrl.query.file;
// ⚠️ Уязвимость: прямой доступ к ФС без проверок
const data = fs.readFileSync(filename, 'utf8'); 
// ...
```
### 4. Проведение патчинга (Фаза 4)
Заменить уязвимый блок на безопасный вариант с использованием path:
```Java
const path = require('path');
// ...
const filename = parsedUrl.query.file;
// ✅ Патч: очистка пути от директорий
const safeFilename = path.basename(filename); 
const data = fs.readFileSync(safeFilename, 'utf8');
// ...
```
После обновления кода перезапустить сервис:
```bash
pm2 restart qa-secure-api
```
