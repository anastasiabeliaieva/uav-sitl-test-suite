# ТЗ: UAV SITL Test Suite
### Портфоліо-проєкт для переходу в Defense Tech / Robotics QA

> **Як читати цей файл у VS Code:** натисни `Ctrl+Shift+V` (на Mac `Cmd+Shift+V`), щоб побачити гарний перегляд.
> Квадратики `- [ ]` — це чекліст. Коли пункт зроблено, заміни `[ ]` на `[x]`.
> Усі незнайомі слова пояснені в розділі **[13. Словник термінів](#13-словник-термінів)** в кінці. Якщо бачиш слово з позначкою 📖 — воно там є.

---

## Зміст

1. [Що ми будуємо і навіщо](#1-що-ми-будуємо-і-навіщо)
2. [Аналогія з тим, що ти вже знаєш](#2-аналогія-з-тим-що-ти-вже-знаєш)
3. [Як усе влаштовано (схема)](#3-як-усе-влаштовано-схема)
4. [Що встановити](#4-що-встановити)
5. [Структура репозиторію](#5-структура-репозиторію)
6. [План по етапах](#6-план-по-етапах)
7. [Тест-кейси](#7-тест-кейси)
8. [Стартовий код](#8-стартовий-код)
9. [Правила якості](#9-правила-якості)
10. [Типові проблеми і як їх вирішити](#10-типові-проблеми-і-як-їх-вирішити)
11. [Що написати в README](#11-що-написати-в-readme)
12. [Як розповідати про проєкт на співбесіді](#12-як-розповідати-про-проєкт-на-співбесіді)
13. [Словник термінів](#13-словник-термінів)
14. [Корисні посилання](#14-корисні-посилання)

---

## 1. Що ми будуємо і навіщо

**Коротко:** набір автоматичних тестів, які «запускають» віртуальний дрон, віддають йому команди (злетіти, полетіти по точках, повернутися додому) і перевіряють, що він поводиться правильно — у тому числі в аварійних ситуаціях (втрата зв'язку, сів акумулятор, вихід за дозволену зону).

**Справжнього дрона не треба.** Використовуємо **ArduPilot** 📖 — найпоширеніше відкрите програмне забезпечення для автопілотів (на ньому літають тисячі реальних дронів, зокрема в оборонних компаніях) — у режимі **SITL** 📖, тобто «прошивка дрона працює на комп'ютері, а фізика польоту симулюється».

**Навіщо це для кар'єри:**
- це рівно те, що пишуть у вакансіях deftech: *Python, pytest, SITL, MAVLink, CI*;
- на співбесіді можна показати живий проєкт, а не розповідати «я вчу»;
- проєкт показує твою сильну сторону — **будувати тестовий процес з нуля** — але вже на дроні.

**Кінцевий результат (Definition of Done 📖 всього проєкту):**
- [ ] публічний репозиторій на GitHub `uav-sitl-test-suite`
- [ ] 10–12 автотестів, які проходять зелено
- [ ] тести автоматично запускаються в **GitHub Actions** 📖 на кожен push
- [ ] **Allure** 📖-звіт з результатами
- [ ] акуратний README з GIF-кою польоту і таблицею тестів
- [ ] посилання на репозиторій додане в резюме

**Орієнтовний час:** 4 тижні по 10–12 годин + бонусні етапи за бажанням.

---

## 2. Аналогія з тим, що ти вже знаєш

Ти щодня працюєш з Playwright. Тут усе майже те саме, тільки об'єкт тестування інший:

| Веб-тестування (знаєш) | Тестування дрона (вчимо) |
|---|---|
| Браузер (Chromium) | **SITL** — симулятор дрона |
| Веб-сайт, який тестуєш | Прошивка **ArduPilot Copter** |
| Playwright API (`page.click()`) | **pymavlink** 📖 (`drone.set_mode("GUIDED")`) |
| HTTP-запити / відповіді | **MAVLink** 📖-повідомлення |
| Page Object Model | «Drone Object» — клас-обгортка з методами `arm()`, `takeoff()`, `land()` |
| `expect(locator).toBeVisible({timeout})` | «чекаємо, поки висота стане 10 м, але не довше 30 с» |
| TypeScript + Playwright Test | **Python + pytest** 📖 |
| GitLab CI | **GitHub Actions** (той самий принцип, інший синтаксис YAML) |

**Головна різниця:** у вебі сторінка «стоїть» і чекає на тебе. Дрон — живий: він постійно рухається, висота і координати змінюються щосекунди. Тому майже кожна перевірка — це **«дочекатися стану з таймаутом»**, а не «перевірити миттєво».

---

## 3. Як усе влаштовано (схема)

```
┌─────────────────────────────┐        MAVLink по TCP        ┌──────────────────────────────┐
│  Твої тести (Python/pytest) │  ── порт 5760 ────────────▶  │  SITL: прошивка ArduCopter   │
│                             │  ◀── телеметрія ───────────  │  + симуляція фізики польоту  │
│  tests/test_takeoff.py      │                              │  (віртуальний квадрокоптер)  │
│  helpers/drone.py           │                              └──────────────┬───────────────┘
└──────────────┬──────────────┘                                             │ порт 5762
               │                                                            ▼
               ▼                                             ┌──────────────────────────────┐
┌─────────────────────────────┐                              │  QGroundControl (за бажанням)│
│  Allure-звіт                │                              │  карта: бачиш, як дрон летить│
└─────────────────────────────┘                              └──────────────────────────────┘
```

**Покроково, що відбувається в одному тесті:**
1. pytest запускає програму `arducopter` (це і є SITL) — на Маку всередині Docker-контейнера, у CI напряму.
2. Тест підключається до неї по `tcp:127.0.0.1:5760`.
3. Чекає **heartbeat** 📖 — сигнал «я живий, я дрон».
4. Відправляє команди (змінити режим, **заармитись** 📖, злетіти…).
5. Читає телеметрію (висоту, координати, режим) і перевіряє через `assert`.
6. Після тесту SITL закривається. Наступний тест отримує «новенький» дрон.

---

## 4. Що встановити

### 4.1 Mac (твій варіант) 🍏

**Як це працює на Маку.** Готовий SITL-файл від ArduPilot — це програма для **Linux**, а не для macOS. Тому запускаємо його всередині **Docker** 📖-контейнера: це «маленький Linux у коробці». А самі тести (Python) працюють прямо на Маку і підключаються до контейнера по порту 5760.

```
┌────────── Твій Mac ──────────────────────────────────────────────┐
│                                                                  │
│  Тести (Python, pytest)  ──tcp:5760──▶  ┌─ Docker-контейнер ──┐  │
│  QGroundControl          ──tcp:5762──▶  │  Linux + arducopter │  │
│                                         └─────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

Бонус: Docker стоїть у тебе в резюме в блоці навчання — тут ти його освоїш природно.

**Крок 1. Дізнайся, який у тебе процесор**
- [ ] Яблучко в лівому верхньому куті → **Про цей Mac**. Якщо написано **Chip Apple M1/M2/M3/M4** — у тебе Apple Silicon. Якщо **Processor Intel** — Intel. Інструкція нижче підходить для обох, різниця тільки в кроці 4.

**Крок 2. Термінал і інструменти розробника**
- [ ] Відкрий програму **Terminal** (Cmd+Пробіл → набери «Terminal»). Команди з цього ТЗ виконуються тут. На Маку термінал працює в оболонці **zsh** 📖 — майже всі Linux-команди (`cd`, `ls`, `ps`, `kill`, `curl`) працюють так само.
- [ ] Встанови інструменти розробника Apple (в них є `git`):
  ```bash
  xcode-select --install
  ```
  З'явиться вікно — натисни «Встановити». Займе 5–15 хвилин.

**Крок 3. Homebrew і Python**
- [ ] Встанови **Homebrew** 📖 — «магазин програм для терміналу». Скопіюй команду з сайту **brew.sh** і виконай. В кінці установник напише 2–3 команди «Next steps» — **обов'язково виконай їх**, інакше `brew` не знайдеться.
- [ ] Встанови Python:
  ```bash
  brew install python
  python3 --version      # має бути 3.10 або новіше
  ```

**Крок 4. Docker Desktop**
- [ ] Завантаж **Docker Desktop for Mac** з docker.com — обери версію під свій процесор (Apple Silicon або Intel). Встанови, запусти, дочекайся зеленого статусу «Engine running» (кит 🐳 у верхньому меню).
- [ ] **Тільки для Apple Silicon:** Docker Desktop → ⚙️ Settings → **General** → увімкни **«Use Rosetta for x86_64/amd64 emulation on Apple Silicon»** → Apply & restart. **Rosetta** 📖 дозволяє запускати Intel-програми на процесорах Apple — швидко і без помітних втрат.
- [ ] Перевір:
  ```bash
  docker run --rm hello-world
  ```
  Має вивести «Hello from Docker!».

> Альтернатива Docker Desktop — **OrbStack** (orbstack.dev): легший і швидший на Маку, команди `docker` ті самі. Можна будь-який з двох.

**Крок 5. VS Code і папка проєктів**
- [ ] У VS Code встанови розширення **Python** і **Docker** (обидва від Microsoft).
- [ ] Щоб командою `code .` відкривати папку у VS Code з терміналу: у VS Code натисни Cmd+Shift+P → набери **Shell Command: Install 'code' command in PATH**.
- [ ] Створи папку для проєктів:
  ```bash
  mkdir -p ~/projects && cd ~/projects
  ```

**Крок 6. Зібрати Docker-образ із симулятором** (після того, як створиш репозиторій на Етапі 1)
- [ ] У корені репозиторію створи файл `Dockerfile` (код у розділі 8.9) і виконай:
  ```bash
  docker build -t uav-sitl .
  ```
  Це **образ** 📖 — «шаблон», з якого потім запускаються контейнери. Збирається один раз, ~1–2 хвилини.
- [ ] Пробний запуск симулятора:
  ```bash
  docker run --rm -it --platform linux/amd64 -p 5760:5760 -p 5762:5762 uav-sitl
  ```
  Побачиш `Waiting for connection ....` — SITL готовий і чекає на підключення. Зупинити: **Ctrl+C**.
- [ ] Щоб тести знали, що SITL треба запускати через Docker, у терміналі перед запуском тестів виконуй:
  ```bash
  export SITL_MODE=docker
  ```
  Або один раз додай цей рядок у файл `~/.zshrc` (відкрий: `code ~/.zshrc`), тоді він діятиме завжди.

> **План Б (якщо з Docker щось не піде):** ArduPilot можна зібрати прямо під macOS за офіційною інструкцією: ardupilot.org/dev/docs/building-setup-mac.html. Це довше (~1 година) і примхливіше, але без Docker. Після збірки вкажи тестам шлях до програми:
> ```bash
> export SITL_MODE=local
> export SITL_BIN=~/ardupilot/build/sitl/bin/arducopter
> export SITL_PARAMS=~/ardupilot/Tools/autotest/default_params/copter.parm
> ```

### 4.2 Windows (довідково, якщо колись працюватимеш з Windows-ноутбука)

SITL — це Linux-програма. На Windows вона працює через **WSL2** 📖 (вбудований у Windows Linux).

- [ ] **PowerShell від імені адміністратора:** `wsl --install -d Ubuntu`, перезавантажити комп'ютер, придумати логін і пароль для Ubuntu.
- [ ] У VS Code — розширення **WSL** і **Python**.
- [ ] В Ubuntu: `sudo apt update && sudo apt install -y python3 python3-venv python3-pip git curl`
- [ ] Проєкт тримай у `~/projects` всередині Ubuntu, **не** на диску `C:` — інакше все буде дуже повільно. Відкрити у VS Code: `code .`
- [ ] SITL запускається напряму, без Docker (`bash scripts/get_sitl.sh`), змінну `SITL_MODE` не задавай.

### 4.3 Акаунти та програми (для всіх)

- [ ] Акаунт на **GitHub** (якщо немає) + налаштований `git`:
  ```bash
  git config --global user.name "Anastasiia Beliaieva"
  git config --global user.email "anastasia.beliaieva@gmail.com"
  ```
- [ ] **QGroundControl** 📖 для свого комп'ютера (є версія для Mac) — щоб бачити дрон на карті. Необов'язково для тестів, але дуже корисно для розуміння і для GIF у README. Сайт: qgroundcontrol.com.
  На Маку при першому запуску може з'явитися «неможливо перевірити розробника» — тоді Системні параметри → Конфіденційність і безпека → «Все одно відкрити».

### 4.4 Перевірка, що все готово

```bash
python3 --version   # 3.10 або новіше
git --version
docker --version    # тільки Mac
```

**Де що запускається — коротка шпаргалка:**

| | Mac (ти) | CI (GitHub Actions) | Windows / Linux |
|---|---|---|---|
| SITL | у Docker-контейнері | напряму, `scripts/get_sitl.sh` | напряму, `scripts/get_sitl.sh` |
| `SITL_MODE` | `docker` | не задавати (`local`) | не задавати (`local`) |
| Тести | на Маку, `pytest -v` | на runner-і Ubuntu | у WSL / Linux |

Тести одні й ті самі — змінюється лише спосіб запуску симулятора. На співбесіді це гарний аргумент: «фреймворк працює і локально на macOS через Docker, і в CI на Linux».

---

## 5. Структура репозиторію

```
uav-sitl-test-suite/
├── .github/
│   └── workflows/
│       └── sitl-tests.yml      # налаштування CI (GitHub Actions)
├── helpers/
│   ├── __init__.py
│   ├── drone.py                # клас Drone: arm(), takeoff(), land()... (як Page Object)
│   └── waits.py                # wait_until(): «чекаємо умову з таймаутом»
├── scripts/
│   └── get_sitl.sh             # завантажує SITL та параметри
├── sitl/                       # сюди скрипт кладе arducopter (у git НЕ додаємо)
├── tests/
│   ├── conftest.py             # фікстури: запуск SITL, підключення
│   ├── test_connection.py
│   ├── test_arming.py
│   ├── test_takeoff_land.py
│   ├── test_mission.py
│   ├── test_rtl.py
│   ├── test_failsafe.py
│   ├── test_geofence.py
│   └── test_params.py
├── docs/
│   └── flight.gif              # GIF польоту з QGroundControl для README
├── .gitignore
├── Dockerfile                  # SITL у контейнері (для Mac)
├── pytest.ini
├── requirements.txt
└── README.md
```

---

## 6. План по етапах

### Етап 0. Підготовка (2–3 дні)

**Мета:** робоче середовище + базовий Python.

- [ ] Пункти з розділу 4 виконані.
- [ ] Python-освіження (у тебе вже є досвід з пет-проєктів — просто згадати синтаксис): функції, класи, `try/except`, `with`, f-рядки, `dataclass`.
- [ ] pytest-основи: `assert`, **фікстури** 📖, `@pytest.mark.parametrize` 📖, `pytest -v -k назва`.
  Аналогія: фікстура в pytest ≈ `beforeEach/afterEach` + `test.extend` у Playwright.
- [ ] Командний рядок (на Маку в Terminal ті самі команди): `cd`, `ls -la`, `pwd`, `mkdir`, `cat`, `less`, `chmod +x`, `ps aux | grep`, `kill`, `curl -O`, `tail -f`.

**Готово, коли:** у папці проєкту створено **venv** 📖, встановлено `pytest`, і проходить тест `def test_ok(): assert 1 + 1 == 2`.

---

### Етап 1. Знайомство з дроном (тиждень 1)

**Мета:** зрозуміти предметну область — **руками**, як при exploratory testing.

- [ ] Створи репозиторій `uav-sitl-test-suite` на GitHub, склонуй у `~/projects`.
- [ ] **Mac:** створи `Dockerfile` (розділ 8.9), збери образ і запусти SITL вручну — Крок 6 розділу 4.1.
- [ ] **Windows/Linux:** створи `scripts/get_sitl.sh` (розділ 8.1), запусти його, а потім SITL вручну:
  ```bash
  cd sitl
  ./arducopter --model quad --speedup 1 --defaults copter.parm -I0 \
      --home -35.363261,149.165230,584,353 -w
  ```
  Побачиш у терміналі щось на зразок `Waiting for connection ....` — це добре, SITL чекає на підключення.
- [ ] Відкрий QGroundControl → **Application Settings → Comm Links → Add**: тип **TCP**, хост `127.0.0.1`, порт `5760` → Connect. На карті з'явиться дрон (це Австралія, стандартне тестове поле ArduPilot).
- [ ] Політай руками в QGroundControl: Arm → Takeoff → клікни точку на карті «Go to location» → RTL → Land.
- [ ] Спробуй **режими польоту** 📖: `GUIDED`, `LOITER`, `RTL`, `LAND`. Запиши у `docs/notes.md`, як поводиться дрон у кожному.
- [ ] Пограйся з **параметрами** 📖 у QGroundControl (Vehicle Setup → Parameters): знайди `FENCE_ENABLE`, `FS_GCS_ENABLE`, `RTL_ALT`, `BATT_LOW_VOLT`. Прочитай опис кожного.
- [ ] Переглянь, які **MAVLink-повідомлення** приходять: у QGroundControl меню **Analyze Tools → MAVLink Inspector**. Знайди `HEARTBEAT`, `GLOBAL_POSITION_INT`, `SYS_STATUS`.

**Готово, коли:** можеш своїми словами пояснити: що таке arm, prearm-перевірки, GUIDED, RTL, failsafe, heartbeat, і чому дрон не армиться в перші ~30 секунд після старту.

---

### Етап 2. Перші автотести (тиждень 2)

**Мета:** каркас проєкту + 3 тести.

- [ ] `requirements.txt`, `pytest.ini`, `.gitignore` (розділ 8).
- [ ] `tests/conftest.py` з фікстурами `sitl` і `drone` (розділ 8.4).
- [ ] `helpers/waits.py` — функція `wait_until()` (розділ 8.5).
- [ ] `helpers/drone.py` — клас `Drone` з методами (сигнатури в розділі 8.6). Пиши методи поступово, по мірі потреби тестів.
- [ ] Тести **TC-01, TC-02, TC-03** (розділ 7). Перший тест уже готовий — розділ 8.7.
- [ ] Перший змістовний **commit** 📖 і **push** на GitHub.

**Готово, коли:** `pytest -v` локально показує 3 зелені тести.

---

### Етап 3. Основний набір тестів (тиждень 3)

**Мета:** покрити ключову поведінку дрона, включно з аварійними сценаріями.

- [ ] TC-04 … TC-12 (розділ 7).
- [ ] Хоча б один тест з `@pytest.mark.parametrize` (наприклад, зліт на 5 / 10 / 20 м).
- [ ] Хоча б один **негативний тест** 📖 (TC-11).
- [ ] Кожен тест має позначку **marker** 📖: `@pytest.mark.smoke` для найважливіших (TC-01, 03, 04, 05), решта `@pytest.mark.regression`.

**Готово, коли:** 10+ тестів, стабільно зелені 3 запуски поспіль (перевір на **flaky** 📖).

---

### Етап 4. CI + звіти + оформлення (тиждень 4)

**Мета:** проєкт виглядає «як у компанії».

- [ ] `.github/workflows/sitl-tests.yml` (розділ 8.8) — тести запускаються на кожен push.
- [ ] Підключи `allure-pytest`; додай у тести `@allure.title`, `@allure.severity`, `allure.attach(...)` з телеметрією (наприклад, CSV висоти по часу).
- [ ] Результати Allure зберігаються як **artifact** 📖 у GitHub Actions.
- [ ] **Badge** 📖 «tests passing» у README.
- [ ] GIF польоту: запиши екран QGroundControl під час тесту місії (QGC підключай до порту **5762**, поки тести тримають 5760), конвертуй у GIF (наприклад, ScreenToGif на Windows), поклади в `docs/flight.gif`.
- [ ] README за шаблоном з розділу 11.
- [ ] **Повідом тата → додаємо посилання в резюме і повертаємо рядок «Portfolio Project».**

**Готово, коли:** зелений badge на GitHub, людина без контексту за 2 хвилини розуміє з README, що це і як запустити.

---

### Бонусні етапи (тиждень 5+, за бажанням — кожен окремо додає балів на співбесіді)

- [ ] **B1. Docker Compose.** `Dockerfile` у тебе вже є (Mac). Наступний крок — `docker-compose.yml`, щоб симулятор і тести запускалися однією командою `docker compose up`, а також перевірка Docker-варіанту в GitHub Actions.
- [ ] **B2. Вітер і погода.** Параметризувати тести зліту/місії різною силою вітру (параметри симуляції `SIM_WIND_SPD`, `SIM_WIND_DIR`) — перевірити, що дрон тримає точність.
- [ ] **B3. Аналіз логів польоту.** Після тесту відкрити **dataflash-лог** 📖 (`.BIN`) через `pymavlink.DFReader` і перевірити: максимальна висота, немає помилок **EKF** 📖, рівень вібрацій. Вручну логи можна дивитися на plot.ardupilot.org.
- [ ] **B4. Регресія між версіями прошивки.** Матриця в GitHub Actions: ті самі тести на `stable` і `latest` версіях ArduCopter. Це дуже «дорослий» QA-підхід.
- [ ] **B5. Нічний прогін.** `schedule:` у GitHub Actions — тести самі запускаються щоночі.
- [ ] **B6. Прискорення.** Підібрати `--speedup` (2–5), щоб тести йшли швидше, і перевірити, що вони не стають flaky.
- [ ] **B7. Allure на GitHub Pages** — звіт відкривається за посиланням прямо з README.

---

## 7. Тест-кейси

> Формат тобі знайомий. ID використовуй у назвах тестів: `test_tc04_takeoff_to_target_altitude`.
> Числа (таймаути, допуски) — стартові; якщо тест нестабільний, обґрунтовано підкоригуй і запиши чому.

| ID | Назва | Передумови | Кроки | Очікуваний результат | Пріоритет |
|---|---|---|---|---|---|
| **TC-01** | Підключення до автопілота | SITL запущений | Підключитись по TCP 5760, чекати HEARTBEAT | HEARTBEAT отримано ≤ 30 с; `autopilot` = ArduPilot; `type` = квадрокоптер | Smoke |
| **TC-02** | Готовність до арму | TC-01 | Чекати, поки пройдуть prearm-перевірки (GPS 3D fix, EKF готовий) | Дрон готовий до арму ≤ 90 с | Smoke |
| **TC-03** | Арм і дизарм на землі | TC-02, режим GUIDED | Заармити → перевірити → дизармити | Стан `armed` = True, потім False | Smoke |
| **TC-04** | Зліт на задану висоту | Заармлений, GUIDED | Команда `MAV_CMD_NAV_TAKEOFF` на 10 м | Відносна висота 10 ± 1 м ≤ 30 с; тримає висоту ±1 м протягом 5 с | Smoke |
| **TC-05** | Посадка | Дрон у повітрі на 10 м | Режим `LAND` | Висота < 0.5 м і автоматичний дизарм ≤ 60 с | Smoke |
| **TC-06** | Політ місії по точках | На землі, GUIDED | Завантажити місію: зліт + 3 **waypoint** 📖 + RTL; режим `AUTO` | Отримано `MISSION_ITEM_REACHED` для кожної точки; дрон повернувся і сів | Regression |
| **TC-07** | Повернення додому (RTL) | Дрон у повітрі, відлетів ~50 м від старту | Режим `RTL` | Дрон повернувся в радіус 3 м від **home** 📖 і сів | Regression |
| **TC-08** | Геозона по висоті | `FENCE_ENABLE=1`, макс. висота 20 м | Злетіти, скомандувати набір висоти до 30 м | Дрон не перевищує ~21 м; спрацьовує дія геозони (RTL/LAND) | Regression |
| **TC-09** | Failsafe: втрата зв'язку з наземною станцією | `FS_GCS_ENABLE` увімкнено; тест надсилає свої heartbeat | Злетіти → припинити надсилати heartbeat | Через таймаут failsafe дрон переходить у RTL або LAND | Regression |
| **TC-10** | Failsafe: низький заряд акумулятора | Налаштований battery failsafe | Злетіти → у симуляції знизити напругу батареї нижче порогу | Режим змінюється на RTL/LAND, у повідомленнях є попередження про батарею | Regression |
| **TC-11** | Негативний: зліт без арму | На землі, дизармлений | Надіслати `MAV_CMD_NAV_TAKEOFF` | `COMMAND_ACK` з результатом ≠ ACCEPTED; висота не змінилась | Regression |
| **TC-12** | Запис/читання параметра | TC-01 | Записати `RTL_ALT` = 2500, прочитати назад | Прочитане значення = записаному | Regression |

**Підказки до складних кейсів:**
- **TC-06 (місія):** протокол завантаження — «рукостискання»: ти надсилаєш `MISSION_COUNT` → дрон просить кожен пункт `MISSION_REQUEST_INT` → ти відповідаєш `MISSION_ITEM_INT` → в кінці дрон шле `MISSION_ACK`. В ArduPilot пункт №0 місії — це home, його теж треба надіслати (будь-які координати, він перезапишеться).
- **TC-09 (GCS failsafe):** pymavlink **сам не надсилає** heartbeat. Щоб дрон «знав», що наземна станція є, запусти в окремому потоці (`threading.Thread`) надсилання heartbeat раз на секунду, а в тесті — зупини цей потік.
- **TC-10 (батарея):** знайди в документації ArduPilot, які параметри відповідають за симуляцію батареї (група `SIM_BATT_*`) і за реакцію на низький заряд (група `BATT_FS_*`, `BATT_LOW_VOLT`). Частина параметрів батареї вимагає перезапуску автопілота — врахуй це у фікстурі.
- **TC-08 / TC-09 / TC-10:** назви параметрів можуть трохи відрізнятися між версіями прошивки. Завжди перевіряй у QGroundControl (Parameters, пошук) або в документації для своєї версії.

---

## 8. Стартовий код

> Це **стартовий каркас**, щоб не починати з порожньої папки. Розбери кожен рядок — на співбесіді можуть спитати про будь-який.

### 8.1 `scripts/get_sitl.sh`

```bash
#!/usr/bin/env bash
# Завантажує готовий SITL ArduCopter для Linux x86_64 і стандартні параметри.
set -euo pipefail

mkdir -p sitl
curl -fL -o sitl/arducopter \
  https://firmware.ardupilot.org/Copter/latest/SITL_x86_64_linux_gnu/arducopter
chmod +x sitl/arducopter

curl -fL -o sitl/copter.parm \
  https://raw.githubusercontent.com/ArduPilot/ardupilot/master/Tools/autotest/default_params/copter.parm

echo "SITL готовий: sitl/arducopter"
```

Запуск: `bash scripts/get_sitl.sh`

### 8.2 `requirements.txt`

```
pymavlink
pytest
pytest-timeout
allure-pytest
```

Встановлення:
```bash
python3 -m venv .venv
source .venv/bin/activate      # робити в кожному новому терміналі
pip install -r requirements.txt
```

### 8.3 `pytest.ini` і `.gitignore`

```ini
# pytest.ini
[pytest]
testpaths = tests
timeout = 300
markers =
    smoke: найважливіші тести, запускаються першими
    regression: повний регресійний набір
```

```
# .gitignore
.venv/
__pycache__/
sitl/
allure-results/
allure-report/
logs/
*.BIN
```

### 8.4 `tests/conftest.py`

Фікстура вміє запускати SITL двома способами: через **Docker** (твій Mac) або **напряму** (CI, Linux). Спосіб обирається **змінною середовища** 📖 `SITL_MODE`.

```python
import os
import subprocess
import time
import uuid
from pathlib import Path

import pytest
from pymavlink import mavutil

ROOT = Path(__file__).resolve().parent.parent

# "docker" — на Mac; "local" (за замовчуванням) — у CI та на Linux
SITL_MODE = os.getenv("SITL_MODE", "local")
SITL_BIN = Path(os.getenv("SITL_BIN", ROOT / "sitl" / "arducopter"))
SITL_PARAMS = Path(os.getenv("SITL_PARAMS", ROOT / "sitl" / "copter.parm"))
DOCKER_IMAGE = "uav-sitl"
HOME = "-35.363261,149.165230,584,353"   # стандартне тестове поле ArduPilot


@pytest.fixture
def sitl(tmp_path):
    """Запускає новий віртуальний дрон для кожного тесту і зупиняє його після."""
    if SITL_MODE == "docker":
        name = f"sitl-{uuid.uuid4().hex[:8]}"     # унікальне ім'я контейнера
        subprocess.run(
            ["docker", "run", "-d", "-t", "--rm", "--name", name,
             "--platform", "linux/amd64",
             "-p", "5760:5760", "-p", "5762:5762", DOCKER_IMAGE],
            check=True, stdout=subprocess.DEVNULL,
        )
        yield name
        subprocess.run(["docker", "stop", "-t", "2", name],
                       stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
    else:
        proc = subprocess.Popen(
            [str(SITL_BIN), "--model", "quad", "--speedup", "1",
             "--defaults", str(SITL_PARAMS), "-I0", "--home", HOME, "-w"],
            cwd=tmp_path,                  # логи кожного тесту — у своїй тимчасовій папці
            stdout=subprocess.DEVNULL,
            stderr=subprocess.STDOUT,
        )
        yield proc
        proc.terminate()
        proc.wait(timeout=10)


@pytest.fixture
def drone(sitl):
    """Підключення до дрона по MAVLink. Повертає з'єднання pymavlink."""
    deadline = time.time() + 60
    while True:
        conn = None
        try:
            conn = mavutil.mavlink_connection("tcp:127.0.0.1:5760", retries=1)
            if conn.wait_heartbeat(timeout=5) is not None:
                break                      # дрон відповів — підключились
        except OSError:
            pass                           # SITL ще стартує — пробуємо ще раз
        if conn is not None:
            conn.close()
        if time.time() > deadline:
            pytest.fail("Не вдалося підключитися до SITL за 60 с")
        time.sleep(1)

    # Просимо дрон надсилати телеметрію (висоту, координати тощо) 10 разів на секунду.
    conn.mav.request_data_stream_send(
        conn.target_system, conn.target_component,
        mavutil.mavlink.MAV_DATA_STREAM_ALL, 10, 1,
    )
    yield conn
    conn.close()
```

> **Чому підключення в циклі?** Docker відкриває порт 5760 одразу, ще до того, як SITL всередині готовий. Перша спроба може «підключитись у порожнечу» — тому пробуємо, поки не отримаємо heartbeat. У терміналі при цьому можуть з'являтись рядки `EOF on TCP socket` — це нормально.

### 8.5 `helpers/waits.py`

```python
import time


def wait_until(condition, timeout, interval=0.2, message="Умова не виконалась"):
    """Чекає, поки condition() поверне True. Аналог expect(...).toPass({timeout}) у Playwright."""
    deadline = time.time() + timeout
    last_error = None
    while time.time() < deadline:
        try:
            if condition():
                return
        except Exception as e:             # запам'ятовуємо, щоб показати у звіті
            last_error = e
        time.sleep(interval)
    raise TimeoutError(f"{message} за {timeout} с. Остання помилка: {last_error}")
```

### 8.6 `helpers/drone.py` — що треба написати самостійно

```python
class Drone:
    """Обгортка над з'єднанням pymavlink. Ідея така сама, як Page Object."""

    def __init__(self, conn): ...

    def set_mode(self, mode: str, timeout=10): ...            # "GUIDED", "LAND", "RTL", "AUTO"
    def wait_ready_to_arm(self, timeout=90): ...               # TC-02
    def arm(self, timeout=30): ...
    def disarm(self, timeout=30): ...
    def is_armed(self) -> bool: ...
    def takeoff(self, altitude_m: float): ...                   # MAV_CMD_NAV_TAKEOFF
    def relative_altitude(self) -> float: ...                   # GLOBAL_POSITION_INT.relative_alt / 1000
    def wait_altitude(self, target_m, tolerance_m=1.0, timeout=30): ...
    def position(self) -> tuple[float, float]: ...              # lat, lon у градусах
    def distance_to_home(self) -> float: ...                    # у метрах
    def set_param(self, name: str, value: float): ...
    def get_param(self, name: str) -> float: ...
    def upload_mission(self, waypoints: list): ...              # TC-06
    def start_gcs_heartbeat(self): ...                          # TC-09
    def stop_gcs_heartbeat(self): ...
```

**Корисні готові функції pymavlink** (подивись, як вони працюють, і використовуй всередині методів):
- `conn.set_mode("GUIDED")` — зміна режиму за назвою;
- `conn.flightmode` — поточний режим (оновлюється, коли приходить heartbeat);
- `conn.arducopter_arm()` / `conn.arducopter_disarm()` — арм/дизарм;
- `conn.motors_armed()` — чи заармлений;
- `conn.recv_match(type="GLOBAL_POSITION_INT", blocking=True, timeout=5)` — отримати конкретне повідомлення;
- `conn.mav.command_long_send(...)` — надіслати команду (наприклад, takeoff);
- `conn.param_set_send(...)` / `conn.param_fetch_one(...)` — параметри.

> ⚠️ **Важливо про арм:** після арму в режимі GUIDED дрон чекає команду зльоту лише кілька секунд — якщо її немає, він сам дизармиться. Тому `arm()` і `takeoff()` викликай одне за одним.

### 8.7 `tests/test_connection.py` — перший тест

```python
from pymavlink import mavutil


def test_tc01_heartbeat_received(drone):
    hb = drone.wait_heartbeat(timeout=10)
    assert hb is not None
    assert hb.autopilot == mavutil.mavlink.MAV_AUTOPILOT_ARDUPILOTMEGA
    assert hb.type == mavutil.mavlink.MAV_TYPE_QUADROTOR
```

Запуск: `pytest -v`

### 8.8 `.github/workflows/sitl-tests.yml`

```yaml
name: UAV SITL tests

on:
  push:
  pull_request:
  workflow_dispatch:        # кнопка «Run workflow» для ручного запуску

jobs:
  sitl-tests:
    runs-on: ubuntu-latest
    timeout-minutes: 45
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Download SITL
        run: bash scripts/get_sitl.sh

      - name: Run tests
        run: pytest -v --alluredir=allure-results

      - name: Upload Allure results
        if: always()          # зберігати звіт навіть якщо тести впали
        uses: actions/upload-artifact@v4
        with:
          name: allure-results
          path: allure-results
```

### 8.9 `Dockerfile` (для Mac)

```dockerfile
# Linux-контейнер з готовим SITL ArduCopter.
# --platform=linux/amd64: програма зроблена під Intel/AMD; на Apple Silicon її запускає Rosetta.
FROM --platform=linux/amd64 ubuntu:24.04

RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates curl \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /sitl
RUN curl -fL -o arducopter https://firmware.ardupilot.org/Copter/latest/SITL_x86_64_linux_gnu/arducopter \
 && chmod +x arducopter \
 && curl -fL -o copter.parm https://raw.githubusercontent.com/ArduPilot/ardupilot/master/Tools/autotest/default_params/copter.parm

# 5760 — для тестів, 5762 — для QGroundControl
EXPOSE 5760 5762

ENTRYPOINT ["./arducopter", "--model", "quad", "--speedup", "1", "--defaults", "copter.parm", \
            "-I0", "--home", "-35.363261,149.165230,584,353", "-w"]
```

Зібрати: `docker build -t uav-sitl .` (повторювати, лише якщо змінила `Dockerfile`; для свіжої версії прошивки — `docker build --no-cache -t uav-sitl .`).

---

## 9. Правила якості

Це те, що відрізняє «написала тести» від «побудувала тестовий фреймворк»:

1. **Жодного `time.sleep()` у тестах** для очікування стану дрона — тільки `wait_until()` з таймаутом і зрозумілим повідомленням про помилку.
2. **Кожен тест незалежний:** свій свіжий SITL (параметр `-w` очищає збережені налаштування). Тест не має залежати від того, що робив попередній.
3. **Зрозумілі повідомлення в assert:**
   `assert alt > 9, f"Дрон не набрав висоту: {alt:.1f} м замість 10 м"`
4. **Магічні числа — в константи** (`TAKEOFF_ALT_M = 10`, `ALT_TOLERANCE_M = 1.0`).
5. **Логіка — в `helpers/`, у тестах — тільки сценарій і перевірки** (як у Page Object).
6. **Commit-повідомлення** англійською, коротко й по суті: `Add TC-04 takeoff test`, `Fix flaky RTL distance check`.
7. **Нестабільний тест — це баг**, а не «перезапусти». Розберись у причині і запиши висновок у `docs/notes.md`.
8. **Кожен тест має ID** з таблиці і `@allure.title` людською мовою.

---

## 10. Типові проблеми і як їх вирішити

| Симптом | Імовірна причина | Що робити |
|---|---|---|
| `ConnectionRefusedError` при підключенні | SITL ще не стартував або вже впав | Перевір, що файл `sitl/arducopter` існує і має права на виконання (`chmod +x`); запусти SITL вручну і подивись помилки |
| `Address already in use` / порт 5760 зайнятий | Залишився процес SITL з попереднього запуску | Linux: `pkill arducopter`; Mac: див. рядок про старі контейнери нижче |
| Дрон не армиться | Не пройшли prearm-перевірки (GPS, EKF ще «прогріваються») | Чекати 30–60 с; дивитись повідомлення `STATUSTEXT` — там дрон пише причину словами |
| Висота весь час `None` / повідомлення не приходять | Не запитано потік телеметрії | Перевір `request_data_stream_send` у фікстурі |
| Після арму дрон сам дизармиться | Не надіслано команду зльоту вчасно | Takeoff одразу після arm |
| Тести проходять локально, падають у CI | CI-сервер повільніший | Збільш таймаути, не став `--speedup` більше 1 у CI без перевірки |
| `Exec format error` при запуску SITL | Linux-програму запущено напряму на Маку | На Маку SITL тільки через Docker: `export SITL_MODE=docker` (розділ 4.1) |
| `Cannot connect to the Docker daemon` | Docker Desktop не запущений | Запусти Docker Desktop, дочекайся «Engine running» |
| `Unable to find image 'uav-sitl'` | Образ ще не зібраний | `docker build -t uav-sitl .` у корені репозиторію |
| Контейнер одразу зупиняється | Помилка всередині | Запусти вручну без `-d`: `docker run --rm -it --platform linux/amd64 -p 5760:5760 uav-sitl` і подивись вивід |
| Зависли старі контейнери, порт зайнятий | Тест упав, не прибравши за собою | `docker ps` — список; `docker rm -f $(docker ps -q --filter name=sitl-)` — прибрати всі |
| `command not found: brew` | Не виконані «Next steps» після встановлення Homebrew | Виконай команди, які установник показав у кінці, і відкрий новий термінал |
| Зміна параметра «не працює» | Деякі параметри діють лише після перезапуску | Перевір в описі параметра «Reboot required» |

**Головний інструмент налагодження:** повідомлення `STATUSTEXT` — дрон текстом пише, що з ним не так. Корисно в кожному тесті при падінні прикладати їх до Allure-звіту.

---

## 11. Що написати в README

```markdown
# UAV SITL Test Suite
![tests](badge-url)

Automated test suite for ArduPilot Copter running in Software-in-the-Loop (SITL)
simulation: flight modes, missions, failsafes and geofence — in Python, pytest
and pymavlink, running in GitHub Actions on every push.

![flight](docs/flight.gif)

## What is tested
| ID    | Scenario               | Type  |
|-------|------------------------|-------|
| TC-01 | Connection & heartbeat | Smoke |
| ...   | ...                    | ...   |

## Architecture
(схема з розділу 3)

## How to run locally
macOS (Docker):
1. docker build -t uav-sitl .
2. export SITL_MODE=docker
3. python3 -m venv .venv && source .venv/bin/activate
4. pip install -r requirements.txt
5. pytest -v

Linux:
1. bash scripts/get_sitl.sh
2. python3 -m venv .venv && source .venv/bin/activate
3. pip install -r requirements.txt
4. pytest -v

## Test design decisions
- Fresh SITL instance per test for full isolation
- No fixed sleeps — condition-based waits with timeouts
- Drone Object pattern (similar to Page Object)

## Roadmap
- [ ] Docker Compose
- [ ] Wind conditions matrix
- [ ] Flight log analysis
```

README пишемо **англійською** — його читатимуть рекрутери й технічні ліди.

---

## 12. Як розповідати про проєкт на співбесіді

Підготуй коротку історію (2 хвилини) за схемою:

1. **Навіщо:** «Хотіла перенести свій досвід побудови тестових процесів у defense tech, тому зробила тестовий фреймворк для автопілота ArduPilot у симуляції».
2. **Що покрито:** базові польотні сценарії, місії і, головне, **аварійні сценарії** (втрата зв'язку, батарея, геозона) — «бо в дронах найдорожчі баги саме там».
3. **Інженерні рішення:** ізоляція тестів (свіжий SITL на кожен тест), очікування станів замість sleep, «Drone Object», CI на кожен push, Allure-звіти.
4. **Чесно про обмеження:** «SITL не замінює HIL і польові тести — це перший рівень піраміди тестування для дронів; наступні кроки — HIL-стенд і тести на реальному залізі».
5. **Зв'язок з досвідом:** «У Unanet я будувала E2E-фреймворк з нуля і запускала його в CI — тут той самий підхід, інший об'єкт тестування».

Пункт 4 особливо цінують: він показує, що ти розумієш **піраміду тестування** 📖 для заліза.

---

## 13. Словник термінів

### Дрони та автопілоти

| Термін | Що це простими словами |
|---|---|
| **UAV / БпЛА** | Unmanned Aerial Vehicle — безпілотний літальний апарат, дрон. |
| **Автопілот** | Програма на борту дрона, яка тримає його в повітрі, виконує команди і місії. |
| **Польотний контролер** (flight controller) | Маленька плата на дроні, на якій працює автопілот. |
| **Прошивка** (firmware) | Програма, «зашита» в польотний контролер. |
| **ArduPilot** | Найпоширеніший відкритий автопілот. Має версії для коптерів (ArduCopter), літаків (ArduPlane), наземних роботів (Rover), підводних апаратів (ArduSub). |
| **PX4** | Інший популярний відкритий автопілот — головний «конкурент» ArduPilot. Принципи схожі. |
| **Коптер / квадрокоптер** | Дрон з гвинтами, що летить вертикально. Квадро — 4 гвинти. |
| **SITL** | Software-In-The-Loop — «програма в контурі». Справжня прошивка автопілота працює на комп'ютері, а датчики й фізика польоту симулюються. Перший і найдешевший рівень тестування дронів. |
| **HIL** | Hardware-In-The-Loop — «залізо в контурі». Справжній польотний контролер підключений до стенду, який імітує датчики. Наступний рівень після SITL. |
| **HIL-стенд** (test bench) | Робоче місце з платою, блоками живлення, вимірювальними приладами, на якому тестують залізо. |
| **Польові тести** (field / flight tests) | Тестування на справжньому дроні в реальному польоті. Найдорожчий рівень. |
| **Піраміда тестування для заліза** | SITL (багато, дешево, швидко) → HIL (менше) → польові тести (мало, дорого). Як unit → integration → E2E у вебі. |
| **GCS** | Ground Control Station — наземна станція керування: програма (або пульт), з якої оператор керує дроном і бачить телеметрію. |
| **QGroundControl (QGC)** | Безкоштовна GCS-програма з картою. Також є **Mission Planner** (тільки Windows, популярний з ArduPilot). |
| **MAVProxy** | Консольна GCS для ArduPilot — керування командами в терміналі. Необов'язкова для цього проєкту. |
| **Телеметрія** | Дані, які дрон постійно надсилає: висота, координати, швидкість, заряд, режим. |
| **Arm / заармити** | «Взвести» дрон — дозволити моторам крутитися. Без арму дрон не злетить. **Disarm** — навпаки. |
| **Prearm checks** | Перевірки перед армом: чи є GPS, чи готові датчики, чи все налаштовано. Поки не пройдені — арм заборонено. |
| **Режим польоту** (flight mode) | Як саме дрон поводиться. Основні в ArduCopter: |
| — `STABILIZE` | Ручне керування, автопілот лише вирівнює. |
| — `LOITER` | Зависає на місці, тримає позицію по GPS. |
| — `GUIDED` | Летить туди, куди скаже програма/GCS. Головний режим для автотестів. |
| — `AUTO` | Виконує завантажену місію. |
| — `RTL` | Return To Launch — повертається на точку зльоту і сідає. |
| — `LAND` | Сідає там, де є. |
| **Home** | Точка, де дрон заармився/злетів. Сюди він повертається в RTL. |
| **Waypoint** | Точка маршруту (широта, довгота, висота). |
| **Місія** (mission) | Список waypoint-ів і команд, які дрон виконує в режимі AUTO. |
| **Відносна висота** (relative altitude) | Висота над точкою home, а не над рівнем моря. |
| **Failsafe** | Аварійна автоматична реакція: при втраті зв'язку, низькому заряді тощо дрон сам переходить у RTL або LAND. Найважливіша зона для тестування. |
| **Геозона** (geofence) | Віртуальна межа (висота, радіус, полігон), за яку дрону виходити не можна. При порушенні — автоматична дія (RTL/LAND). |
| **Параметри** (parameters) | Сотні налаштувань автопілота (назва + число), наприклад `RTL_ALT` — висота повернення в сантиметрах. |
| **EKF** | Extended Kalman Filter — «мозок орієнтації»: алгоритм, що поєднує дані GPS, гіроскопа, акселерометра, барометра й обчислює, де дрон і як він нахилений. Поки EKF не готовий — арм заборонено. |
| **GPS fix / 3D fix** | GPS «зачепився» за супутники і знає координати й висоту. |
| **Dataflash-лог (.BIN)** | Детальний «чорний ящик» польоту, який пише автопілот. |
| **Speedup** | Прискорення симуляції: `--speedup 2` — час у симуляторі йде вдвічі швидше. |

### Протокол зв'язку

| Термін | Що це простими словами |
|---|---|
| **MAVLink** | Протокол «спілкування» з дронами — як HTTP для вебу, але для дронів. Складається з повідомлень (messages) і команд (commands). |
| **pymavlink** | Python-бібліотека для роботи з MAVLink. Для тебе — це «Playwright для дронів». |
| **HEARTBEAT** | Повідомлення «я живий», яке і дрон, і GCS надсилають раз на секунду. Якщо дрон перестає чути heartbeat від GCS — може спрацювати failsafe. |
| **GLOBAL_POSITION_INT** | Повідомлення з координатами і висотою дрона. `relative_alt` — у міліметрах. |
| **SYS_STATUS** | Стан систем дрона, зокрема батареї. |
| **STATUSTEXT** | Текстові повідомлення від дрона людською мовою: попередження, причини відмови в армі тощо. |
| **COMMAND_LONG / MAV_CMD_…** | Команда дрону, наприклад `MAV_CMD_NAV_TAKEOFF` — злетіти. |
| **COMMAND_ACK** | Відповідь дрона на команду: прийнято (`ACCEPTED`), відхилено (`DENIED`), помилка (`FAILED`)… Як HTTP-статус. |
| **MISSION_ITEM_INT / MISSION_COUNT / MISSION_ACK** | Повідомлення для завантаження місії в дрон. |
| **MISSION_ITEM_REACHED** | Дрон повідомляє: «досяг точки №N». |
| **TCP-порт 5760 / 5762** | «Двері», через які підключаються до SITL. 5760 — для тестів, 5762 — для QGroundControl одночасно з тестами. |

### Тестування, Python, інструменти

| Термін | Що це простими словами |
|---|---|
| **pytest** | Найпопулярніший фреймворк для тестів на Python. |
| **Фікстура** (fixture) | Підготовка й прибирання для тесту (запустити SITL, підключитись, потім закрити). Аналог `beforeEach/afterEach` + `test.extend` у Playwright. |
| **`yield` у фікстурі** | Все до `yield` — підготовка, після — прибирання. |
| **Marker** (`@pytest.mark.smoke`) | Мітка тесту, щоб запускати групи: `pytest -m smoke`. Аналог тегів `@smoke`. |
| **parametrize** | Запуск одного тесту з різними даними: `@pytest.mark.parametrize("alt", [5, 10, 20])`. |
| **assert** | Перевірка: якщо умова хибна — тест падає. |
| **Негативний тест** | Перевіряє, що система правильно **відмовляє** в некоректній дії. |
| **Flaky test** | «Плаваючий» тест: то проходить, то падає без змін у коді. Ворог №1. |
| **Таймаут** | Максимальний час очікування. |
| **Definition of Done (DoD)** | Чіткий критерій, коли задача вважається виконаною. |
| **venv** (virtual environment) | Окрема «коробка» з Python-бібліотеками для проєкту, щоб не змішувати з іншими. Як `node_modules`. |
| **pip / requirements.txt** | Встановлювач бібліотек і список бібліотек проєкту. Як `npm` і `package.json`. |
| **Allure** | Гарні HTML-звіти тестів з кроками, вкладеннями, графіками. |
| **CI/CD** | Автоматичний запуск тестів (і збірки) при кожній зміні коду. |
| **GitHub Actions** | CI/CD прямо в GitHub. Налаштовується YAML-файлом у `.github/workflows/`. |
| **Workflow / job / step** | Сценарій CI / його завдання / окремі кроки. |
| **Runner** | Віртуальна машина GitHub, на якій виконується CI (`ubuntu-latest`). |
| **Artifact** | Файл, який CI зберігає після запуску (наприклад, результати Allure) — можна завантажити. |
| **Badge** | Значок у README «tests passing / failing» — оновлюється автоматично. |
| **Commit / push / branch / PR** | Збереження змін у git / відправка на GitHub / гілка / запит на злиття змін. |
| **WSL2** | Windows Subsystem for Linux — справжній Linux усередині Windows. |
| **Homebrew (brew)** | «Магазин програм» для терміналу Mac: `brew install python`. |
| **zsh** | Програма-оболонка терміналу на Маку. Команди майже такі самі, як у Linux (bash). Файл налаштувань — `~/.zshrc`. |
| **Rosetta** | Технологія Apple, яка дозволяє запускати програми для Intel-процесорів на процесорах Apple (M1/M2/M3/M4). |
| **Docker Desktop** | Програма для Mac/Windows, яка запускає Docker-контейнери. |
| **Образ** (image) | «Шаблон» контейнера: ОС + програми. Збирається з `Dockerfile` командою `docker build`. |
| **Контейнер** (container) | Запущений екземпляр образу. Ми запускаємо новий контейнер з SITL на кожен тест і зупиняємо після. |
| **Порт** і **`-p 5760:5760`** | «Двері» для мережевого підключення. `-p` прокидає порт контейнера на твій Mac, щоб тести могли до нього підключитись. |
| **Змінна середовища** (environment variable) | Налаштування, яке передається програмам з терміналу: `export SITL_MODE=docker`. Тести читають її через `os.getenv`. |
| **Docker / docker compose** | «Контейнери» — упакована програма з усім потрібним, запускається однаково на будь-якому комп'ютері. Compose — запуск кількох контейнерів однією командою. |
| **Bash** | Мова командного рядка Linux; `.sh` — скрипти на ній. |
| **Процес** | Запущена програма. SITL у тестах — окремий процес, який ми запускаємо і зупиняємо з Python. |
| **Потік** (thread) | Паралельне виконання коду в одній програмі — потрібен для фонового heartbeat у TC-09. |

---

## 14. Корисні посилання

**ArduPilot:**
- Документація SITL: ardupilot.org/dev/docs/sitl-simulator-software-in-the-loop.html
- Режими польоту Copter: ardupilot.org/copter/docs/flight-modes.html
- Повний список параметрів Copter: ardupilot.org/copter/docs/parameters.html
- Failsafe-и: ardupilot.org/copter/docs/failsafe-landing-page.html
- Геозона: ardupilot.org/copter/docs/common-geofencing-landing-page.html
- Як влаштовані офіційні автотести ArduPilot (джерело ідей!): ardupilot.org/dev/docs/the-ardupilot-autotest-framework.html
- Форум (там відповідають розробники): discuss.ardupilot.org

**MAVLink і pymavlink:**
- Список усіх повідомлень і команд: mavlink.io/en/messages/common.html
- Протокол місій: mavlink.io/en/services/mission.html
- Посібник з pymavlink (ArduSub, але все працює і для Copter): ardusub.com/developers/pymavlink.html

**Інструменти:**
- pytest: docs.pytest.org
- Allure + pytest: allurereport.org/docs/pytest/
- GitHub Actions: docs.github.com/actions
- Перегляд логів польоту онлайн: plot.ardupilot.org

**Порада:** коли застрягнеш — встав у ШІ-асистент (Claude, ChatGPT, Copilot у VS Code) текст помилки разом із кодом і цей файл. Але **кожен рядок, який він запропонує, розбери і зрозумій** — на співбесіді спитають саме тебе 🙂

---

*Успіхів! Починай з Етапу 0 і рухайся по чеклісту. Маленький щоденний прогрес краще за рідкі великі ривки.*
