<h1 align="center">🐧 Ubuntu Server 24.04: OS Hardening & Security Audit</h1>

<p align="center">
  <img src="https://img.shields.io/badge/OS-Ubuntu_24.04-E95420?style=flat-square&logo=ubuntu&logoColor=white" alt="Ubuntu">
  <img src="https://img.shields.io/badge/Security-OS_Hardening-00A550?style=flat-square&logo=security&logoColor=white" alt="Security">
  <img src="https://img.shields.io/badge/Firewall-UFW-005C84?style=flat-square" alt="UFW">
  <img src="https://img.shields.io/badge/Status-Secured-brightgreen?style=flat-square" alt="Secured">
</p>

## 📌 Про проект

Цей проект являє собою практичний аудит безпеки (Security Audit) та зміцнення (Hardening) "голого" сервера на базі Ubuntu 24.04 LTS. Мета проекту – усунути базові вразливості свіжовстановленої ОС, мінімізувати поверхню атаки, налаштувати мережевий екран та захистити критично важливі служби від брутфорс-атак.

**Тестовий стенд:** `Ubuntu 24.04.4 LTS` | `UFW` | `Fail2Ban` | `Термінал Linux (CLI)`

---

### 📊 Звіт за результатами аудиту та налаштування

| Фаза | Тип робіт | Опис | Статус |
|:---:|:---|:---|:---:|
| **1** | `System Baseline` | Аналіз вразливих пакетів та сканування відкритих портів | ✅ **DONE** |
| **2** | `Network Security` | Патчинг системи та налаштування міжмережевого екрана (UFW) | ✅ **SECURED** |
| **3** | `SSH Hardening` | Захист від брутфорсу (встановлення та налаштування Fail2Ban) | ✅ **SECURED** |
| **4** | `Privilege Escalation` | Пошук SUID-вразливостей та аудит системних логів (auth.log) | ✅ **CLEAN** |
---

## 🛠 Детальні звіти по фазах

### Фаза 1: Базовий зріз (System Baseline)
> **Мета:** Оцінити вихідний стан безпеки сервера до внесення змін.
* **Дії:** Аналіз очікуваних оновлень (`apt list --upgradable`), перевірка слухачів портів (`ss -tuln`) та статусу брандмауера (`ufw status`).
* **Результат аудиту:** Виявлено критичні системні оновлення (включаючи ядро). Виявлено, що порт SSH (22) відкритий всім, а міжмережевий екран **відключений** (Status: inactive).
<details> 
<summary>📸 Скріншоти: Початковий стан сервера</summary> 

![Updates](images/baseline-updates.png) ![Ports](images/baseline-ports.png) ![UFW Inactive](images/baseline-ufw.png) </details>

### Фаза 2: Мережева безпека та Патчінг (Network & Firewall)
> **Мета:** Усунути відомі вразливості (CVE) та мінімізувати поверхню атаки.
* **Дії:** Встановлення патчів (`apt upgrade`). Налаштування політики UFW: `default deny incoming`, `default allow outgoing`. Явна роздільна здатність доступу тільки для порту `22/tcp` (SSH).
* **Результат:** Сервер оновлено. Вхідний трафік заблокований, крім легітимного SSH-підключення.
<details> 
<summary>📸 Скріншот: Налаштований UFW</summary> 

![UFW Active](images/ufw-active.png) </details>

### Фаза 3: Захист доступу (SSH Hardening)
> **Мета:** Захистити сервер від автоматизованих атак по перебору паролів (Brute-force).
* **Дії:** Встановлення, запуск та перевірка служби `fail2ban`. Перевірка конфігурації `sshd_config` щодо заборони входу від імені користувача `root` за паролем.
* **Результат:** Створено "в'язницю" (jail) для SSH. IP-адреси, які роблять невдалі спроби входу, автоматично блокуватимуться на рівні файрвола.
<details> 
<summary>📸 Скріншот: Статус Fail2Ban Jail</summary> 

![Fail2Ban](images/fail2ban-status.png) </details>

### Фаза 4: Аудит привілеїв та Логування (Privilege & Audit)
> **Мета:** Перевірити систему на наявність векторів локального підвищення привілеїв та переконатися у коректності журналування подій безпеки.
* **Дії:** 
1. Пошук файлів із SUID-бітами (`find/-perm -4000`). 
2. Аналіз журналу аутентифікації (`/var/log/auth.log`) щодо несанкціонованих викликів `sudo`.
* **Результат:** Аномальних SUID-файлів не виявлено (тільки стандартні системні утиліти на кшталт `passwd` та `sudo`). Логи коректно фіксують весь Audit Trail адміністратора. Система визнана безпечною.
<details> 
<summary>📸 Скріншоти: SUID Audit та Syslog</summary> 

![SUID Find](images/suid-audit.png) ![Auth Log](images/auth-log.png) </details> 

---

## 🚀 Інструкція з відтворення (Step-by-Step Guide)

Якщо ви хочете провести аналогічний аудит та базове зміцнення (Hardening) на своєму сервері Ubuntu, виконайте такі кроки:

### Крок 1: Збір базової інформації (System Baseline)
Перевіряємо наявність оновлень:
``` bash
apt list --upgradable
````
Скануємо відкриті порти, які слухають мережу:
``` bash
ss-tuln
````
Перевіряємо статус вбудованого файрвола:
``` bash
udo ufw status
````
### Крок 2: Патчінг і налаштування Firewall (UFW)
Встановлюємо всі актуальні оновлення безпеки:
``` bash
sudo apt upgrade -y
````
Налаштовуємо базові політики мережного екрану (забороняємо все, що входить, дозволяємо все вихідне):
``` bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
````
Важливо: Дозволяємо SSH-з'єднання, щоб не втратити доступ до сервера:
``` bash
sudo ufw allow ssh
````
Включаємо файрвол та перевіряємо його докладний статус:
``` bash
sudo ufw enable
sudo ufw status verbose
````
### Крок 3: Захист від Brute-force атак (SSH Hardening)
Встановлюємо та запускаємо fail2ban для автоматичного блокування зловмисників:
``` bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
````
Перевіряємо статус "в'язниці" для SSH:
``` bash
sudo fail2ban-client status sshd
````
Перевіряємо конфігурацію SSH-сервера щодо блокування входу від імені root за паролем:
``` bash
sudo grep PermitRootLogin /etc/ssh/sshd_config
````
### Крок 4: Пошук SUID-уразливостей та перевірка логів
Шукаємо всі файли в системі, що мають встановлений SUID-біт (вектор локального підвищення привілеїв):
``` bash
find / -perm -4000 -type f 2>/dev/null
````
Аналізуємо системний журнал аутентифікації щодо підозрілої активності:
``` bash
sudo cat /var/log/auth.log | grep sudo | tail -n 10
````
