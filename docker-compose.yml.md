<h1 align="center">🐳 Docker Infrastructure QA & Security Testing</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Docker-Ready-2496ED?style=flat-square&logo=docker&logoColor=white" alt="Docker">
  <img src="https://img.shields.io/badge/Security_Scan-Trivy-00A550?style=flat-square&logo=security&logoColor=white" alt="Trivy">
  <img src="https://img.shields.io/badge/Load_Test-Apache_Benchmark-D22128?style=flat-square&logo=apache&logoColor=white" alt="Apache Benchmark">
  <img src="https://img.shields.io/badge/Status-100%25_Passed-brightgreen?style=flat-square" alt="Passed">
</p>

## 📌 О проекте

Этот проект демонстрирует комплексный E2E-подход к тестированию контейнерной инфраструктуры. В отличие от стандартного функционального тестирования (UI/API), здесь фокус сделан на **DevSecOps-практиках**: отказоустойчивости, изоляции сетей, мониторинге нагрузок и поиске уязвимостей в базовых образах (CVE).

**Тестовый стенд:** `Ubuntu 24.04 LTS` | `Docker Compose` | `WordPress + MySQL`

---

## 📊 Сводка результатов тестирования

| ID | Тип тестирования | Инструмент | Проверяемый компонент | Статус |
|:---|:---|:---|:---|:---:|
| **TC-01** | `Smoke / Networking` | Docker CLI | Развертывание и изоляция портов БД | ✅ PASS |
| **TC-02** | `Internal API` | cURL (Alpine) | Внутренняя DNS-маршрутизация | ✅ PASS |
| **TC-03** | `Resilience` | Docker Volumes | Сохранность данных при краше БД | ✅ PASS |
| **TC-04** | `Security (CVE)` | AquaSec Trivy | Уязвимости базовых образов | ⚠️ WARN |
| **TC-05** | `Load / Stress` | Apache Benchmark | Стабильность под трафиком (1k req) | ✅ PASS |

---

## 🛠 Подробные Тест-кейсы и Отчеты

### 1. Тестирование развертывания и сетевой изоляции (Smoke)
> **Цель:** Убедиться, что сервисы поднимаются корректно, а критичные порты (MySQL 3306) не торчат наружу.
* **Шаги:** Запуск среды командой `docker compose up -d`. Анализ проброса портов через `docker ps`.
* **Фактический результат:** Web-сервер успешно отдает порт 80. База данных изолирована внутри сети `qa-docker-test_default`, доступ извне заблокирован (Connection refused).
<details>
  <summary>📸 Показать скриншот (Console & UI)</summary>
  
  ![Docker PS](images/docker-ps.png)
  ![WP Start](images/wp-start.png)
</details>

### 2. Тестирование внутренней маршрутизации (Internal API)
> **Цель:** Проверить доступность микросервисов внутри закрытой виртуальной сети Docker.
* **Шаги:** Запуск изолированного Alpine-контейнера с отправкой `curl -I -s http://qa_test_web`.
* **Фактический результат:** Получен ответ `HTTP/1.1 200 OK`. Служба обнаружения сервисов (Docker DNS) работает без сбоев.
<details>
  <summary>📸 Показать скриншот (cURL Response)</summary>
  
  ![cURL Test](images/curl-test.png)
</details>

### 3. Краш-тест и отказоустойчивость (Resilience & Data Persistence)
> **Цель:** Доказать, что при фатальном сбое контейнера базы данных пользовательские данные не пропадают.
* **Шаги:**
  1. Жесткое удаление запущенной БД: `docker rm -f qa_test_db`.
  2. Подтверждение падения приложения (White screen / DB Error).
  3. Перезапуск инфраструктуры: `docker compose up -d`.
* **Фактический результат:** Приложение полностью восстановилось. Данные сохранены благодаря внешнему монтированию Docker Volumes (`db_data` и `wp_data`).
<details>
  <summary>📸 Показать скриншоты (Crash & Recovery)</summary>
  
  ![DB Crash](images/db-crash.png)
  ![Recovery](images/recovery-commands.png)
  ![WP Recovered](images/wp-recovered.png)
</details>

### 4. Сканирование безопасности (Vulnerability Scanning & CVE Analysis)
> **Цель:** Проактивный поиск известных уязвимостей (CVE) в используемых Docker-образах.

#### ⚠️ Управление инцидентом (Incident Response & Troubleshooting)

В процессе проведения теста на сканирование безопасности среда столкнулась с последствиями **Supply Chain атаки**, произошедшей в марте 2026 года.

1. **🚫 Попытка 1 (Провал):** Стандартный метод запуска сканера (из реестра Docker Hub) не сработал.
   
   *Команда:*
   ```bash
   docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image wordpress:latest

   Результат (Фактическая ошибка):
   Error response from daemon: manifest for aquasec/trivy:latest not found: manifest unknown

   Попытка 2 (Фикс): Процесс тестирования был оперативно адаптирован. Источник образа был изменен на официальный, безопасный реестр GitHub Container Registry (GHCR), который не был затронут атакой.

   ```bash
   docker run --rm -v /var/run/docker.sock:/var/run/docker.sock ghcr.io/aquasecurity/trivy:latest image wordpress:latest

* **Фактический результат:** Обнаружены уязвимости уровня `MEDIUM` (в частности, в библиотеке `zlib1g` CVE-2026-27171). Данные собраны для передачи в отдел разработки/DevOps.
<details>
  <summary>📸 Показать скриншот (Trivy Report)</summary>
  
  ![Trivy Scan](images/trivy-scan.png)
</details>

### 5. Нагрузочное тестирование (Stress Test)
> **Цель:** Проверка способности инфраструктуры выдерживать всплески трафика.
* **Шаги:** Генерация 1000 запросов с 50 параллельными потоками через временный контейнер: `ab -n 1000 -c 50 http://<server-ip>/`.
* **Фактический результат:** * Успешно выполнено: 100% (0 failed requests).
  * Среднее время ответа (Mean time per request): `~750 ms`.
  * Пропускная способность: `~66 req/sec`. Сервер стабилен.
<details>
  <summary>📸 Показать скриншот (Apache Benchmark Report)</summary>
  
  ![AB Report](images/ab-report.png)
</details>
