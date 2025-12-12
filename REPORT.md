# REPORT.MD

## Лабораторная работа №2: CI и инструменты DevSecOps

**Цель работы:** Настройка пайплайна непрерывной интеграции (CI) с интеграцией инструментов DevSecOps (SAST, Secret Scanning, SCA) и генерацией артефактов (SARIF, SBOM) для обеспечения безопасности и качества кода.

---

## 1. Конфигурация CI/CD (GitHub Actions)

В соответствии с заданием, была создана конфигурация CI (Continuous Integration) в репозитории с использованием GitHub Actions (`.github/workflows/ci.yml`).

Пайплайн включает следующие ключевые этапы (`jobs`):

| Задание (Job) | Инструмент | Тип проверки | Назначение |
| :--- | :--- | :--- | :--- |
| `precommit` | `pre-commit` | Code Style/Linting | Форматирование кода (Black), сортировка импортов (isort), Flake8. |
| `build-test` | Docker / `pytest` | Build/Unit Test | Сборка Docker-образа (`app:ci`) и запуск тестов внутри контейнера. |
| `semgrep` | Semgrep | SAST (Static Analysis) | Сканирование кода на уязвимости (OWASP Top 10) с генерацией SARIF. |
| `gitleaks` | Gitleaks | Secret Scanning | Поиск секретов, ключей и учетных данных в истории коммитов. |
| `trivy-fs` | Trivy | SCA (Dependencies) | Сканирование зависимостей файловой системы на уязвимости. |
| `trivy-image` | Trivy | Vulnerability Scan | Сканирование Docker-образа (`app:ci`) на уязвимости ОС и библиотек. |
| `sbom` | Trivy | Artifact Generation | Генерация Спецификации материалов программного обеспечения (SBOM) в формате CycloneDX. |

---

## 2. Анализ прогона CI и устранение ошибок

При первичном запуске пайплайна были обнаружены две критические ошибки в заданиях безопасности.

### 2.1. Ошибка 1: `gitleaks` (Secret Scanning)

| Параметр | Значение |
| :--- | :--- |
| Задание | `gitleaks` (Run gitleaks) |
| Ошибка | `Error: Process completed with exit code 127.` |
| Лог | `./gitleaks: No such file or directory` |

**Анализ и решение:**
Ошибка `No such file or directory` возникла, потому что исполняемый файл Gitleaks, установленный через `curl` в одном шаге, не был доступен по относительному пути `./gitleaks` в следующем шаге.

**Исправление:** Рекомендовано заменить ручную установку (`curl`) на использование официального GitHub Action `gitleaks/gitleaks-action@v2` для гарантии доступности исполняемого файла в `$PATH` среды.

### 2.2. Ошибка 2: `sbom` (Генерация SBOM)

| Параметр | Значение |
| :--- | :--- |
| Задание | `sbom` (Generate SBOM (FS)) |
| Ошибка | `Error: Process completed with exit code 1.` |
| Лог | `SBOM decode error: unknown scanning is not yet supported` |
| Примечание | `Detected SBOM format format="unknown"` |

**Анализ и решение:**
Инструмент Trivy не смог определить тип сканируемого объекта (`format="unknown"`) и столкнулся с файлом, который не может быть декодирован для построения SBOM.

**Исправление:** Необходимо явно исключить файлы и директории, не являющиеся исходным кодом или зависимостями, с помощью флагов `--skip-dirs` и/или `--skip-files`.

**Пример команды для исправления:**

```bash
trivy sbom --format cyclonedx --output sbom.cdx.json . --skip-dirs .git,dist,build
````

### 2.3. Дополнительные замечания (Trivy и Code Scanning)

  * **Trivy Fail-on Policy:** В задании требуется настроить политику `fail-on` для Trivy на `HIGH,CRITICAL`. Это настроено в заданиях `trivy-fs` и `trivy-image` с помощью параметров `severity: 'HIGH,CRITICAL'` и `exit-code: '1'`.
  * **Code Scanning Warning:** При генерации SBOM Trivy выдал предупреждение, что `--format cyclonedx` отключает сканирование безопасности, если не указать явно `--scanners vuln`. Для соответствия требованиям Code Scanning (SARIF) и получения отчета об уязвимостях в SBOM, следует использовать флаг `--scanners vuln`.

-----

## 3\. Артефакты и доказательства

| Требование | Место для вставки |
| :--- | :--- |
| Ссылка на репозиторий | [url](https://github.com/misaains/lab3/new/main) |
| Ссылки на успешные/провальные прогонки CI | [url](https://github.com/misaains/lab3/actions/runs/20160048957), [url](https://github.com/misaains/lab3/actions/runs/20159992838) |
| Скачанные отчеты (SARIF/JSON) | [url](https://github.com/misaains/lab3/actions/runs/20160048957/artifacts/4847694846) |
| Скриншоты настроек Branch Protection/Required checks | <img width="582" height="270" alt="image" src="https://github.com/user-attachments/assets/cf30f2e0-f434-482d-b815-ae2e92f8e32e" />|

-----
