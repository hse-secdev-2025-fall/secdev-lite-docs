# Шпаргалка по командам для S06–S12 (lite-track)

Короткий набор команд, которые пригодятся на семинарах **S06–S12**.

- Примеры даны для **Linux/macOS / WSL**.
- На Windows вместо `source .venv/bin/activate` будет команда из PowerShell:
  `.\.venv\Scripts\Activate.ps1`.

В командах `<...>` нужно заменить на свои значения (URL репозитория, теги и т.п.).

---

## 1. Базовые команды (общие)

### Клонирование репозитория и переход в папку

```bash
git clone <URL_РЕПОЗИТОРИЯ> <ИМЯ_ПАПКИ>
cd <ИМЯ_ПАПКИ>
```

### Виртуальное окружение и установка зависимостей

```bash
python -m venv .venv
source .venv/bin/activate    # Windows PowerShell: .\.venv\Scripts\Activate.ps1

pip install -r requirements.txt
```

### Запуск тестов

```bash
pytest -q
```

---

## 2. S06 — локальный запуск и тесты (seed S06–S08)

### Старт работы с seed-репозиторием

```bash
git clone <URL_secdev-seed-s06-s08> secdev-seed-s06-s08
cd secdev-seed-s06-s08

python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### Запуск тестов и приложения

```bash
# все тесты
pytest -q

# (если есть) запуск приложения
python app.py
# или:
uvicorn app.main:app --reload
```

---

## 3. S07 — Docker / Docker Compose

### Сборка Docker-образа

```bash
# в корне репозитория, где лежит Dockerfile
docker build -t foodgo-app:local .
```

### Запуск контейнера

```bash
docker run --rm -p 8080:8080 foodgo-app:local
```

После этого приложение будет доступно по адресу:

```text
http://localhost:8080
```

### (Опционально) docker-compose

```bash
docker compose up
# остановить
docker compose down
```

---

## 4. S08 — CI (GitHub Actions)

### Отправка изменений в GitHub (чтобы сработал CI)

```bash
git status
git add .
git commit -m "S08: setup CI pipeline"
git push origin main
```

### Где смотреть CI

1. Зайти в свой репозиторий на GitHub.
2. Открыть вкладку **Actions**.
3. Выбрать нужный workflow и run, посмотреть лог.

---

## 5. S09 — SBOM и SCA (Syft + Grype)

Рабочая директория — корень проекта.

### Генерация SBOM (Syft)

```bash
syft dir:. -o cyclonedx-json > EVIDENCE/foodgo-sbom-<DATE>.json
```

### Анализ уязвимостей по SBOM (Grype)

```bash
grype sbom:EVIDENCE/foodgo-sbom-<DATE>.json \
  --fail-on high \
  -o json > EVIDENCE/foodgo-sca-<DATE>.json
```

---

## 6. S10 — SAST + поиск секретов

### Semgrep (SAST)

```bash
semgrep --config p/ci \
        --severity=high \
        --error \
        --json \
        --output EVIDENCE/foodgo-sast-<DATE>.json
```

### Gitleaks (поиск секретов)

```bash
# текущее дерево
gitleaks detect \
  --no-git \
  --report-format json \
  --report-path EVIDENCE/foodgo-secrets-<DATE>.json

# вся история git
gitleaks detect \
  --log-opts="--all" \
  --report-format json \
  --report-path EVIDENCE/foodgo-secrets-history-<DATE>.json
```

---

## 7. S11 — DAST (пример с OWASP ZAP в Docker)

Перед запуском DAST нужно поднять приложение локально (через `python` или Docker).

### Простой baseline-скан ZAP (через Docker)

```bash
docker run --rm -t owasp/zap2docker-stable \
  zap-baseline.py \
  -t http://host.docker.internal:8080 \
  -r zap-report-<DATE>.html
```

- Для Linux иногда можно использовать `http://localhost:8080` и флаг `--network=host` (если позволяет окружение).
- Полученный отчёт сохранить в `EVIDENCE/zap-report-<DATE>.html`.

---

## 8. S12 — Container / IaC / Policy (Trivy и др.)

### Trivy для Docker-образа

```bash
trivy image \
  --severity HIGH,CRITICAL \
  foodgo-app:local > EVIDENCE/foodgo-trivy-image-<DATE>.txt
```

### Trivy для конфигураций (Dockerfile, манифесты и т.п.)

```bash
trivy config . \
  --severity HIGH,CRITICAL \
  > EVIDENCE/foodgo-trivy-config-<DATE>.txt
```

### (Опционально) Checkov для IaC

```bash
checkov -d . > EVIDENCE/checkov-<DATE>.txt
```

### (Опционально) Hadolint для Dockerfile

```bash
hadolint Dockerfile > EVIDENCE/hadolint-<DATE>.txt
```

---

## 9. Гигиена секретов (.env и GitHub Secrets)

### Локальный `.env` и `.env.example`

```bash
# .env.example — шаблон без реальных значений
cp .env.example .env
# дальше редактируем .env руками (НЕ коммитим)
```

Добавить `.env` в `.gitignore`:

```text
# .gitignore
.env
```

### Секреты в GitHub (CI)

В GitHub:

1. Settings → Secrets and variables → Actions.
2. Добавить `FOODGO_DB_URL`, `FOODGO_API_KEY` и т.п.

В workflow (`.github/workflows/ci.yml`):

```yaml
- name: Run tests with env
  env:
    FOODGO_DB_URL: ${{ secrets.FOODGO_DB_URL }}
    FOODGO_API_KEY: ${{ secrets.FOODGO_API_KEY }}
  run: |
    pytest -q
```
