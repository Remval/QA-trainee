<h1 align="center">🐧 Ubuntu Server 24.04: OS Hardening & Security Audit</h1>

<p align="center">
  <img src="https://img.shields.io/badge/OS-Ubuntu_24.04-E95420?style=flat-square&logo=ubuntu&logoColor=white" alt="Ubuntu">
  <img src="https://img.shields.io/badge/Security-OS_Hardening-00A550?style=flat-square&logo=security&logoColor=white" alt="Security">
  <img src="https://img.shields.io/badge/Firewall-UFW-005C84?style=flat-square" alt="UFW">
  <img src="https://img.shields.io/badge/Status-Secured-brightgreen?style=flat-square" alt="Secured">
</p>

## 📌 О проекте

Этот проект представляет собой практический аудит безопасности (Security Audit) и укрепление (Hardening) "голого" сервера на базе Ubuntu 24.04 LTS. Цель проекта — устранить базовые уязвимости свежеустановленной ОС, минимизировать поверхность атаки, настроить сетевой экран и защитить критически важные службы от брутфорс-атак.

**Тестовый стенд:** `Ubuntu 24.04.4 LTS` | `UFW` | `Fail2Ban` | `Терминал Linux (CLI)`

---

## 📊 Сводка результатов аудита и настройки

| Фаза | Тип работ | Описание | Статус |
|:---|:---|:---|:---:|
| **1** | `System Baseline` | Анализ уязвимых пакетов и сканирование открытых портов | ✅ DONE |
| **2** | `Network Security` | Патчинг системы и настройка межсетевого экрана (UFW) | ✅ SECURED |
| **3** | `SSH Hardening` | Защита от брутфорса (установка и настройка Fail2Ban) | ✅ SECURED |
| **4** | `Privilege Escalation`| Поиск SUID-уязвимостей и аудит системных логов (auth.log) | ✅ CLEAN |

---

## 🛠 Подробные отчеты по фазам

### Фаза 1: Базовый срез (System Baseline)
> **Цель:** Оценить исходное состояние безопасности сервера до внесения изменений.
* **Действия:** Анализ ожидающих обновлений (`apt list --upgradable`), проверка слушающих портов (`ss -tuln`) и статуса брандмауэра (`ufw status`).
* **Результат аудита:** Обнаружены критические системные обновления (включая ядро). Выявлено, что SSH порт (22) открыт всем, а межсетевой экран **отключен** (Status: inactive).
<details>
  <summary>📸 Скриншоты: Исходное состояние сервера</summary>
  
  ![Updates](images/baseline-updates.png) ![Ports](images/baseline-ports.png) ![UFW Inactive](images/baseline-ufw.png) </details>

### Фаза 2: Сетевая безопасность и Патчинг (Network & Firewall)
> **Цель:** Устранить известные уязвимости (CVE) и минимизировать поверхность атаки.
* **Действия:** Установка патчей (`apt upgrade`). Настройка политики UFW: `default deny incoming`, `default allow outgoing`. Явное разрешение доступа только для порта `22/tcp` (SSH).
* **Результат:** Сервер обновлен. Входящий трафик заблокирован, за исключением легитимного SSH-подключения. 
<details>
  <summary>📸 Скриншот: Настроенный UFW</summary>
  
  ![UFW Active](images/ufw-active.png) </details>

### Фаза 3: Защита доступа (SSH Hardening)
> **Цель:** Защитить сервер от автоматизированных атак по перебору паролей (Brute-force).
* **Действия:** Установка, запуск и проверка службы `fail2ban`. Проверка конфигурации `sshd_config` на предмет запрета входа от имени пользователя `root` по паролю.
* **Результат:** Создана "тюрьма" (jail) для SSH. IP-адреса, совершающие неудачные попытки входа, будут автоматически блокироваться на уровне файрвола.
<details>
  <summary>📸 Скриншот: Статус Fail2Ban Jail</summary>
  
  ![Fail2Ban](images/fail2ban-status.png) </details>

### Фаза 4: Аудит привилегий и Логирование (Privilege & Audit)
> **Цель:** Проверить систему на наличие векторов локального повышения привилегий и убедиться в корректности журналирования событий безопасности.
* **Действия:**
  1. Поиск файлов с SUID-битами (`find / -perm -4000`).
  2. Анализ журнала аутентификации (`/var/log/auth.log`) на предмет несанкционированных вызовов `sudo`.
* **Результат:** Аномальных SUID-файлов не обнаружено (только стандартные системные утилиты вроде `passwd` и `sudo`). Логи корректно фиксируют весь Audit Trail администратора. Система признана безопасной.
<details>
  <summary>📸 Скриншоты: SUID Audit и Syslog</summary>
  
  ![SUID Find](images/suid-audit.png) ![Auth Log](images/auth-log.png) </details>

  ---

## 🚀 Инструкция по воспроизведению (Step-by-Step Guide)

Если вы хотите провести аналогичный аудит и базовое укрепление (Hardening) на своем сервере Ubuntu, выполните следующие шаги:

### Шаг 1: Сбор базовой информации (System Baseline)
Проверяем наличие обновлений:
```bash
apt list --upgradable
```
Сканируем открытые порты, которые слушают сеть:
```bash
ss -tuln
```
Проверяем статус встроенного файрвола:
```bash
udo ufw status
```
### Шаг 2: Патчинг и настройка Firewall (UFW)
Устанавливаем все актуальные обновления безопасности:
```bash
sudo apt upgrade -y
```
Настраиваем базовые политики сетевого экрана (запрещаем всё входящее, разрешаем всё исходящее):
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
Важно: Разрешаем SSH-соединения, чтобы не потерять доступ к серверу:
```bash
sudo ufw allow ssh
```
Включаем файрвол и проверяем его подробный статус:
```bash
sudo ufw enable
sudo ufw status verbose
```
### Шаг 3: Защита от Brute-force атак (SSH Hardening)
Устанавливаем и запускаем fail2ban для автоматической блокировки злоумышленников:
```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```
Проверяем статус "тюрьмы" для SSH:
```bash
sudo fail2ban-client status sshd
```
Проверяем конфигурацию SSH-сервера на предмет блокировки входа от имени root по паролю:
```bash
sudo grep PermitRootLogin /etc/ssh/sshd_config
```
### Шаг 4: Поиск SUID-уязвимостей и проверка логов
Ищем все файлы в системе, имеющие установленный SUID-бит (вектор локального повышения привилегий):
```bash
find / -perm -4000 -type f 2>/dev/null
```
Анализируем системный журнал аутентификации на предмет подозрительной активности:
```bash
sudo cat /var/log/auth.log | grep sudo | tail -n 10
```
