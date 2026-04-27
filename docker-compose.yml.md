<h1 align="center">🐳 Docker Infrastructure QA & Security Testing</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Docker-Ready-2496ED?style=flat-square&logo=docker&logoColor=white" alt="Docker">
  <img src="https://img.shields.io/badge/Security_Scan-Trivy-00A550?style=flat-square&logo=security&logoColor=white" alt="Trivy">
  <img src="https://img.shields.io/badge/Load_Test-Apache_Benchmark-D22128?style=flat-square&logo=apache&logoColor=white" alt="Apache Benchmark">
  <img src="https://img.shields.io/badge/Status-100%25_Passed-brightgreen?style=flat-square" alt="Passed">
</p>

## 📌 Про проект

Цей проект демонструє комплексний E2E підхід до тестування контейнерної інфраструктури. На відміну від стандартного функціонального тестування (UI/API), тут фокус зроблено на **DevSecOps-практиках**: відмовостійкості, ізоляції мереж, моніторингу навантажень та пошуку вразливостей у базових образах (CVE).

**Тестовий стенд:** `Ubuntu 24.04 LTS` | `Docker Compose` | `WordPress + MySQL`

---

## 📊 Зведення результатів тестування

| ID | Тип тестування Інструмент | Перевірюваний компонент | Статус |
|:---|:---|:---|:---|:---:|
| **TC-01** | `Smoke/Networking` | Docker CLI | Розгортання та ізоляція портів БД | ✅ PASS |
| **TC-02** | `Internal API` | CURL (Alpine) | Внутрішня DNS-маршрутизація ✅ PASS |
| **TC-03** | `Resilience` | Docker Volumes | Збереження даних при фарбуванні БД ✅ PASS |
| **TC-04** | 'Security (CVE)' | AquaSec Trivy | Вразливості базових образів ⚠️ WARN |
| **TC-05** | `Load/Stress` | Apache Benchmark Стабільність під трафіком (1k req) ✅ PASS |

---

## 🛠 Детальні Тест-кейси та Звіти

### 1. Тестування розгортання та мережевої ізоляції (Smoke)
> **Мета:** Переконатися, що сервіси піднімаються коректно, а критичні порти (MySQL 3306) не стирчать назовні.
* **Кроки:** Запуск середовища командою `docker compose up -d`. Аналіз прокидання портів через `docker ps`.
* **Фактичний результат:** Web-сервер успішно віддає порт 80. База даних ізольована всередині мережі `qa-docker-test_default`, доступ ззовні заблокований (Connection refused).
<details> 
<summary>📸 Показати скріншот (Console & UI)</summary> 

![Docker PS](images/docker-ps.png) 
![WP Start](images/wp-start.png)
</details>

### 2. Тестування внутрішньої маршрутизації (Internal API)
> **Мета:** Перевірити доступність мікросервісів усередині закритої віртуальної мережі Docker.
* **Кроки:** Запуск ізольованого Alpine-контейнера з відправкою `curl -I -s http://qa_test_web`.
* **Фактичний результат:** Отримана відповідь `HTTP/1.1 200 OK`. Служба виявлення послуг (Docker DNS) працює без збоїв.
<details> 
<summary>📸 Показати скріншот (cURL Response)</summary> 

![cURL Test](images/curl-test.png)
</details>

### 3. Краш-тест і відмовостійкість (Resilience & Data Persistence)
> **Мета:** Довести, що при фатальному збої контейнера бази даних дані користувача не пропадають.
* **Кроки:** 
1. Жорстке видалення запущеної БД: `docker rm -f qa_test_db`. 
2. Підтвердження падіння програми (White screen/DB Error). 
3. Перезапуск інфраструктури: `docker compose up -d`.
* **Фактичний результат:** Додаток повністю відновився. Дані збережені завдяки зовнішньому монтуванню Docker Volumes (`db_data` та `wp_data`).
<details> 
<summary>📸 Показати скріншоти (Crash & Recovery)</summary> 

![DB Crash](images/db-crash.png) 
![Recovery](images/recovery-commands.png) 
![WP Recovered](images/wp-recovered.png)
</details>

### 4. Сканування безпеки (Vulnerability Scanning & CVE Analysis)
> **Мета:** Проактивний пошук відомих уразливостей (CVE) у використовуваних Docker-образах.

#### ⚠️ Управління інцидентом (Incident Response & Troubleshooting)

У процесі проведення тесту на сканування безпеки середовище зіштовхнулося з наслідками **Supply Chain атаки**, що відбулася у березні 2026 року.

1. **🚫 Спроба 1 (Провал):** Стандартний метод запуску сканера (з реєстру Docker Hub) не спрацював.
   
   *Команда:* 
``` bash 
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image 

Результат (Фактична помилка): 
Error response from daemon: manifest for aquasec/trivy: 

Спроба 2 (Фікс): Процес тестування оперативно адаптований. Джерело образу було змінено на офіційний, безпечний реєстр GitHub Container Registry (GHCR), який не торкнувся атаки. 
```
``` bash 
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock ghcr.io/aquasecurity/trivy:latest image
```
* **Фактичний результат:** Виявлено вразливості рівня `MEDIUM` (зокрема, у бібліотеці `zlib1g` CVE-2026-27171). Дані зібрані для передачі у відділ розробки/DevOps.
<details> 
<summary>📸 Показати скріншот (Trivy Report)</summary> 

![Trivy Scan](images/trivy-scan.png)
</details>

### 5. Навантажувальне тестування (Stress Test)
> **Мета:** Перевірка можливості інфраструктури витримувати сплески трафіку.
* **Кроки:** Генерація 1000 запитів з 50 паралельними потоками через тимчасовий контейнер: `ab -n 1000 -c 50 http://<server-ip>/`.
* **Фактичний результат:** * Успішно виконано: 100% (0 failed requests). 
* Середній час відповіді (Mean time per request): `~750 ms`. 
* Пропускна спроможність: `~66 req/sec`. Сервер стабільний.
<details> 
<summary>📸 Показати скріншот (Apache Benchmark Report)</summary> 

![AB Report](images/ab-report.png)
</details>
